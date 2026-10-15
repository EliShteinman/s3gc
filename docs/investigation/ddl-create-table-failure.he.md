# בדיקה: `IncompleteRead` ב-`CREATE TABLE` של שלב האיסוף

## מה ידוע

- **הפקודה:** `python s3gc.py --cfg <config> --verbose --collectonly --unordered`
  (הגרסה של החברה).
- **הקריסה:** `IncompleteRead` על ה-`CREATE TABLE IF NOT EXISTS` של טבלת העזר,
  לפני כל פנייה ל-S3. זה קרה למרות שהטבלאות כבר היו קיימות.
- **מצב ה-cluster:** באותו זמן replica אחד היה כבוי. כשהוא עלה, הבעיה נעלמה.
- **ה-route נשלל:** ה-route של OpenShift מחזיק חיבורים פתוחים הרבה אחרי 30
  שניות, אז לא הוא חתך את החיבור.
- **הקוד המקורי:** ב-`s3gc.py` של Altinity ה-CREATE הוא `ENGINE ReplacingMergeTree`
  בלי `ON CLUSTER` ([שורה 1128](../../s3gc.py#L1128)). פקודה כזו לא אמורה
  לחכות ל-replicas אחרים.

## השערות

| # | השערה | מה יאשר אותה |
| --- | --- | --- |
| H1 | ה-database של טבלת העזר הוא `ENGINE = Replicated`, אז כל DDL בו מחכה לכל ה-replicas | בדיקה 2 מחזירה `Replicated` |
| H2 | בגרסה של החברה ה-CREATE כולל `ON CLUSTER` | בדיקה 7 מוצאת `ON CLUSTER` |
| H3 | ClickHouse שלח `200 OK`, נכשל באמצע (timeout בהמתנה ל-replica) וקטע את התשובה | בדיקות 1, 4 ו-5 |

H1 ו-H2 מסבירות **למה** ה-CREATE חיכה. H3 מסבירה **למה** השגיאה נראית כמו
`IncompleteRead` ולא כמו שגיאה של ClickHouse.

## הבדיקות

כל הבדיקות כאן הן **קריאה בלבד**, חוץ מבדיקה 8. בכל מקום שכתוב `<...>` צריך
להחליף בערך שלך.

### 1. כמה זמן עבר עד הקריסה

בלוג של s3gc (עם `--verbose`) לכל שורה יש חותמת זמן. מספיק ההפרש בין השורה
`creating table ...` לבין ה-traceback.

| זמן בערך | מה כנראה נגמר |
| --- | --- |
| 180 שניות | `distributed_ddl_task_timeout`, ההמתנה ב-`ON CLUSTER` (H2) |
| 300 שניות | `database_replicated_initial_query_timeout_sec` (H1) |
| 1800 שניות | `--chtimeout` של s3gc, ה-timeout של ה-client |
| מיידי / אחר | כיוון אחר, לבדוק את בדיקה 4 |

### 2. סוג ה-database

```sql
SELECT name, engine FROM system.databases;
```

צריך את השורה של ה-database שבו טבלת העזר נוצרת: החלק שלפני הנקודה
ב-`collecttableprefix`. אם אין נקודה, זה ה-database ברירת המחדל של המשתמש.
לגלות אותו:

```sql
SELECT currentDatabase();
```

### 3. המנוע של טבלת העזר, ובאילו replicas היא קיימת

```sql
SELECT hostName(), database, name, engine
FROM clusterAllReplicas('<cluster>', system.tables)
WHERE name LIKE '<prefix-without-db>%'
ORDER BY hostName(), name;
```

- **הטבלה קיימת בכל ה-replicas:** ה-DDL שוכפל (H1 או H2).
- **הטבלה קיימת רק ב-replica אחד:** ה-DDL לא שוכפל, וצריך לחשוב מחדש.

### 4. השגיאה האמיתית לפי ClickHouse

```sql
SELECT hostName(), event_time, type, exception_code, exception, query_duration_ms
FROM clusterAllReplicas('<cluster>', system.query_log)
WHERE query ILIKE '%CREATE TABLE IF NOT EXISTS%'
  AND event_date >= today() - 7
  AND type != 'QueryStart'
ORDER BY event_time DESC
LIMIT 20;
```

אם הקריסה הייתה לפני יותר משבוע, צריך להגדיל את `today() - 7`. ייתכן שה-log
כבר נמחק (תלוי בהגדרת ה-TTL שלו).

### 5. תור ה-DDL המשותף

```sql
SELECT entry, host, status, exception_code, query_create_time, query_duration_ms
FROM system.distributed_ddl_queue
WHERE query ILIKE '%CREATE TABLE IF NOT EXISTS%'
ORDER BY query_create_time DESC
LIMIT 20;
```

- **יש שורות:** ה-DDL עבר דרך התור המשותף, כלומר `ON CLUSTER` או database
  מסוג `Replicated`.
- **`status` של replica:** ‏`Finished` אומר שסיים. ‏`Active` או `Inactive` אומרים
  שלא סיים, וזה כנראה ה-replica שהיה כבוי.

### 6. ה-timeouts שהגדרת ה-DDL משתמשת בהם

```sql
SELECT name, value, changed
FROM system.settings
WHERE name IN (
    'distributed_ddl_task_timeout',
    'distributed_ddl_output_mode',
    'database_replicated_initial_query_timeout_sec'
);
```

צריך להריץ את זה עם **אותו משתמש** ש-s3gc משתמש בו, כי ל-profile שלו יכולים
להיות ערכים אחרים.

### 7. הקוד של החברה

לחפש בקוד של החברה:

```bash
grep -n "ON CLUSTER\|CREATE TABLE\|CREATE DATABASE\|ENGINE" s3gc.py
grep -n "get_client\|send_receive_timeout\|chtimeout" s3gc.py
```

- **שאלה 1:** האם יש `ON CLUSTER` ב-CREATE של טבלת העזר? (H2)
- **שאלה 2:** האם המנוע שונה מ-`ReplacingMergeTree`, למשל `Replicated...`?
- **שאלה 3:** מה ה-timeout של ה-client ל-ClickHouse?

### 8. שחזור עם השגיאה המלאה (רק אם בדיקות 1-7 לא הכריעו)

לשחזר את הבעיה צריך replica כבוי, כך שזה לא דבר שעושים בכל זמן. אם יש חלון
כזה, אפשר להריץ את ה-CREATE ישירות על pod, בלי s3gc ובלי HTTP:

```bash
oc exec -it <clickhouse-pod> -- clickhouse-client \
  --query "CREATE TABLE IF NOT EXISTS <db>.s3gc_ddl_probe (x UInt8) ENGINE MergeTree ORDER BY x"
```

- **מה זה נותן:** הפרוטוקול ה-native מציג את השגיאה של ClickHouse במלואה.
- **מה זה יוצר:** טבלת בדיקה ריקה. אחרי הבדיקה מוחקים אותה עם
  `DROP TABLE <db>.s3gc_ddl_probe`, וגם זה DDL שיחכה ל-replicas.

## מה לשתף ומה לא

**בסדר לשתף:**
- קודי שגיאה וטקסט השגיאה, אחרי ניקוי של שמות
- זמנים ומשכים
- שמות מנועים (`Replicated`, ‏`Atomic`, ‏`ReplacingMergeTree`...)
- ערכי `status`
- ערכי ה-settings
- תשובות כן/לא על הקוד של החברה

**לא לשתף:**
- שמות hosts ו-pods
- שם ה-cluster ושמות databases אמיתיים
- סיסמאות ומפתחות
- כתובות
- קטעי קוד של החברה, אלא אם מותר

אפשר להחליף שמות ב-`replica-1`, ‏`replica-2` וכו'.
