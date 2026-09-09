# Path Traversal — תיקון

**הפרצה**: נתיב שמקבל שם קובץ מהמשתמש ומגיש אותו בלי לוודא שהוא נשאר בתוך התיקייה המותרת. `../../../.env` יוצא מהתיקייה וקורא קבצי שרת רגישים.

**עיקרון התיקון**: לנרמל את הנתיב הסופי (`resolve`) ולוודא שהוא **מתחיל** בנתיב הבסיס המותר. בנוסף — לנקות את שם הקובץ מסלאשים ו-`..`.

## Python / Flask

```python
from pathlib import Path
from werkzeug.utils import secure_filename
import os

IMAGES_ROOT = Path("/app/static/images").resolve()

@app.route("/static/images/<client_id>/<filename>")
def serve_image(client_id, filename):
    # 1. נקה את שם הקובץ — מסיר .. וסלאשים
    filename = secure_filename(filename)
    if not filename:
        abort(404)

    # 2. נקה גם את client_id (מגיע מהמשתמש!)
    client_id = secure_filename(client_id)

    # 3. בנה את הנתיב המלא ונרמל
    target = (IMAGES_ROOT / client_id / filename).resolve()

    # 4. ודא שלא יצאנו מהתיקייה המותרת — זו הבדיקה הקריטית
    if not str(target).startswith(str(IMAGES_ROOT) + os.sep):
        abort(403)

    # 5. ודא שזה קובץ קיים
    if not target.is_file():
        abort(404)

    return send_file(target)
```

**למה שתי הגנות (secure_filename + startswith)**: `secure_filename` לבד לא מספיק לכל edge case, ו-`startswith` לבד לא חוסם symlinks בצורה מושלמת. יחד הם fail-closed.

## Node / Express

```javascript
const path = require('path');
const fs = require('fs');

const IMAGES_ROOT = path.resolve('/app/static/images');

app.get('/static/images/:clientId/:filename', (req, res) => {
  const clientId = path.basename(req.params.clientId);   // מסיר נתיבים
  const filename = path.basename(req.params.filename);
  const target = path.resolve(IMAGES_ROOT, clientId, filename);

  if (!target.startsWith(IMAGES_ROOT + path.sep)) {
    return res.sendStatus(403);
  }
  if (!fs.existsSync(target) || !fs.statSync(target).isFile()) {
    return res.sendStatus(404);
  }
  res.sendFile(target);
});
```

## PHP / WordPress

```php
$root = realpath('/app/static/images');
$target = realpath($root . '/' . basename($client_id) . '/' . basename($filename));

if ($target === false || strpos($target, $root . DIRECTORY_SEPARATOR) !== 0) {
    http_response_code(403);
    exit;
}
readfile($target);
```

## אחרי התיקון — חובה
פרצת traversal שמגיעה ל-.env אומרת שכל סוד בקובץ הזה **דלף בהנחת ברירת מחדל**. אחרי סגירת הפרצה:
1. Rotation לכל מפתח/טוקן/סוד שהיה ב-.env (מפתחות AI, DB password, session secret).
2. הרמת השרת של הפרודקשן שיטען את הסודות החדשים.
3. בדיקה: נסה לגלוש ל-`https://DOMAIN/static/images/../../.env` — אמור להחזיר 403.
