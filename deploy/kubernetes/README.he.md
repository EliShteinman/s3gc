[English](README.md) | **עברית**

# מריץ ה-Job של Kubernetes

הריצו את `s3gc` כ-Job חד-פעמי ב-namespace של ClickHouse. מדריך זה הוא נוהל
הייצור.

<a id="prerequisite"></a>

## דרישה מקדימה

השתמשו ב-Python 3.11 מתיקיית השורש של ה-repository. צרו את הסביבה הווירטואלית
של הפרויקט לפני שאתם מרנדרים Job:

```bash
python3.11 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt -r requirements-dev.txt
```

`render.py` משתמש רק בספרייה הסטנדרטית של Python. ה-runbook עדיין משתמש
ב-`.venv/bin/python` כדי שכל פקודה תרוץ עם גרסת ה-Python הנתמכת.

<a id="runbook"></a>

## Runbook (נוהל הפעלה)

### 1. הכינו תיקיית הרצה פרטית

שמרו קבצים שנוצרים מחוץ ל-repository הזה:

```bash
export S3GC_RUN_DIR=/path/to/private-s3gc-runs/customer-cluster
mkdir -p "$S3GC_RUN_DIR"
cp deploy/kubernetes/example.env "$S3GC_RUN_DIR/s3gc.env"
```

התיקייה תכיל את התצורה ללא הסודות ו-manifest מרונדר אחד לכל שלב:

```text
$S3GC_RUN_DIR/
├── s3gc.env
├── collect.yaml
├── dry-run.yaml
├── delete.yaml
└── verify.yaml
```

אל תבצעו commit לתיקייה הזו. היא מזהה סביבת יעד גם כשאין בה פרטי הזדהות.

### 2. אשרו את היקף הניקוי

הגדירו את הערכים ב-`s3gc.env`, ואז בדקו את חמשת הסעיפים האלה לפני הרינדור:

1. הגדירו את `CHHOST` ל-Service אחד של ClickHouse ברמת replica בודד (per-replica), והשתמשו באותו host בכל השלבים. טבלת המצאי היא מקומית לצומת (node-local); Service עם איזון עומסים (load-balanced) עלול לשלוח שלב מאוחר יותר ל-replica שאין בו את הטבלה הזו.
2. הגדירו את ה-bucket המדויק, את הקידומת ואת שם דיסק האובייקטים הבסיסי (underlying). `S3PATH` יכול להיות ריק. ב-GCS נהוג להשתמש ב-`gcs`; לעולם אל תשתמשו בדיסק `*_cache`.
3. הגדירו את ה-`CLUSTERNAME` וה-`EXPECTED_REPLICAS` האמיתיים לניקוי ב-cluster. המחיקה נכשלת באופן סגור (fails closed) אם בדיקת ה-preflight הזו אינה תואמת.
4. בחרו `COLLECTTABLEPREFIX` ייחודי ומסויג בשם מסד נתונים (database-qualified), כמו `s3gc.s3gc_<run>_`. שמרו עליו אחרי מחיקה חלקית כדי שה-Job החלופי יוכל להשתמש ב-checkpoints של המחיקה. קידומת חשופה (ללא שם מסד נתונים) משתמשת במסד הנתונים הנוכחי של משתמש ה-ClickHouse, שעשוי להיות שונה מ-`default` ולגרום לכשל הרשאות (grant).
5. הגדירו את `USEAGE_HOURS` ל-24 או יותר. העלו אותו כשיש merges איטיים או mutations ארוכות; שלבי ייצור אינם יכולים להוריד אותו.

לניקוי ברמת כל ההתקנה, הגדירו את `S3PATH` לקידומת-האב שמכילה את האובייקטים
של כל ה-replicas, כמו `clickhouse/<installation>/`, ולא לקידומת בודדת מסוג
`<installation>/<replica>/` — אבל רק כאשר קידומת-האב הזו מכילה אובייקטים עבור
`S3DISKNAME` האחד שמנוקה. בדיקת היתומים מתאימה מול `system.remote_data_paths`
של ClickHouse לפי `disk_name`, ולכן אובייקט ששייך לדיסק אחר ונשמר תחת אותה
קידומת-אב לעולם לא יימצא תואם וייראה יתום. כאשר מספר דיסקים חולקים
קידומת-אב, אספו במקום זאת קידומת מוגבלת לדיסק, או תנו לכל דיסק מבנה נתיבי S3
משלו.
אספו את הקידומת שנבחרה פעם אחת לתוך טבלת מצאי אחת על ה-host המקובע.
כאשר `CLUSTERNAME` מוגדר, dry-run ו-delete משווים את המצאי מול
`system.remote_data_paths` בכל ה-replicas, כך שאובייקט שאיזשהו replica עדיין
מפנה אליו לעולם לא יהיה מועמד למחיקה. קידומת של replica בודד מפספסת את
היתומים של שאר ה-replicas. אספו קידומות per-replica רק כשזו הכוונה שלכם, ותנו
לכל אחת `COLLECTTABLEPREFIX` משלה.

