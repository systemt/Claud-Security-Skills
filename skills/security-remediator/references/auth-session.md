# Auth & Session — תיקון

הדרג הכי מסוכן לשבור — טעות פה נועלת את כל המשתמשים בחוץ. הצע טסט לוגין לפני ומיד אחרי כל שינוי.

מקבץ ממצאים נפוצים ותיקון לכל אחד.

## 1. Session secret שנוצר מחדש בכל restart
**הבעיה**: הסוד שמחתים את הסשן נוצר אקראית באתחול → כל deploy מבטל את כל הסשנים, וגם אין יציבות.
**תיקון**: טען את הסוד מ-env, קבוע:
```python
app.secret_key = os.environ["SESSION_SECRET"]   # קבוע, לא os.urandom() בכל הרצה
```
צור אותו פעם אחת (`python -c "import secrets; print(secrets.token_hex(32))"`), שים ב-env של הפרודקשן.

## 2. טוקן דטרמיניסטי — אותו HMAC תמיד, בלי nonce/expiry
**הבעיה**: טוקן שהוא פונקציה קבועה של user_id → זהה לנצח, אין תפוגה, אין הגנת replay. מי שתפס אותו פעם אחת נכנס לתמיד.
**תיקון**: עבור ל-session מבוסס-שרת עם מזהה אקראי ותפוגה, במקום טוקן שנגזר מהמשתמש:
```python
import secrets, time

def create_session(self, user_id):
    sid = secrets.token_urlsafe(32)              # אקראי, לא נגזר מ-user_id
    expires = int(time.time()) + 60 * 60 * 8     # 8 שעות
    self.db.execute(
        "INSERT INTO sessions (sid, user_id, expires_at) VALUES (?, ?, ?)",
        (sid, user_id, expires),
    )
    return sid

def validate_session(self, sid):
    row = self.db.execute(
        "SELECT user_id, expires_at FROM sessions WHERE sid = ?", (sid,)
    ).fetchone()
    if not row or row[1] < time.time():
        return None
    return row[0]
```

## 3. Cookie בלי secure — נשלח על HTTP
```python
resp.set_cookie(
    "session", sid,
    secure=True,       # HTTPS בלבד
    httponly=True,     # לא נגיש ל-JS (הגנת XSS)
    samesite="Lax",    # הגנת CSRF בסיסית
    max_age=60 * 60 * 8,
)
```

## 4. Logout מוחק רק cookie בצד לקוח — השרת לא יכול לבטל טוקן גנוב
כשהסשן ב-store בשרת (סעיף 2), logout מוחק את הרשומה — וזה מבטל את הטוקן באמת:
```python
def logout(self, sid):
    self.db.execute("DELETE FROM sessions WHERE sid = ?", (sid,))
    # ובנוסף מחיקת ה-cookie בצד לקוח
```

## 5. Rate limiter שנעקף — מופעל רק כשמעבירים ip=
**הבעיה**: `_require_admin` שמפעיל rate limit רק אם הפרמטר `ip` הועבר, ורוב הנתיבים לא מעבירים → brute-force חופשי.
**תיקון**: קבע את ה-IP במקום אחד מרכזי, לא כפרמטר אופציונלי:
```python
def _require_admin(self):
    ip = request.headers.get("X-Forwarded-For", request.remote_addr)
    ip = ip.split(",")[0].strip()      # ה-IP האמיתי מאחורי proxy
    if not self._rate_ok(ip):          # תמיד נבדק, לא תלוי בפרמטר
        abort(429)
    # ... המשך אימות אדמין
```
הסר את הפרמטר `ip=` האופציונלי מכל החתימות — ה-IP נלקח מהבקשה, לא מהקורא.

## Node / Express
`express-session` עם store חיצוני (connect-redis/connect-pg), `cookie: { secure: true, httpOnly: true, sameSite: 'lax' }`, `express-rate-limit` עם `keyGenerator` שמבוסס IP אמיתי.

## בדיקה אחרי הדרג
1. התחבר → קבל cookie → גש לעמוד מוגן → מצליח.
2. Logout → נסה לגשת שוב עם אותו טוקן → נחסם (401).
3. אחרי deploy → משתמש שהיה מחובר עדיין מחובר (הסוד קבוע מ-env).
