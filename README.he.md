[English](README.md) | **עברית**

# s3gc

`s3gc` מאתר אובייקטים יתומים (orphaned objects) בדיסק S3 של ClickHouse או
במאגר אובייקטים תואם. הכלי אוסף מצאי (inventory) לתוך ClickHouse, משווה אותו
מול `system.remote_data_paths`, ומדווח על אובייקטים ש-ClickHouse אינו מפנה
אליהם.

לעבודה מול לקוחות ובסביבת ייצור, השתמשו ב-runbook של Kubernetes:

```text
collect → dry-run → explicit human approval → delete → verify
```

(איסוף ← הרצה יבשה ← אישור אנושי מפורש ← מחיקה ← אימות)

קראו את [מריץ ה-Job של Kubernetes](deploy/kubernetes/README.he.md) לפני שאתם
מריצים ניקוי. זהו נוהל הייצור המחייב.

## גבול הבטיחות

מחיקת אובייקט היא בלתי הפיכה. `s3gc` שומר על מנגנוני הבקרה הבאים:

- Job של מחיקה דורש את טוקן האישור `DELETE_ORPHANS`.
- מחיקה בסביבת cluster בודקת את שם ה-cluster המוגדר ואת מספר ה-replicas הצפוי
  לפני שהיא פונה ל-S3.
- Job מחיקה שנכשל אינו מנסה שוב אוטומטית. אצוות (batches) שאושרו נשמרות
  ב-checkpoint, כך ש-Job חלופי יכול להמשיך מאותה נקודה עם אותה קידומת של טבלת
  האיסוף (collection-table prefix).
- `USEAGE` חייב להיות לפחות 24 שעות מחוץ לאוטומציית הפיתוח. חלון הגיל הזה מגן
  על part ש-S3 כבר קיבל לפני ש-ClickHouse רשם אותו.
- ה-renderer של Kubernetes דורש image מקובע לפי digest ושומר את פרטי ההזדהות
  (credentials) מחוץ ל-manifest.

לניקוי ברמת כל ההתקנה (installation-wide), אספו את קידומת-האב (parent prefix)
שמכילה את האובייקטים של כל ה-replicas והגדירו את שם ה-cluster, כך שההשוואה
תכסה את כל ה-replicas. קידומת של replica בודד מפספסת את היתומים של שאר
ה-replicas. אספו קידומת-אב רק אם היא מכילה אובייקטים של `S3DISKNAME` אחד בלבד —
בדיקת היתומים מתאימה לפי שם הדיסק, ולכן אובייקטים של דיסק אחר תחת אותה קידומת
ייראו יתומים.

השתמשו בקידומת ייחודית לטבלת האיסוף עבור כל bucket וקידומת. לעולם אל תבצעו
commit לפרטי הזדהות, manifests מרונדרים, פרטי ה-cluster היעד, או פלט של הרצות.

## התקנה מקומית ותצוגה מקדימה לקריאה בלבד

התקינו את התלויות המקובעות (pinned) ובדקו את האפשרויות הזמינות:

```bash
python3.11 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt -r requirements-dev.txt
.venv/bin/python s3gc.py --help
```

הגדירו את התצורה באמצעות דגלים (flags) או משתני סביבה מסוג `S3GC_*`. תצורה
מינימלית ללא סודות נראית כך:

```bash
export S3GC_CHHOST='<per-replica-clickhouse-host>'
export S3GC_CHPORT=8123
export S3GC_CHUSER='<clickhouse-user>'
export S3GC_S3IP='s3.eu-central-1.amazonaws.com'
export S3GC_S3PORT=443
export S3GC_S3BUCKET='<bucket>'
export S3GC_S3PATH='<target-prefix>/'
export S3GC_S3REGION='eu-central-1'
export S3GC_S3SECURE_FLAG=true
export S3GC_S3DISKNAME=s3
export S3GC_CLUSTERNAME='<clickhouse-cluster>'
export S3GC_EXPECTED_REPLICAS=2
export S3GC_COLLECTTABLEPREFIX='s3gc_example_'
export S3GC_USEAGE=24
```

הזריקו סיסמאות ומפתחות ממנהל סודות (secret manager) או מה-shell שלכם. אל
תשמרו אותם בקובץ או בהיסטוריית הפקודות. לאחר מכן הריצו תצוגה מקדימה:

```bash
.venv/bin/python s3gc.py --verbose --dry-run
```

השתמשו במחיקה ישירה רק לעבודת פיתוח מבוקרת. ה-runbook של Kubernetes מפריד בין
איסוף, סקירה, אישור, מחיקה ואימות.

## הזדהות ומאגרי אובייקטים

בחרו מצב הזדהות אחד ל-S3 באמצעות `S3GC_S3AUTH`:

| מצב | פרטי הזדהות | מתי להשתמש |
| --- | --- | --- |
| `static` | `S3GC_S3ACCESSKEY` ו-`S3GC_S3SECRETKEY`; טוקן session אופציונלי | פרטי הזדהות מפורשים ממנהל סודות |
| `aws` | שרשרת boto3; `S3GC_S3PROFILE` אופציונלי | AWS SSO או profile בעל שם בתחנת עבודה |
| `iam` | ספק workload-identity של MinIO | EKS IRSA, EC2 instance profiles, או ECS task roles |

מצב static אינו קורא את `AWS_ACCESS_KEY_ID` או `AWS_SECRET_ACCESS_KEY`. העבירו
את הערכים האלה בתור `S3GC_S3ACCESSKEY` ו-`S3GC_S3SECRETKEY`.