השתמשו ב-digest בלתי משתנה של image רב-ארכיטקטורות (multi-architecture)
ב-`IMAGE`. ה-CI מדפיס את הערך המדויק לאחר הפרסום. אל תשתמשו בתגית image.

לאיסוף אין checkpoint להמשכה. עבור bucket גדול, אספו shards נפרדים לפי קידומת
והריצו מחדש רק shard שנכשל. רישום מחדש (re-listing) של shard הוא בטוח, כי
טבלת העזר מחליפה שורות לפי נתיב האובייקט.

### 3. ספקו פרטי הזדהות וזהות

צרו או השתמשו מחדש ב-Secret של Kubernetes ששמו מוגדר ב-`CREDENTIALS_SECRET`:

| `S3AUTH` | ערכים ב-Secret |
| --- | --- |
| `static` | `S3GC_CHUSER`, `S3GC_CHPASS`, `S3GC_S3ACCESSKEY`, `S3GC_S3SECRETKEY`, ו-`S3GC_S3SESSIONTOKEN` אופציונלי |
| `iam` | `S3GC_CHUSER` ו-`S3GC_CHPASS` |
| `aws` | `S3GC_CHUSER`, `S3GC_CHPASS`, ובנוסף מקור פרטי הזדהות שזמין ל-boto3 |

הגדירו את `SERVICE_ACCOUNT` ל-ServiceAccount קיים ומקומי ל-namespace, שה-cluster
שלכם מגדיר עבור מצב ה-workload-identity שנבחר. ה-Job שומר על
`automountServiceAccountToken: false` כי הוא אינו פונה ל-API של Kubernetes.
עבור `iam`, הפלטפורמה חייבת להזריק או לספק בדרך אחרת את פרטי ההזדהות של
ה-workload identity. ServiceAccounts קיימים של ClickHouse או ClickHouse-backup
מתאימים כאשר הם עומדים במדיניות ה-cluster הזו.

צרו מסד נתונים עזר ייעודי ומשתמש ClickHouse ייעודי לפני ההרצה הראשונה.
החליפו את `<cluster-name>`, `<auxiliary-database>` ואת ממלא-המקום של הסיסמה.
השתמשו באותה קידומת `<auxiliary-database>.` ב-`COLLECTTABLEPREFIX`.

```sql
CREATE DATABASE IF NOT EXISTS <auxiliary-database>
ON CLUSTER <cluster-name>;

CREATE USER IF NOT EXISTS s3gc
ON CLUSTER <cluster-name>
IDENTIFIED WITH sha256_password BY '<strong-password>';

GRANT ON CLUSTER <cluster-name>
    SELECT ON system.* TO s3gc;

GRANT ON CLUSTER <cluster-name>
    SELECT, INSERT, CREATE TABLE ON <auxiliary-database>.* TO s3gc;

GRANT ON CLUSTER <cluster-name>
    REMOTE ON *.* TO s3gc;
```

ההרשאה על מסד הנתונים מכסה את טבלת המצאי העזר ואת טבלאות יומן ההרצות העמיד.
`REMOTE` מתיר את קריאות ה-`clusterAllReplicas` שמשמשות את ה-dry-run בסביבת
cluster ואת ה-preflight של המחיקה. אל תעניקו את הרשאת המקור `S3` של
ClickHouse: s3gc משתמש בלקוח S3 משלו וב-Secret או ב-workload identity במקום.

עבור `S3AUTH=static`, צרו את ה-Secret מקובץ פרטי הזדהות פרטי בסגנון shell.
שמרו את הקובץ הזה מחוץ ל-repository, הגבילו אותו לבעליו, ולעולם אל תדפיסו
אותו או תבצעו לו commit. במצב static, s3gc אינו קורא את `AWS_ACCESS_KEY_ID` או
`AWS_SECRET_ACCESS_KEY`. שמרו את הערכים האלה תחת השמות של s3gc במקום:

