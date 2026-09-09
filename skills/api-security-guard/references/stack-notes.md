# הערות ספציפיות לסטאק

קרא את הסעיף הרלוונטי לסטאק שזוהה בשלב 1.

## Node.js / Express

- סודות: `process.env` + חבילת `dotenv` (רק בפיתוח; בפרודקשן env אמיתי).
- Rate limiting: `express-rate-limit` (+ `rate-limit-redis` אם יש כמה instances).
- מלכודת נפוצה: קובץ `config.js` עם המפתח בפנים שמיובא גם ל-frontend דרך bundler.
- בדוק ש-`app.use(cors())` לא פתוח לכולם — הגדר `origin` מפורש.

## Next.js

- **מלכודת קריטית**: כל env var עם קידומת `NEXT_PUBLIC_` מוזרק ל-bundle של הדפדפן. מפתח AI לעולם לא יקבל קידומת זו. בדוק `next.config.js` על `env:` שמדליף.
- קריאות ל-AI: רק ב-Route Handlers (`app/api/*/route.ts`) או `pages/api/*` או Server Actions — לעולם לא ב-Client Component.
- בדוק שאין `'use client'` בקובץ שמחזיק את המפתח.
- Rate limiting: `@upstash/ratelimit` או middleware מותאם; ב-Vercel זכור שה-IP מגיע מ-`x-forwarded-for`.

## Vite / React SPA (כולל AppDeploy react-vite)

- **מלכודת קריטית**: כל env var עם קידומת `VITE_` נחשף לדפדפן. אותו כלל כמו NEXT_PUBLIC_.
- ל-SPA טהור אין backend — אם יש קריאת AI, חובה backend נפרד או פונקציית serverless. אין דרך מאובטחת לקרוא לספק AI ישירות מ-SPA.
- ב-AppDeploy: הקריאות עוברות דרך `backend/index.ts` עם feature `ai.generate` וכו' — המפתחות מנוהלים ע"י הפלטפורמה ב-secrets. וודא שה-frontend משתמש ב-`api` מ-`@appdeploy/client` ולא ב-fetch ישיר לספק.

## PHP / WordPress

- סודות: קבועים ב-`wp-config.php` (מעל שורת ה-stop editing) או `getenv()`. לעולם לא ב-functions.php של theme, לא בקוד plugin שנמצא ב-wp-content (נגיש לקריאה אם יש LFI/backup חשוף).
- קריאות AI: דרך `wp_remote_post()` בצד השרת, עם nonce verification (`check_ajax_referer`) על ה-AJAX endpoint.
- מלכודת: endpoints של `admin-ajax.php` עם `wp_ajax_nopriv_` — זה אנונימי לגמרי. אם ה-handler קורא ל-AI, כל האינטרנט יכול להפעיל אותו.
- Rate limiting: transients per-IP (`set_transient('ai_limit_' . $ip, ...)`) או plugin ייעודי.
- בדוק שאין קבצי backup חשופים: `wp-config.php.bak`, `.env` בשורש.

## Python (Flask/FastAPI)

- סודות: `os.environ` + `python-dotenv` בפיתוח.
- Rate limiting: `slowapi` (FastAPI) או `flask-limiter`.
- מלכודת: `settings.py`/`config.py` עם מפתח בקומיט.

## חתימות מפתחות לפי ספק (לסריקה)

| ספק | דפוס |
|-----|------|
| OpenAI | `sk-` / `sk-proj-` |
| Anthropic | `sk-ant-` |
| Google (Gemini/Maps) | `AIza...` |
| Meta/Facebook | `EAA...` (access tokens) |
| Green API / Evolution API | טוקנים בכתובות webhook — בדוק שה-webhook URL לא בקוד frontend |
| Stripe | `sk_live_` / `sk_test_` |
