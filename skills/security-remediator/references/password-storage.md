# Password Storage — תיקון

**הפרצה**: סיסמאות נשמרות כ-plaintext ב-DB ומושוות עם `==`. שני חטאים: (1) דליפת DB חושפת את כל הסיסמאות; (2) `==` הוא לא constant-time — מאפשר timing attack להשוואת סיסמה תו-תו.

**עיקרון**: לעולם לא שומרים סיסמה. שומרים hash חד-כיווני עם salt (bcrypt/argon2), ומשווים עם פונקציית verify של הספרייה (constant-time).

## התיקון

```python
import bcrypt

# בהרשמה / שינוי סיסמה — שמירה
def set_password(self, user_id, plain):
    hashed = bcrypt.hashpw(plain.encode(), bcrypt.gensalt(rounds=12))
    self.db.execute(
        "UPDATE users SET password_hash = ? WHERE id = ?",
        (hashed.decode(), user_id),
    )

# בלוגין — אימות (constant-time, מתקן גם את פרצת ה-== )
def verify_password(self, user_id, attempt):
    row = self.db.execute(
        "SELECT password_hash FROM users WHERE id = ?", (user_id,)
    ).fetchone()
    if not row:
        # הרץ hash דמה כדי לא לחשוף בזמן-תגובה אם המשתמש קיים
        bcrypt.checkpw(b"dummy", bcrypt.hashpw(b"dummy", bcrypt.gensalt()))
        return False
    return bcrypt.checkpw(attempt.encode(), row[0].encode())
```

## Migration — הסיסמאות הקיימות

זה החלק שרוב האנשים שוכחים. יש לך כבר סיסמאות plaintext ב-DB. אי אפשר "להצפין אותן אחורה" בלי הסיסמה — אבל אפשר ל-hash אותן פעם אחת בסקריפט migration:

```python
# migration_hash_passwords.py — הרצה חד-פעמית
import bcrypt, sqlite3

db = sqlite3.connect("app.db")
rows = db.execute("SELECT id, password FROM users").fetchall()
for user_id, plain in rows:
    if plain and not plain.startswith("$2b$"):     # דלג על מה שכבר עבר hash
        hashed = bcrypt.hashpw(plain.encode(), bcrypt.gensalt(rounds=12)).decode()
        db.execute("UPDATE users SET password_hash = ? WHERE id = ?", (hashed, user_id))
db.commit()
# אחרי אימות שהכל עבד — הסר את העמודה הישנה:
# ALTER TABLE users DROP COLUMN password;
```

**סדר בטוח:**
1. הוסף עמודת `password_hash` (בלי למחוק את `password` עדיין).
2. הרץ את ה-migration שממלא את `password_hash`.
3. שנה את קוד הלוגין לקרוא מ-`password_hash`.
4. בדוק שלוגין עובד עם משתמש אמיתי.
5. רק אז — מחק את עמודת `password` הישנה.

## Node
```javascript
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash(plain, 12);           // שמירה
const ok = await bcrypt.compare(attempt, storedHash); // אימות
```

## WordPress
משתמש כבר ב-`wp_hash_password()` / `wp_check_password()` מובנה — אם בנית auth ידני מעל WP, עבור למנגנון המובנה.

## בדיקה
1. אחרי migration — התחבר עם סיסמה קיימת של משתמש אמיתי, ודא שעובד.
2. הצץ ב-DB — עמודת ה-hash אמורה להתחיל ב-`$2b$`, לא סיסמה קריאה.
