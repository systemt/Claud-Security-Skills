# SQL Injection — תיקון

שלושה סוגים נפרדים, לכל אחד תיקון אחר. הכלל העל: **ערכים → placeholders. שמות (עמודות/טבלאות) → whitelist.** אי אפשר לפרמטר שם עמודה, ולכן שם עמודה שמגיע מקלט חייב להיבדק מול רשימה סגורה.

## 1. Value injection — קלט לא פרמטרי

**לפני:**
```python
db.execute(f"SELECT * FROM users WHERE email = '{email}'")   # פגיע
```
**אחרי:**
```python
db.execute("SELECT * FROM users WHERE email = ?", (email,))  # sqlite
# psycopg: db.execute("... WHERE email = %s", (email,))
```

## 2. Column-name injection — שמות עמודות מקלט

הפרצה: `update_client()` שבונה `SET` clause מהמפתחות של dict שהמשתמש שולח. אי אפשר לפרמטר שם עמודה, אז חובה whitelist.

**לפני:**
```python
def update_client(self, client_id, data):
    clause = ", ".join(f"{k} = ?" for k in data)   # k מגיע מהמשתמש!
    self.db.execute(f"UPDATE clients SET {clause} WHERE id = ?",
                    (*data.values(), client_id))
```
**אחרי:**
```python
ALLOWED_COLUMNS = {"name", "email", "phone", "status", "notes"}

def update_client(self, client_id, data):
    cols = [k for k in data if k in ALLOWED_COLUMNS]   # רק שמות מותרים
    if not cols:
        raise ValueError("no valid columns to update")
    clause = ", ".join(f"{c} = ?" for c in cols)       # שמות מ-whitelist בלבד
    values = [data[c] for c in cols]
    self.db.execute(f"UPDATE clients SET {clause} WHERE id = ?",
                    (*values, client_id))
```
שים לב: הערכים עדיין עוברים כ-`?` (פרמטרים), רק שמות העמודות מגיעים מה-whitelist.

## 3. LIKE injection — מטא-תווים לא-escaped

הפרצה: קלט משתמש ב-`LIKE` בלי לברוח מ-`%` ו-`_`. `%` בקלט = "החזר הכל" → dump של טבלאות.

**לפני:**
```python
db.execute("SELECT * FROM contacts WHERE name LIKE ?", (f"%{term}%",))
```
**אחרי:**
```python
def escape_like(s):
    return s.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_")

pattern = f"%{escape_like(term)}%"
db.execute(
    "SELECT * FROM contacts WHERE name LIKE ? ESCAPE '\\'",
    (pattern,),
)
```
ה-`ESCAPE '\'` אומר ל-SQL להתייחס ל-`\%` כתו אחוז מילולי ולא כ-wildcard.

## Node / Express
```javascript
// value: prepared
pool.query('SELECT * FROM users WHERE email = $1', [email]);
// column: whitelist
const ALLOWED = new Set(['name', 'email', 'phone']);
const cols = Object.keys(data).filter(k => ALLOWED.has(k));
```

## WordPress
```php
$wpdb->get_results($wpdb->prepare(
    "SELECT * FROM $table WHERE email = %s", $email
));
// LIKE:
$like = '%' . $wpdb->esc_like($term) . '%';
$wpdb->prepare("... WHERE name LIKE %s", $like);
```

## בדיקה
נסה קלט `' OR '1'='1` בשדה חיפוש/לוגין → אמור לא להחזיר שורות מיותרות. נסה `%` בחיפוש → אמור לחפש את התו אחוז מילולית, לא להחזיר הכל.
