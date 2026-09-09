# Hardening & DoS — תיקון

דרג 3 — לא דליפה פעילה, אלא הקשחה שמונעת ניצול לרעה, קריסה, וחשיפת מידע.

## 1. שאילתות בלי LIMIT — צריכת זיכרון לא חסומה
**הבעיה**: `get_messages()` / `list_lead_notes()` מושכות את כל השורות ל-זיכרון ומרנדרות HTML ענק. אלפי שורות = קריסה / DoS.
**תיקון**: LIMIT + pagination:
```python
def get_messages(self, conversation_id, limit=100, offset=0):
    limit = min(int(limit), 200)     # תקרה קשיחה, גם אם הלקוח ביקש יותר
    return self.db.execute(
        "SELECT * FROM messages WHERE conversation_id = ? "
        "ORDER BY created_at DESC LIMIT ? OFFSET ?",
        (conversation_id, limit, offset),
    ).fetchall()
```

## 2. /health חושף מידע רגיש בלי אימות
**הבעיה**: endpoint שמחזיר גרסת תוכנה, שימוש דיסק/RAM, עומק תור, ומצב per-client — מפת דרכים לתוקף.
**תיקון**: הפרד ל-two-tier —
```python
# ציבורי — רק "חי/מת", בלי פרטים
@app.route("/health")
def health():
    return {"status": "ok"}, 200

# מפורט — מאחורי אימות אדמין בלבד
@app.route("/health/detailed")
def health_detailed():
    require_admin()      # אותו מנגנון מ-auth-session.md
    return {"version": ..., "disk": ..., "queue": ...}, 200
```

## 3. Config ממוזג בלי validation — ערכים שליליים/ענקיים
**הבעיה**: followup config ממוזג מ-tenant spec בלי בדיקת טיפוס/גבולות → delay שלילי או step ענק גורמים לשיטפון הודעות WhatsApp.
**תיקון**: validate עם גבולות לפני שימוש:
```python
def validated_followup(raw):
    delay = int(raw.get("delay_minutes", 60))
    step  = int(raw.get("step_minutes", 30))
    if not (1 <= delay <= 10080):        # דקה עד שבוע
        raise ValueError("delay out of range")
    if not (1 <= step <= 1440):
        raise ValueError("step out of range")
    return {"delay_minutes": delay, "step_minutes": step}
```
עדיף עוד — schema מפורש (pydantic/marshmallow) עם bounds, ולא בדיקות ידניות.

## 4. PII בלוגים — טלפון + תוכן הודעה ב-INFO בלי מיסוך
**הבעיה**: הודעות נכנסות (מספר שולח + 80 תווים ראשונים) נכתבות ל-log ב-INFO. לוגים דולפים/נגישים = דליפת PII.
**תיקון**: מסך לפני כתיבה:
```python
def mask_phone(p):
    return p[:4] + "****" + p[-2:] if p and len(p) > 6 else "****"

logger.info("incoming msg from %s (len=%d)", mask_phone(sender), len(text))
# אל תרשום את תוכן ההודעה ב-INFO. אם צריך לדיבוג — DEBUG בלבד, כבוי בפרודקשן.
```

## 5. Impersonation בלי audit — גישה שקטה לכל דייר
**הבעיה**: אדמין נכנס לפורטל של דייר בלי שנרשם אירוע → אין עקבות למי ניגש למה.
**תיקון**: כתוב audit event על כל פעולה חוצת-דייר:
```python
def impersonate(self, admin_id, target_client_id):
    self.db.execute(
        "INSERT INTO audit_log (actor_id, action, target, ip, at) "
        "VALUES (?, 'impersonate', ?, ?, ?)",
        (admin_id, target_client_id, request.remote_addr, int(time.time())),
    )
    # ... המשך ה-impersonation
```
audit_log צריך להיות append-only ולא ניתן למחיקה מה-UI.

## בדיקות
1. משוך שיחה עם הרבה הודעות → מוגבל ל-LIMIT, לא תוקע זיכרון.
2. `curl DOMAIN/health` → רק `{"status":"ok"}`, בלי פרטים.
3. אדמין נכנס לדייר → מופיעה שורה ב-audit_log.