| משתנה AWS | מפתח ב-Secret |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | `S3GC_S3ACCESSKEY` |
| `AWS_SECRET_ACCESS_KEY` | `S3GC_S3SECRETKEY` |
| `AWS_SESSION_TOKEN` | `S3GC_S3SESSIONTOKEN` (אופציונלי) |

הקובץ חייב לבצע export לערכים האלה:

```bash
export S3GC_CHUSER=s3gc
export S3GC_CHPASS='<ClickHouse password>'
export S3GC_S3ACCESSKEY='<S3 access key>'
export S3GC_S3SECRETKEY='<S3 secret key>'
# export S3GC_S3SESSIONTOKEN='<S3 session token>'  # only for temporary credentials
```

צרו את ה-Secret. הפקודה הזו יוצרת אותו פעם אחת ומסרבת לדרוס Secret קיים:

```bash
set -a
. /private/path/s3gc-secrets.env
set +a

literals=(
  --from-literal=S3GC_CHUSER="$S3GC_CHUSER"
  --from-literal=S3GC_CHPASS="$S3GC_CHPASS"
  --from-literal=S3GC_S3ACCESSKEY="$S3GC_S3ACCESSKEY"
  --from-literal=S3GC_S3SECRETKEY="$S3GC_S3SECRETKEY"
)
if [ -n "${S3GC_S3SESSIONTOKEN:-}" ]; then
  literals+=(--from-literal=S3GC_S3SESSIONTOKEN="$S3GC_S3SESSIONTOKEN")
fi

kubectl -n <namespace> create secret generic <credentials-secret-name> \
  "${literals[@]}"

unset S3GC_CHUSER S3GC_CHPASS S3GC_S3ACCESSKEY S3GC_S3SECRETKEY S3GC_S3SESSIONTOKEN
```

ה-principal של S3 צריך `s3:ListBucket` על ה-bucket/הקידומת שבהיקף לצורך
האיסוף. Job מחיקה צריך בנוסף `s3:DeleteObject` על מפתחות האובייקטים שבהיקף.
החזיקו את הרשאת המחיקה בנפרד עד לשלב מחיקה מאושר.

### 4. הריצו כל שלב

עבור כל שלב, ערכו רק את הערכים שמוצגים למטה, רנדרו את ה-Job, אמתו אותו בצד
השרת (server-side), החילו אותו (apply), ועקבו אחרי הלוגים שלו:

```bash
.venv/bin/python deploy/kubernetes/render.py "$S3GC_RUN_DIR/s3gc.env" \
  > "$S3GC_RUN_DIR/<phase>.yaml"
kubectl apply --dry-run=server -f "$S3GC_RUN_DIR/<phase>.yaml"
kubectl apply -f "$S3GC_RUN_DIR/<phase>.yaml"
kubectl -n <namespace> logs -f job/<job-name>
```

| שלב | שינוי ב-`s3gc.env` | תוצאה נדרשת |
| --- | --- | --- |
| Collect (איסוף) | הגדירו `PHASE=collect` ו-`JOB_NAME` חדש. | מצאי של אובייקטי S3. לא מתבצעת מחיקה. |
| Review (סקירה) | הגדירו `PHASE=dry-run` ו-`JOB_NAME` חדש. | סקרו את מספר המועמדים, הבתים, ה-bucket והקידומת. |
| Delete (מחיקה) | אחרי אישור אנושי בכתב, הגדירו `PHASE=delete`, `JOB_NAME` חדש, ו-`DELETE_CONFIRMATION=DELETE_ORPHANS`. | מחיקה ו-checkpoint של אצוות שאושרו. |
| Verify (אימות) | הגדירו `PHASE=dry-run`, `JOB_NAME` חדש, ונקו את `DELETE_CONFIRMATION`. | דיווח על אפס מועמדים. |

אל תריצו מחיקה מ-dry-run שלא נסקר. לפני המחיקה המלאה הראשונה עבור image או
cluster חדשים, הגדירו את `USETOTAL` לכמה אלפי אובייקטים עבור מחיקה מוגבלת אחת.
נקו אותו לפני ההרצה המלאה.

### 5. התאוששות או אימות

אם Job מחיקה נכשל, צרו Job מחיקה חלופי עם `JOB_NAME` חדש ואותו
`COLLECTTABLEPREFIX`. אל תאספו מחדש אלא אם ההיקף השתנה.
`backoffLimit: 0` מונע ניסיונות חוזרים אוטומטיים.

אחרי מחיקה מוצלחת, הריצו את ה-dry-run של האימות. אל תכריזו שהניקוי הושלם עד
שהוא מדווח על אפס מועמדים.

