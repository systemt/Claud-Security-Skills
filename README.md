# Claude Skills

סקילים מותאמים אישית ל-[Claude Code](https://claude.com/claude-code). כל סקיל הוא תיקייה עם `SKILL.md` ותיקיית `references/` עם הנחיות מפורטות שנטענות לפי הצורך.

## הסקילים

### `api-security-guard` — מוצא
אודיט והקשחת אבטחה לפרויקטים שמשתמשים במפתחות API של ספקי AI (OpenAI, Anthropic, Gemini וכו').

- סריקת מפתחות חשופים בקוד, ב-frontend ובהיסטוריית git
- בניית proxy בצד שרת שמסתיר את המפתח
- rate limiting, אימות, ניטור ותקרות שימוש
- עצירת חירום כשנמצא מפתח חשוף — rotation לפני כל תיקון

**מתי נטען:** "אודיט אבטחה", "תבדוק חשיפת מפתחות", "המפתח שלי דלף", "יש לי שימוש מטורף ב-API", "תגן על המפתחות".

**קבצים:** `SKILL.md`, `references/stack-notes.md`, `references/proxy-templates.md`

### `security-remediator` — מתקן
תיקון והקשחה של פרצות אבטחה שנמצאו באודיט, על מערכות חיות — לכן תמיד מדורג, עם עצירה לבדיקה אחרי כל דרג.

- auth/session, SQL injection, בידוד דיירים (tenant isolation)
- path traversal, אחסון סיסמאות, rate limiting, דליפת PII ותקרות DoS

**מתי נטען:** "תתקן את הפרצות", "תסגור את הממצאים מהאודיט", "fix security", "remediate", או תיאור של פרצה ספציפית.

**קבצים:** `SKILL.md` + `references/` — `auth-session.md`, `sql-injection.md`, `tenant-isolation.md`, `path-traversal.md`, `password-storage.md`, `hardening-dos.md`

השניים משלימים: `api-security-guard` מוצא, `security-remediator` מתקן.

## התקנה

לכל הפרויקטים (סקיל אישי):

```bash
git clone https://github.com/systemt/claude-skills.git
cp -r claude-skills/skills/* ~/.claude/skills/
```

לפרויקט בודד:

```bash
mkdir -p .claude/skills
cp -r /path/to/claude-skills/skills/api-security-guard .claude/skills/
```

Claude Code טוען את הסקיל אוטומטית כשהבקשה מתאימה ל-`description` שלו, או בהפעלה ידנית עם `/api-security-guard`.

## מבנה

```
skills/
├── api-security-guard/
│   ├── SKILL.md
│   └── references/
└── security-remediator/
    ├── SKILL.md
    └── references/
```