`S3GC_S3PROFILE` בוחר במצב `aws`; הכלי דוחה הגדרות סותרות. תצוגה מקדימה במצב
`aws` עדיין דורשת `s3:ListBucket` עבור ה-bucket והקידומת המוגדרים.

GCS משתמש במפתחות HMAC לתאימות (interoperability), ובדרך כלל דיסק האובייקטים
נקרא `gcs`. הוא אינו תומך במחיקת אצווה של S3, ולכן `s3gc` נסוג למחיקה איטית
יותר של אובייקט-אחר-אובייקט. הגדירו `S3GC_S3DISKNAME=gcs`; אל תשתמשו בדיסק
`*_cache` כהיקף הייחוס.

## פריסה ב-Kubernetes

מריץ ה-Kubernetes נמצא ב-[deploy/kubernetes](deploy/kubernetes/). הוא משתמש
ב-Job חד-פעמי, בקובץ סביבה ללא סודות, וב-Secret של Kubernetes או ב-workload
identity עבור פרטי ההזדהות.

התחילו מהקישורים האלה:

1. [הרצת תהליך הייצור](deploy/kubernetes/README.he.md#runbook)
2. [הגדרת ה-Job](deploy/kubernetes/README.he.md#configuration-reference)
3. [העתקת הדוגמה ללא סודות](deploy/kubernetes/example.env)

images משוחררים זמינים באופן ציבורי ב-`ghcr.io/altinity/s3gc`. השתמשו תמיד
ב-digest:

```bash
docker pull ghcr.io/altinity/s3gc@sha256:<digest>
```

ה-CI מפרסם את הערך המדויק של `IMAGE=` בסיכום ה-job שלו. ה-renderer דוחה
תגיות image משתנות (mutable tags).

## היסטוריית הרצות עמידה

כברירת מחדל, כל הרצה כותבת אירועים מובנים לטבלה
`<COLLECTTABLEPREFIX><S3DISKNAME>_log` ב-ClickHouse. הטבלה מתעדת תחילת שלבים,
התקדמות האיסוף, checkpoints של מחיקה, סיכומים, אזהרות ושגיאות. היא שורדת את
ה-Job ואת לוגי ה-pod שעוברים רוטציה.

שאלו על הרצה אחת באותו host של ClickHouse שמקובע ל-replica:

```sql
SELECT event_time, phase, event, objects, bytes, message
FROM   <db>.<prefix><disk>_log
WHERE  run_id = '<job-name>'
ORDER BY event_time;
```

הגדירו `RUNLOG=false` רק כאשר stdout מהווה תיעוד מספק. כשל ביומן ההרצה נסוג
ל-stdout ואינו עוצר את הניקוי.

## Skill לסוכן AI

[`skills/altinity-clickhouse-s3gc`](skills/altinity-clickhouse-s3gc/) ([תרגום](skills/altinity-clickhouse-s3gc/SKILL.he.md))
נותן לסוכן את אותם גבולות בטיחות כמו ה-runbook של Kubernetes. התקינו אותו על
ידי יצירת symlink או העתקת התיקייה לתיקיית ה-skills של הסוכן. עבור Codex:

```bash
mkdir -p ~/.codex/skills
ln -s /path/to/s3gc/skills/altinity-clickhouse-s3gc \
      ~/.codex/skills/altinity-clickhouse-s3gc
```

ה-skill לעולם אינו מספק את אישור המחיקה. אדם מורשה חייב לאשר את תוצאת
ה-dry-run שנסקרה.

## תהליך פיתוח בעזרת AI

`CLAUDE.md` (מדריך מלא) ו-`AGENTS.md` (הפניה קצרה) מסבירים לסוכני קוד מבוססי
AI איך לעבוד כאן. עבור פיצ'ר חדש, תיקון באג או שינוי התנהגות:

1. **Spec Intake (קליטת אפיון)** — אם הבקשה אינה מפורטת מספיק, הסוכן שואל מה
   צריך לקרות במקום, איך לעורר את המצב, האם זה נוגע במחזור החיים של המחיקה,
   ומהם מקרי קצה ידועים.
2. **Spec (אפיון)** — הגדרת בעיה קצרה וקריטריוני קבלה, ב-commit שמתחיל את
   השינוי.
3. **TDD** — בדיקה נכשלת לכל קריטריון קבלה, ולאחר מכן מימוש שגורם לה לעבור.

שינויים עריכתיים (תיעוד, הערות) פטורים מזה. ראו את `CLAUDE.md` לכללים
המלאים.

## בדיקות פיתוח

הריצו בדיקות offline:

```bash
.venv/bin/python -m pytest -v -m "not dev_cluster"
```

רנדרו ואמתו את ה-Job לדוגמה בלי לפנות ל-cluster:

```bash
.venv/bin/python deploy/kubernetes/render.py deploy/kubernetes/example.env > /tmp/s3gc-job.yaml
docker run --rm --entrypoint /kubeconform -v /tmp:/tmp:ro \
  ghcr.io/yannh/kubeconform@sha256:85dbef6b4b312b99133decc9c6fc9495e9fc5f92293d4ff3b7e1b30f5611823c \
  -strict -summary /tmp/s3gc-job.yaml
```

ראו את [CLAUDE.md](CLAUDE.he.md) לדרישות לתורמים,
את [CHANGELOG.md](CHANGELOG.he.md) להיסטוריה תפעולית, ואת [TODO.md](TODO.he.md)
לעבודה שנדחתה.
