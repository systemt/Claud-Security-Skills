# Tenant Isolation — תיקון

**הפרצה**: במערכת multi-tenant, שאילתה שנוגעת בנתוני דייר בלי לסנן לפי `client_id` (או ה-scope המקביל) מחזירה נתונים של דיירים אחרים. דוגמה קלאסית: `lookup_phone_cross_client()` שמחזירה טלפונים מכל הדיירים.

**עיקרון התיקון**: כל פונקציה שנוגעת בטבלה עם דאטה של דיירים **חייבת** לקבל `client_id` ולסנן לפיו ב-WHERE. אם היא לא מקבלת — זו פרצה, לא נוחות ארכיטקטונית.

## התיקון

**לפני (פגיע):**
```python
def lookup_phone(self, phone):
    return self.db.execute(
        "SELECT * FROM contacts WHERE phone = ?", (phone,)
    ).fetchall()
```

**אחרי (מבודד):**
```python
def lookup_phone(self, phone, client_id):
    return self.db.execute(
        "SELECT * FROM contacts WHERE phone = ? AND client_id = ?",
        (phone, client_id),
    ).fetchall()
```

## איך למצוא את כל המקומות

`client_id` שהתווסף לפונקציה אחת חייב להיות מועבר מכל מי שקורא לה. עברו על ה-call sites:
```bash
grep -rn "lookup_phone\|get_messages\|list_.*notes\|get_contacts" --include=*.py .
```
לכל קריאה — ודאו שיש `client_id` זמין בהקשר (מה-session של המשתמש המחובר) ומועבר.

## דפוס מומלץ — לאכוף במקום אחד

במקום לזכור `client_id` בכל שאילתה (שביר), עטפו את הגישה ל-DB ב-repository שמקבל `client_id` פעם אחת:

```python
class TenantRepo:
    def __init__(self, db, client_id):
        if not client_id:
            raise ValueError("client_id required — refusing unscoped query")
        self.db = db
        self.client_id = client_id

    def contacts_by_phone(self, phone):
        return self.db.execute(
            "SELECT * FROM contacts WHERE phone = ? AND client_id = ?",
            (phone, self.client_id),
        ).fetchall()
```

היתרון: אי אפשר בטעות להריץ שאילתה לא-מסוננת — ה-repo לא נבנה בלי `client_id`, וכל השאילתות עוברות דרכו.

## admin / impersonation
פעולות אדמין שחוצות דיירים במכוון (כמו impersonation) — מותרות, אבל **חייבות audit log** (ראה hardening-dos.md, ממצא audit). ההבחנה: cross-tenant של אדמין מתועד = תקין; cross-tenant של פונקציה רגילה = פרצה.

## בדיקה ידנית אחרי התיקון
1. התחבר כדייר A, נסה למשוך נתון של דייר B (למשל טלפון שקיים רק אצל B) — אמור להחזיר ריק/403.
2. ודא שדייר A עדיין רואה את הנתונים שלו עצמו (לא שברת את הזרימה הרגילה).
