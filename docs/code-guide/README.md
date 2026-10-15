# מדריך לקוד של s3gc

מדריך לימוד שמסביר את `s3gc.py` פונקציה אחרי פונקציה, **לפי סדר הריצה**.

> מספרי השורות מתייחסים ל-`s3gc.py` בגרסה של commit `7801fd4` (ה-master של
> Altinity בזמן הכתיבה). אם הקובץ משתנה, ייתכן שהמספרים יזוזו. הגרסה של
> החברה שונה (למשל `--unordered` ו-Ceph), כך שמספרי השורות בה לא יתאימו.

## סדר הקריאה

| חלק | נושא | מה תבין בסוף |
| --- | --- | --- |
| [1. הפעלה וחיבור ל-ClickHouse](01-startup-and-clickhouse.md) | מה קורה מרגע ה-`python s3gc.py` ועד שיש חיבור ל-ClickHouse ויומן הרצות | איך התצורה נטענת, איך נבחר השלב, למה יש שני clients |
| [2. חיבור ל-S3 ואיסוף](02-s3-and-collect.md) | הזדהות, בניית ה-client של S3, ושלב 1: איסוף ה-inventory לטבלת העזר | איך נבנית טבלת העזר ומה נכנס אליה |
| [3. התאמה ומחיקה](03-use-and-delete.md) | שלב 2: ה-anti-join, ה-modulo, המחיקה, ה-tombstones וההמשכה אחרי כשל | איך נקבע מה יתום, ואיך מחיקה שנכשלה ממשיכה בבטחה |

## התמונה הגדולה בעמוד אחד

```text
python s3gc.py ...
   │
   ├─ [import time]  parser → args → נרמול בוליאנים → logger → שמות טבלאות
   │
   └─ main()
        ├─ connect_to_ch()        שני clients: ch_client + ch_writer
        ├─ init_run_log()         CREATE TABLE <prefix><disk>_log
        ├─ connect_to_s3()        (לא ב-dry-run על נתונים שנאספו)
        ├─ do_collect()           שלב 1: S3 → טבלת העזר   (אם לא --usecollected)
        ├─ do_use()               שלב 2: anti-join → מחיקה (אם לא --collectonly)
        └─ graceful_exit()        "s3gc: OK"
```

| שלב ב-Kubernetes | דגלים שה-entrypoint מעביר | S3? | collect | use |
| --- | --- | --- | --- | --- |
| `collect` | `--collectonly --keepdata` | רישום | ✓ | ✗ |
| `dry-run` | `--usecollected --dry-run` | **לא בכלל** | ✗ | ספירה בלבד |
| `delete` | `--usecollected --keepdata --non-interactive` | מחיקה | ✗ | ✓ |

## מילון מונחים קצר

| מונח | פירוש |
| --- | --- |
| אובייקט יתום (orphan) | אובייקט ב-S3 ש-ClickHouse כבר לא מפנה אליו |
| `system.remote_data_paths` | טבלת מערכת ב-ClickHouse שמפרטת לאילו אובייקטים ב-S3 כל דיסק מפנה |
| טבלת העזר (auxiliary table) | הטבלה ש-s3gc יוצר ב-ClickHouse ובה רשימת כל האובייקטים שנמצאו ב-S3 |
| anti-join | "תביא לי שורות משמאל שאין להן התאמה מימין", כלומר מה שקיים ב-S3 ולא מוזכר ב-ClickHouse |
| sample | אחד מ-N החלקים (לפי modulo) שטבלת העזר מחולקת אליהם |
| tombstone | שורה עם `active=false` שנכתבת אחרי מחיקה מוצלחת, ומסמנת שהאובייקט טופל |
| preflight | בדיקה לפני מחיקה ש-cluster ומספר ה-replicas הם מה שמצפים |
| replica-pinned | חיבור ל-replica אחד וקבוע, ולא ל-Service עם איזון עומסים |
