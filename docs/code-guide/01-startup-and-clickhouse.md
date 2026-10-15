[← חזרה לתוכן](README.md) | [חלק 2 →](02-s3-and-collect.md)

# חלק 1: מההפעלה ועד החיבור ל-ClickHouse

## שלב 0: מה שרץ עוד לפני שפונקציה כלשהי נקראת

כשמריצים `python s3gc.py ...`, ‏Python מבצע את הקובץ מלמעלה למטה. רוב ההכנות
קורות כבר כאן, ברמת המודול ולא בתוך פונקציה.

### `strtobool` ו-`coerce_bool` ([שורות 45-69](../../s3gc.py#L45-L69))

שתי פונקציות עזר להמרת ערך ל-`True`/`False`:
- **`strtobool`** מקבלת מחרוזת. `"yes"`, `"true"`, `"on"` ו-`"1"` הופכים ל-1,
  ו-`"no"`, `"false"`, `"off"` ו-`"0"` הופכים ל-0. כל ערך אחר זורק `ValueError`.
- **`coerce_bool`** מקבלת כל דבר. bool חוזר כמו שהוא, ריק או `None` הופך
  ל-`False`, מספר לפי ערכו, ומחרוזת עוברת דרך `strtobool`.

**למה צריך את זה:** משתנה סביבה תמיד מגיע כמחרוזת, ובפייתון כל מחרוזת לא ריקה
נחשבת אמת. בעבר `S3GC_DRYRUN_FLAG=false` הפך בפועל ל-**true**. ההערה
ב-[שורה 58](../../s3gc.py#L58) מתעדת את זה.

### בניית ה-parser ([שורות 71-545](../../s3gc.py#L71-L545))

```python
parser = ArgumentParser(usage=usage, env_prefix="S3GC", default_env=True, exit_on_error=False)
```

- **`env_prefix="S3GC"` + `default_env=True`:** כל אפשרות אפשר לתת גם כמשתנה
  סביבה `S3GC_<DEST באותיות גדולות>`. למשל `dest="chhost"` הופך ל-`S3GC_CHHOST`.
  כך ה-Job ב-Kubernetes מעביר את כל התצורה בלי דגלים.
- **`--cfg`** ([שורה 545](../../s3gc.py#L545)): קובץ תצורה (YAML), למשל
  `--cfg playground.yaml`.

**דפוס שחוזר בכל דגל בוליאני:** יש לו שתי צורות עם אותו `dest`:

```python
--dryrun        action="store_true"    # לשורת פקודה: הנוכחות שלו = True
--dryrunflag    type=coerce_bool       # לסביבה/קובץ: מקבל ערך true/false
```

הדגלים המרכזיים, לפי נושא:

| נושא | דגלים חשובים | ברירת מחדל |
| --- | --- | --- |
| ClickHouse | `chhost`, `chport`, `chuser`, `chpass`, `chtimeout` | ‏localhost, ‏8123, ‏default, ריק, ‏1800 שנ' |
| S3 | `s3ip`, `s3port`, `s3bucket`, `s3path`, `s3auth`, `s3secure` | ‏static, ללא TLS |
| S3 עמידות | `s3_connect_timeout`, `s3_read_timeout`, `s3_retries` | ‏15, ‏120, ‏3 |
| בחירת שלב | `collectonly`, `usecollected`, `dryrun` | כולם False |
| טבלת העזר | `collecttableprefix`, `s3diskname`, `samples`, `collectbatchsize` | ‏`s3objects_for_`, ‏`s3`, ‏4, ‏1024 |
| בטיחות | `useage`, `clustername`, `expected_replicas`, `non-interactive` | ‏24, ריק, אין, אינטראקטיבי |
| מחיקה | `deletebatchsize`, `use_remove_objects`, `usetotal`, `useafter` | ‏1000, ‏True, אין, אין |
| איסוף חלקי | `total`, `collectafter`, `age` | אין, אין, ‏0 |
| תחזוקה | `keepdata`, `drop_collecttable`, `create_database` | כולם False |
| יומן | `runlog`, `runid`, `verbose`, `debug`, `silent` | ‏True, ריק, ‏False |

### `args = parser.parse_args()` ([שורה 551](../../s3gc.py#L551))

**זו הפעולה האמיתית הראשונה בהרצה.** ה-parser מאחד את ברירות המחדל, קובץ
ה-`--cfg`, משתני `S3GC_*` ושורת הפקודה לאובייקט אחד, `args`. מכאן והלאה כל
הקוד קורא את התצורה ממנו (`args.chhost`, ‏`args.samples` וכו').

### נרמול בוליאנים ([שורות 557-586](../../s3gc.py#L557-L586))

לולאה על `BOOLEAN_DESTS` מעבירה כל דגל בוליאני דרך `coerce_bool`. אם הערך לא
חוקי, למשל `S3GC_DRYRUN_FLAG=maybe`, ההרצה נעצרת מיד עם הודעה ברורה. אחרי השלב
הזה שאר הקוד יכול לסמוך על כך שאלה bool אמיתיים.

### `--listoptions` ([שורות 588-609](../../s3gc.py#L588-L609))

כלי פנימי: מדפיס את כל האפשרויות כשורת `S3GC_...=...` ויוצא. בהרצה רגילה לא
נכנסים לכאן.

### הגדרת ה-logger ([שורות 611-618](../../s3gc.py#L611-L618))

| דגל | רמה | מה רואים |
| --- | --- | --- |
| (כלום) | WARNING | רק אזהרות ושגיאות |
| `--verbose` | INFO | התקדמות, שאילתות anti-join, checkpoints |
| `--debug` | DEBUG | **שורה לכל אובייקט.** בסקייל גדול זה מציף |
| `--silent` | CRITICAL | כמעט כלום. נבדק אחרון, ולכן גובר על השאר |

### `LogFormatter` ([שורות 622-651](../../s3gc.py#L622-L651))

formatter שמחליף סודות ב-`****` בכל שורת לוג. הנקודה העדינה: הרשימה
`filter_strings` מחושבת **בזמן הגדרת המחלקה**, ולא בכל שורת לוג. היא כוללת את
`chpass`, ‏`s3accesskey`, ‏`s3secretkey` ו-`s3sessiontoken`, רק אם הם ארוכים
מ-3 תווים.

`LogFormatter._filter` משמש גם את יומן ההרצות העמיד (ראו למטה), כדי שסוד לא
ייכתב לטבלה ב-ClickHouse.

### `graceful_exit` ([שורות 654-657](../../s3gc.py#L654-L657))

מדפיסה `s3gc: OK` (אלא אם `--silent`) ויוצאת עם קוד 0. זו ההודעה שמסמנת
שההרצה הצליחה.

### שמות הטבלאות ([שורות 669-688](../../s3gc.py#L669-L688))

`collecttableprefix` מתפצל לפי נקודה:

| קידומת | `tname` (טבלת העזר) | `log_tname` (יומן) |
| --- | --- | --- |
| `s3objects_for_` (בלי database) | `` `s3objects_for_<disk>` `` | `` `s3objects_for_<disk>_log` `` |
| `mydb.s3gc_run1_` (עם database) | `` `mydb`.`s3gc_run1_<disk>` `` | `` `mydb`.`s3gc_run1_<disk>_log` `` |
| `a.b.c` | שגיאה: `invalid collecttableprefix` | |

- **`<disk>` הוא `s3diskname`.** לכן שני דיסקים מקבלים שתי טבלאות שונות.
- **בלי שם database,** הטבלה נוצרת ב-database ברירת המחדל של משתמש ה-ClickHouse.
  לכן ה-runbook דורש קידומת מסויגת בשם database.

### הגדרות אחרונות לפני הריצה ([שורות 690-733](../../s3gc.py#L690-L733))

- **שלושת החיבורים** (`minio_client`, ‏`ch_client`, ‏`ch_writer`) מאותחלים
  ל-`None`, וייווצרו ב-`main()`.
- **`MINIMUM_USEAGE_HOURS = 24`:** רצפת חלון הגיל. ההסבר בחלק 3.
- **`S3DeletionError`:** השגיאה שנזרקת כשמחיקה ב-S3 נכשלת, אחרי שמה שהצליח כבר
  נרשם.

בסוף הקובץ ([שורה 1509](../../s3gc.py#L1509)):

```python
if __name__ == "__main__":
    main()
```

## שלב 1: `main()` ([שורות 1480-1506](../../s3gc.py#L1480-L1506)), המנצח על כל ההרצה

```python
connect_to_ch()                                    # 1. תמיד
init_run_log()                                     # 2. תמיד
run_log("start", ...)
if not (args.usecollected_flag and args.dryrun_flag):
    connect_to_s3()                                # 3. לא ב-dry-run על נתונים שנאספו
if not args.usecollected_flag:
    do_collect()                                   # 4. שלב 1
if not args.collectonly_flag:
    do_use()                                       # 5. שלב 2
run_log("finish", ...)
graceful_exit()
```

שלושת הדגלים קובעים את המסלול:

| מצב | דגלים | S3? | collect? | use? |
| --- | --- | --- | --- | --- |
| collect | `--collectonly` | כן (רישום) | ✓ | ✗ |
| dry-run | `--usecollected --dry-run` | **לא בכלל** | ✗ | ✓ ספירה בלבד |
| delete | `--usecollected` | כן (מחיקה) | ✗ | ✓ מחיקה |
| ללא דגלים | — | כן | ✓ | ✓ הכל בבת אחת |

**טיפול בשגיאות:**
- **`UserVisibleError`:** שגיאה "צפויה" למפעיל, כמו בעיית הרשאות ב-S3. נרשמת
  ליומן, מודפסת **בלי traceback** (אלא אם `--debug`), ויוצאת עם קוד 1.
- **כל שגיאה אחרת:** נרשמת ליומן העמיד, ואז נזרקת הלאה כמו שהיא, עם traceback
  מלא.
- **`graceful_exit()` משתמשת ב-`exit()`,** שזורק `SystemExit`, ו-`SystemExit`
  אינו `Exception`. לכן יציאה מוצלחת לא נתפסת כשגיאה.

## `connect_to_ch()` ([שורות 899-926](../../s3gc.py#L899-L926))

כותבת ללוג את פרטי החיבור, ואז יוצרת **שני** clients זהים של
`clickhouse_connect`:

- **`ch_client`:** לכל השאילתות הרגילות, כולל ה-stream הארוך של ה-anti-join.
- **`ch_writer`:** רק לכתיבות שקורות **בזמן** ש-`ch_client` עסוק ב-stream:
  tombstones ויומן ההרצות.

**למה שניים:** ClickHouse מתיר שאילתה אחת בכל רגע לכל session, ולכל client יש
session משלו. אם ה-tombstone היה נכתב על אותו client שמחזיק את ה-stream, השרת
היה מחזיר `SESSION_IS_LOCKED` (373). זה בדיוק הבאג של 0.6.0: האובייקטים נמחקו
מ-S3 אבל לא נרשמו, וההרצה מתה. ההערה ב-[שורה 925](../../s3gc.py#L925) אומרת
במפורש: "Do not simplify it away".

**`send_receive_timeout=args.chtimeout`** (ברירת מחדל 1800 שניות, חצי שעה) הוא
כמה זמן הלקוח מחכה לתשובה מ-ClickHouse. זה לא קשור ל-timeout של proxy או route
שיושב בדרך.

## יומן ההרצות העמיד ([שורות 713-846](../../s3gc.py#L713-L846))

**למה הוא קיים:** הלוגים של ה-pod עוברים רוטציה ונמחקים עם ה-Job, כך שאחרי ניקוי
ארוך לא נשארה ראיה למה שנעשה. היומן נכתב ל-ClickHouse ולא לקובץ או ל-S3:
ל-container יש מערכת קבצים לקריאה בלבד, וקובץ שנכתב ל-S3 תחת אותה קידומת היה
נאסף ב-collect הבא ונמחק כיתום.

### `init_run_log()` ([שורה 764](../../s3gc.py#L764))

אם `--runlog` דלוק:
1. קובעת `run_id` דרך `resolve_run_id()`.
2. מריצה `CREATE TABLE IF NOT EXISTS <log_tname>` (MergeTree, ממוין לפי
   `run_id, event_time`) על **`ch_writer`**.
3. אם זה נכשל, מוציאה אזהרה (`run log unavailable, continuing with stdout only`)
   וממשיכה בלי יומן. **לעולם לא מפילה את ההרצה.**

### `resolve_run_id()` ([שורה 756](../../s3gc.py#L756))

אם הועבר `--runid`, משתמשת בו. ב-Kubernetes ה-Job מעביר את `JOB_NAME`. אחרת
היא יוצרת מזהה מחותמת זמן ב-UTC ו-8 תווים אקראיים, למשל
`20261015T150335Z-a1b2c3d4`.

### `run_log(event, message, objects, bytes_, phase)` ([שורה 805](../../s3gc.py#L805))

מכניסה שורה אחת לטבלת היומן, עם הזמן, ה-`run_id`, השלב, האירוע, ההודעה (אחרי
השחרה דרך `LogFormatter._filter`), מונים, ופרטי ההיקף: bucket, ‏path, דיסק,
cluster, ‏dry-run, ‏host. שלושה כללים:
- **הכתיבה הולכת ל-`ch_writer`,** לעולם לא ל-`ch_client`, מאותה סיבה של
  `SESSION_IS_LOCKED`.
- **אם היומן כבוי** (`run_log_enabled=False`), היא לא עושה כלום.
- **כשל אחד מכבה את היומן** לשארית ההרצה, במקום לנסות שוב בכל אצווה.

האירועים שנכתבים לאורך ההרצה:

| `event` | מתי |
| --- | --- |
| `start` / `finish` | תחילת וסוף ההרצה, ותחילת וסוף כל שלב |
| `progress` | כל 100,000 אובייקטים באיסוף, ובתחילת כל sample |
| `checkpoint` | אחרי כל אצוות מחיקה מוצלחת |
| `warning` | אי-התאמה ב-samples, או חלון גיל קצר במצב פיתוח |
| `error` | כל כשל |

### `current_phase()` ([שורה 838](../../s3gc.py#L838))

מחזירה שם קריא של השלב לפי הדגלים: `collect`, ‏`dry-run`, ‏`delete` או
`collect+use`. משמשת רק להודעות ביומן.

## הערות ותצפיות מחלק 1

- **סיסמה קצרה עלולה להודפס גלויה.** `connect_to_ch` כותב ללוג
  `password={args.chpass}` במפורש ([שורה 901](../../s3gc.py#L901)), ורק
  `LogFormatter` מסתיר אותו. סיסמה של 3 תווים או פחות לא מסוננת.
- **התצורה נקראת בזמן ה-import.** לכן אי אפשר לייבא את `s3gc.py` כמודול רגיל
  בלי שהוא יקרא את `sys.argv`. הבדיקות עוקפות את זה עם `runpy.run_path` ב-`tests/conftest.py`.

[← חזרה לתוכן](README.md) | [חלק 2 →](02-s3-and-collect.md)
