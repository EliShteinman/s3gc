[← חלק 2](02-s3-and-collect.md) | [חזרה לתוכן](README.md)

# חלק 3: שלב ההתאמה והמחיקה (use)

`main()` קורא ל-`do_use()` אם לא הוגדר `--collectonly`
([שורה 1489](../../s3gc.py#L1489)). זה השלב שמחליט מה יתום, ובמצב delete גם
מוחק. כל הפונקציה נמצאת ב-[שורות 1227-1477](../../s3gc.py#L1227-L1477).

הסדר בתוך `do_use()`:

```text
do_use()
  ├─ 1. בדיקת חלון הגיל (useage ≥ 24)
  ├─ 2. preflight ראשון                     (לא ב-dry-run)
  ├─ 3. בחירת הצד הימני של ה-join (מקומי / clusterAllReplicas)
  ├─ 4. בדיקה שטבלת העזר קיימת ולא ריקה
  ├─ 5. check_samples_match_partitioning()
  ├─ 6. הגדרת make_antijoin()
  ├─ 7. אישור אינטראקטיבי                   (רק במסוף, לא ב-Kubernetes)
  ├─ 8. לולאה על ה-samples:
  │      preflight → anti-join בזרימה → מחיקה באצוות → tombstones
  └─ 9. סיכום, ספירה מצטברת, TRUNCATE (אלא אם --keepdata)
```

## 1. חלון הגיל ([שורות 1228-1253](../../s3gc.py#L1228-L1253))

`--useage` (ברירת מחדל 24) קובע: **רק אובייקטים בני יותר מ-X שעות ייחשבו
יתומים.** המינימום הוא `MINIMUM_USEAGE_HOURS = 24`
([שורה 706](../../s3gc.py#L706)).

**למה זה קריטי:** ClickHouse כותב קודם את הקובץ ל-S3, ורק אחר כך רושם אותו
ב-`system.remote_data_paths`. בחלון הזמן שביניהם האובייקט **נראה יתום**, אבל
הוא לא. חלון הגיל הוא ההגנה היחידה מפני מחיקה שלו.

- **מתחת ל-24 נדחה גם ב-dry-run.** אחרת ה-dry-run היה מראה רשימה שונה ממה
  שה-delete באמת ימחק.
- **החריג היחיד:** `--dev-allow-short-useage`, שמיועד לשלב ה-dev-automation
  בלבד. הוא כותב אזהרה בולטת ושורה ביומן ההרצות.

## 2. ה-preflight הראשון ([שורה 1255](../../s3gc.py#L1255))

`preflight_cluster()` ([שורות 867-892](../../s3gc.py#L867-L892)) רץ רק כשזו
מחיקה אמיתית. מה הוא בודק:

1. **אם `--expected-replicas` לא הוגדר, הוא חוזר בלי לבדוק כלום.** בשלב
   `delete` של Kubernetes ה-entrypoint מחייב אותו, אבל בהרצה ידנית לא.
2. `--expected-replicas` בלי `--cluster` זו שגיאה.
3. `SELECT getMacro('cluster')` חייב להחזיר בדיוק את `--cluster`. זה מוודא
   שהתחברת ל-cluster הנכון.
4. `SELECT count() FROM clusterAllReplicas('<cluster>', system.one)` חייב
   להיות שווה ל-`--expected-replicas`. כל replica מחזיר שורה אחת, אז זו ספירה
   של ה-replicas שעונים **כרגע**.

**למה זה קריטי:** אם replica אחד לא עונה, ההפניות שלו לא יופיעו בצד הימני של
ה-join. האובייקטים שרק הוא מפנה אליהם ייראו יתומים, וימחקו. לכן עדיף להיכשל.

## 3. הצד הימני של ה-join ([שורות 1258-1260](../../s3gc.py#L1258-L1260))

```python
srdp = "system.remote_data_paths"
if args.clustername:
    srdp = f"clusterAllReplicas('{args.clustername}', {srdp})"
```

- **עם `--cluster`:** ‏`clusterAllReplicas` מאחד את `remote_data_paths` מכל
  ה-replicas. זה הנכון כשכמה replicas חולקים את אותו bucket.
- **בלי `--cluster`:** רק ה-replica המקומי. אם יש עוד replicas שכותבים לאותו
  bucket, ההפניות שלהם לא ייבדקו.

## 4. האם יש בכלל מה לעבוד עליו ([שורות 1262-1281](../../s3gc.py#L1262-L1281))

`SELECT COUNT(1) FROM <tname>`. אם התוצאה 0, **או** שהשאילתה נכשלה (למשל
הטבלה לא קיימת), נזרק `RuntimeError`.

**למה לא פשוט "אין מה למחוק, הצלחה":** טבלה ריקה בשלב הזה כמעט תמיד אומרת
שמשהו השתבש: ה-collect לא רץ, רץ מול replica אחר, או שהטבלה רוקנה. זו
בדיוק התקלה שה-route עם איזון העומסים שלך יגרום: collect ל-replica א' ו-delete
ל-replica ב'.

## 5. `check_samples_match_partitioning()` ([שורות 1194-1224](../../s3gc.py#L1194-L1224))

בודקת שה-`--samples` של השלב הזה שווה ל-`% N` שבו הטבלה נוצרה באיסוף. היא רק
**מזהירה** ולא עוצרת. ההערה בקוד מתעדת את המחיר: sample אחד לקח כ-2 דקות
כשהערכים התאימו, וכ-26 דקות כשלא, כי partition pruning לא עבד.

ראה בתצפיות למטה למה הבדיקה הזו לא באמת רצה במקרה הנפוץ.

## 6. `make_antijoin(calc_only, sample)` ([שורות 1285-1306](../../s3gc.py#L1285-L1306))

פונקציה פנימית שבונה את ה-SQL. זו השאילתה שמחליטה מה יתום:

```sql
SELECT s3o.objpath, s3o.size AS size, s3o.last_modified AS last_modified
FROM <tname> AS s3o
LEFT ANTI JOIN clusterAllReplicas('<cluster>', system.remote_data_paths) AS rdp
    ON (rdp.remote_path = s3o.objpath AND rdp.disk_name = '<s3diskname>')
WHERE CRC32(s3o.objpath) % <samples> = <sample>   -- רק ה-sample הנוכחי
  AND s3o.active = true                           -- לא נמחק עדיין
  AND s3o.objpath > '<useafter>'                  -- רק עם --useafter
  AND s3o.last_modified < now() - interval <useage> hour
ORDER BY s3o.objpath                              -- רק עם --order-by-objpath
LIMIT <usetotal>                                  -- רק עם --usetotal
SETTINGS final = 1
```

שורה אחרי שורה:

| חלק | מה הוא עושה |
| --- | --- |
| `LEFT ANTI JOIN` | מחזיר שורות מ-`s3o` (מה שיש ב-S3) **שאין** להן התאמה ב-`rdp` (מה ש-ClickHouse מכיר) |
| `ON remote_path = objpath AND disk_name` | ההתאמה לפי נתיב, רק בדיסק שצוין ב-`--s3diskname` |
| `CRC32(objpath) % samples = sample` | מגביל ל-partition אחד, אותה נוסחה כמו ב-`PARTITION BY` |
| `active = true` | מדלג על אובייקטים שכבר סומנו כנמחקים |
| `last_modified < now() - useage` | חלון הבטיחות מסעיף 1 |
| `SETTINGS final = 1` | קורא את הטבלה כאילו `FINAL`: לכל `objpath` רק השורה האחרונה. בלי זה, אובייקט שנמחק היה מופיע שוב דרך השורה הישנה עם `active=true` |

**למה זה עובד עם ה-tombstones:** אחרי מחיקה נכתבת שורה חדשה עם אותו `objpath`
ו-`active=false`. ‏FINAL בוחר אותה במקום המקורית, והתנאי `active = true`
מסנן אותה. כך הרצה חוזרת לא מנסה למחוק שוב את מה שכבר נמחק.

עם `calc_only=True` אין תנאי sample, והשאילתה עטופה ב-
`SELECT COUNT(1), SUM(size) FROM (...)`. משמש רק לאישור האינטראקטיבי.

## 7. אישור אינטראקטיבי ([שורות 1309-1333](../../s3gc.py#L1309-L1333))

רץ רק אם **כל** התנאים מתקיימים: `--interactive` דלוק (ברירת מחדל), זו לא
dry-run, ו-stdin ו-stdout הם מסוף אמיתי.
- סופר את כל היתומים (שאילתת `calc_only`). אם אין, יוצא בהצלחה.
- שואל "Proceed with removing N objects of total size S? (y/n)" עד שמקבל
  תשובה תקינה. "n" יוצא בלי למחוק.

ב-Kubernetes זה לא רץ: ה-entrypoint מעביר `--non-interactive`, ואין מסוף.
האישור שם הוא `DELETE_ORPHANS` ב-entrypoint.

## 8. הלולאה על ה-samples ([שורות 1340-1440](../../s3gc.py#L1340-L1440))

קודם בודק ש-`--deletebatchsize` הוא לפחות 1 (שורה 1338). אחר כך, עבור כל
`sample` מ-0 עד `samples-1`:

### 8.1 preflight לכל sample (שורה 1346)

שוב `preflight_cluster()`. ההערה בקוד מסבירה: sample יכול לרוץ שעות, ו-replica
יכול ליפול באמצע. בדיקה אחת בהתחלה לא מספיקה.

### 8.2 שאילתה בזרימה (שורה 1359)

```python
with ch_client.query_row_block_stream(antijoin) as stream:
    for block in stream:
```

התוצאה לא נטענת כולה לזיכרון. ClickHouse שולח אותה בבלוקים, והקוד מעבד בלוק
אחרי בלוק. **כל עוד הזרימה פתוחה, ה-session של `ch_client` תפוס.** לכן
ה-tombstones נכתבים דרך `ch_writer` (ראה חלק 1).

### 8.3 dry-run: רק ספירה (שורות 1368-1371)

ב-dry-run כל בלוק רק מוסיף למונים, וממשיכים לבלוק הבא. אין S3, אין
tombstones, אין preflight.

### 8.4 מחיקה באצוות (שורות 1373-1401)

הבלוק נחתך לאצוות של `--deletebatchsize` (ברירת מחדל 1000, המקסימום
ש-`DeleteObjects` מקבל). לכל אצווה יש שני מסלולים:

**מסלול א': מחיקה באצוות** (`--use-remove-objects`, ברירת המחדל, ומה ש-Ceph
משתמש בו):
1. `remove_objects_reconnecting(batch_rows)`: בקשת `DeleteObjects` אחת לכל
   האצווה (ראה חלק 2).
2. מקבלים רשימת שגיאות, אחת לכל אובייקט שלא נמחק.
3. מוציאים מכל שגיאה את שם האובייקט שלה.
4. **אם לשגיאה כלשהי אין שם** (`None in failed_names`), אי אפשר לדעת מה
   נכשל. לכן **אף אובייקט באצווה לא מסומן כנמחק.** זה בטוח: בהרצה הבאה הם
   ינסו להימחק שוב, ומחיקה חוזרת לא מזיקה.
5. אחרת, כל מי שלא ברשימת השגיאות נחשב הצלחה.

**מסלול ב': אובייקט אחד בכל בקשה** (GCS, או `--use-remove-objects false`):
`remove_object` לכל שורה. הצלחה נכנסת ל-`successful_rows`, וכישלון נאסף
ל-`errors`.

### 8.5 tombstones ו-checkpoint (שורות 1403-1427)

עבור כל אובייקט שנמחק בהצלחה נכתבת שורה `[objpath, size, last_modified,
False]` לטבלת העזר, דרך `ch_writer`. אחר כך:
- לוג INFO: `delete checkpoint: N objects / S bytes removed so far`.
- `run_log("checkpoint")`, התאום העמיד של שורת הלוג. אם ה-pod נמחק, זה מה
  שנשאר כדי לדעת עד איפה הגענו.

**הסדר חשוב:** קודם מוחקים מ-S3, ורק אחר כך כותבים tombstone. אם התהליך מת
בין השניים, האובייקט נמחק אבל לא סומן. בהרצה הבאה ינסו למחוק אותו שוב, וזה
לא מזיק. הסדר ההפוך (קודם לסמן) היה יכול להשאיר אובייקט "מסומן כנמחק" שבעצם
עדיין קיים.

### 8.6 כשל: עצירה מיידית (שורות 1429-1438)

אם הייתה שגיאת מחיקה כלשהי באצווה, **אחרי** שההצלחות כבר סומנו:
- `run_log("error")`.
- `raise S3DeletionError(...)` ([שורה 709](../../s3gc.py#L709)). ההרצה נעצרת,
  וה-Job ב-Kubernetes נכשל.

**ה-Job לא מנסה שוב לבד.** זה כלל בטיחות (`backoffLimit: 0`). מפעיל צריך
להסתכל על השגיאה ולהחליט. כשמריצים שוב את `delete`, ה-tombstones גורמים לכך
שממשיכים מאיפה שעצרנו.

## 9. אחרי כל ה-samples ([שורות 1439-1477](../../s3gc.py#L1439-L1477))

1. **סיכום "in this attempt":** כמה נמחקו **בהרצה הזו בלבד**. ההערה בקוד
   מספרת שבעבר כתוב שם "this run", וזה הטעה: בהרצה שהמשיכה אחרי כשל, השורה
   הראתה 16.61 TiB פחות ממה שנמחק בפועל.
2. **ספירה מצטברת** (רק במחיקה): כמה tombstones יש בטבלה, מכל ההרצות.
   ```sql
   SELECT count(), sum(size) FROM <tname> FINAL WHERE active = false
   ```
   זו האמת המצטברת. אם השאילתה נכשלת, רק כותבים לוג, ולא מכשילים הרצה
   שהצליחה.
3. **`TRUNCATE`** של טבלת העזר, אלא אם `--keepdata` או dry-run. ב-Kubernetes
   כל השלבים מעבירים `--keepdata`, כך שהטבלה נשארת. אפשר לבדוק אותה אחרי, והיא
   נדרשת להמשכה.

## כמה קריאות יש בשלב הזה

| יעד | פעולה | כמות |
| --- | --- | --- |
| ClickHouse | preflight (2 שאילתות) | 1 + samples |
| ClickHouse | anti-join | אחת לכל sample (ברירת מחדל 4) |
| ClickHouse | INSERT של tombstones | אחת לכל אצוות מחיקה שהצליחה בחלקה |
| ClickHouse | run_log | כ-2 לכל אצווה + כמה בודדות |
| S3 | `DeleteObjects` (POST) | בערך (מספר היתומים) / 1000 |
| S3 | `DeleteObject` (מסלול ב') | אחת לכל יתום |
| S3 | ניסיונות חוזרים | עד 3 לבקשה (urllib3), ועוד ניסיון אחד אחרי התחברות מחדש |

**לגבי ה-rate limit שלך:** במסלול א', 1,000,000 יתומים הם כ-1,000 בקשות
`DeleteObjects`. **אבל** צריך לברר איך Ceph RGW סופר את המגבלה: לפי בקשות
HTTP, או לפי אובייקטים שנמחקו. אם לפי אובייקטים, כל בקשה "שווה" 1000. בנוסף,
אין בקוד שום השהיה בין אצוות. הבקשות נשלחות ברצף, אחת אחרי השנייה, הכי מהר
ש-S3 עונה.

## הערות ותצפיות מחלק 3

- **בדיקת ה-samples לא עובדת עם קידומת שכוללת database.**
  `tname.strip('`').split('.')[-1]` על ‏`` `s3gc`.`s3gc_run1_s3` `` מחזיר
  `` s3gc_run1_s3` `` עם backtick בסוף. בנוסף השאילתה מחפשת ב-
  `currentDatabase()` ולא ב-database של הטבלה. התוצאה: אין שורות, והאזהרה
  אף פעם לא מופיעה. ה-runbook ממליץ בדיוק על קידומת כזו. **נבדק בסימולציה של
  הלוגיקה, לא בבדיקה על הקוד עצמו.**
- **`--usetotal` הוא לכל sample, לא לכל ההרצה.** ה-`LIMIT` נמצא בתוך השאילתה
  של כל sample, ולכן עם `--usetotal 1000` ו-4 samples יכולים להימחק עד 4000.
- **הצד הימני של ה-join נקרא מחדש בכל sample.** `clusterAllReplicas(system.remote_data_paths)`
  לא מסונן לפי sample, אז כל ה-cluster נקרא 4 פעמים (פעם לכל sample). ב-hash
  join רגיל ClickHouse בונה את הצד הימני בזיכרון. זה המקום הראשון לחפש בו
  כשמשהו איטי או נופל על זיכרון בסקייל גבוה.
- **ה-preflight מדלג בשקט בלי `--expected-replicas`.** בהרצה ידנית בלי הדגל,
  אין בדיקת topology בכלל. ה-entrypoint של Kubernetes מכסה את זה, הרצה ידנית
  לא.
- **שגיאה בשאילתת ה-COUNT נבלעת.** כל exception בשורה 1266 נרשם ב-INFO
  ונחשב כ-0. זה בטוח (התוצאה היא עצירה), אבל הודעת השגיאה תגיד "לא קיימת או
  ריקה" גם כשהבעיה האמיתית היא, למשל, timeout.

[← חלק 2](02-s3-and-collect.md) | [חזרה לתוכן](README.md)