<a id="configuration-reference"></a>

## מדריך תצורה

`render.py` קורא קובץ `KEY=VALUE` ללא סודות. הוא דוחה ערכי חובה חסרים, images
לא מקובעים, שלבים לא תקינים, ערכים בוליאניים לא תקינים וחלונות גיל לא בטוחים.
התחילו מ-[example.env](example.env).

| קבוצה | ערכי חובה | הערות |
| --- | --- | --- |
| Job | `JOB_NAME`, `NAMESPACE`, `IMAGE`, `IMAGE_PULL_SECRET`, `SERVICE_ACCOUNT`, `CREDENTIALS_SECRET` | `JOB_NAME` הוא תווית DNS והופך למזהה ההרצה העמיד. השאירו את `IMAGE_PULL_SECRET` ריק עבור ה-image הציבורי. |
| ClickHouse | `CHHOST`, `CHPORT`, `CLUSTERNAME`, `EXPECTED_REPLICAS`, `COLLECTTABLEPREFIX` | קבעו את `CHHOST` ל-replica אחד לאורך כל הניקוי. |
| מאגר אובייקטים | `S3IP`, `S3PORT`, `S3BUCKET`, `S3PATH`, `S3REGION`, `S3SECURE_FLAG`, `S3DISKNAME`, `S3AUTH`, `S3PROFILE` | `S3PROFILE` דורש `S3AUTH=aws`. |
| מגבלות | `SAMPLES`, `DELETE_BATCH_SIZE`, `USEAGE_HOURS`, `ACTIVE_DEADLINE_SECONDS`, `TTL_SECONDS_AFTER_FINISHED`, `MEMORY_REQUEST`, `MEMORY_LIMIT` | אל תשנו את `SAMPLES` אחרי האיסוף, לטובת partition pruning. |
| התנהגות | `PHASE`, `DELETE_CONFIRMATION`, `ORDER_BY_OBJPATH`, `VERBOSE` | ה-renderer דורש את טוקן האישור עבור מחיקה ועבור אוטומציית פיתוח. |

`USETOTAL` הוא אופציונלי ומגביל שלב use אחד. `RUNLOG` מוגדר כברירת מחדל
ל-`true`; הגדירו אותו ל-`false` רק כאשר stdout מהווה תיעוד תפעולי מספק.

<a id="durable-run-history"></a>

## היסטוריית הרצות עמידה

לוגים של pod נועדו לניטור, לא לראיה עמידה. ה-kubelet מבצע להם רוטציה, וה-TTL
של ה-Job מסיר את ה-pod. כברירת מחדל, `s3gc` כותב אירועי הרצה לטבלה
`<COLLECTTABLEPREFIX><S3DISKNAME>_log` ב-ClickHouse.

השתמשו באותו host המקובע ל-replica כדי לבחון הרצה:

```sql
SELECT event_time, phase, event, objects, bytes, message
FROM   <db>.<prefix><disk>_log
WHERE  run_id = '<job-name>'
ORDER BY event_time;
```

עבור מחיקה שנכשלה, בחנו את אירוע ה-`checkpoint` האחרון לפני שאתם מפעילים את
ה-Job החלופי. כשל בכתיבה ליומן ההרצה נסוג ל-stdout ואינו עוצר את הניקוי.

<a id="development-automation-only"></a>

## לאוטומציית פיתוח בלבד

`PHASE=dev-automation` מריץ collect, dry-run ו-delete ב-Job אחד. הוא מתחיל
ממצאי חדש ודורש `DELETE_CONFIRMATION=DELETE_ORPHANS`, `CLUSTERNAME`
ו-`EXPECTED_REPLICAS`.

השתמשו בו רק עבור fixtures מבודדים של פיתוח. הוא עוקף את שער הסקירה והאישור
הידני, ורק הוא רשאי להשתמש בחלון גיל של פחות מ-24 שעות.

<a id="preserved-safeguards"></a>

## אמצעי הגנה שנשמרים

- ה-manifest רץ כמשתמש שאינו root, עם מערכת קבצי שורש לקריאה בלבד וללא Linux capabilities.
- פרטי ההזדהות מגיעים מ-`CREDENTIALS_SECRET` או מ-workload identity, לעולם לא מה-image או מקובץ הסביבה.
- ה-renderer דורש image מקובע לפי digest ומאמת את אישור המחיקה לפני שה-Job מגיע ל-cluster.
- מחיקה דורשת את ה-preflight של cluster/replicas, מתעדת checkpoints, ולעולם אינה מנסה שוב אוטומטית.
