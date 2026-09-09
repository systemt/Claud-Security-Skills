# תבניות Proxy מאובטח לקריאות AI

עקרונות משותפים לכל התבניות:
1. המפתח מ-env בלבד.
2. אימות משתמש לפני הכל.
3. model, max_tokens, system prompt — קבועים בשרת. הלקוח שולח רק את התוכן.
4. הגבלת אורך קלט.
5. Rate limit לפני הקריאה לספק.
6. לוג אחרי הקריאה.

## Node.js / Express

```javascript
import rateLimit from 'express-rate-limit';

const aiLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 10, // 10 בקשות לדקה
  keyGenerator: (req) => req.user?.id || req.ip,
  message: { error: 'Too many requests' },
});

app.post('/api/ai/analyze', requireAuth, aiLimiter, async (req, res) => {
  const { text } = req.body;
  if (!text || typeof text !== 'string' || text.length > 2000) {
    return res.status(400).json({ error: 'Invalid input' });
  }

  // תקרה יומית per-user (דוגמה עם מונה ב-DB/Redis)
  const usedToday = await getDailyUsage(req.user.id);
  if (usedToday >= 50) return res.status(429).json({ error: 'Daily limit reached' });

  const r = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'gpt-4o-mini',        // קבוע בשרת
      max_tokens: 500,              // קבוע בשרת
      messages: [
        { role: 'system', content: SYSTEM_PROMPT }, // קבוע בשרת
        { role: 'user', content: text },
      ],
    }),
  });
  const data = await r.json();

  await logAiCall({
    userId: req.user.id, ip: req.ip,
    tokens: data.usage?.total_tokens, endpoint: 'analyze',
  });

  res.json({ result: data.choices?.[0]?.message?.content ?? '' });
});
```

## Next.js (App Router Route Handler)

```typescript
// app/api/ai/route.ts — רץ בשרת בלבד
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';

export async function POST(req: NextRequest) {
  const session = await getServerSession();
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const { text } = await req.json();
  if (!text || text.length > 2000) {
    return NextResponse.json({ error: 'Invalid input' }, { status: 400 });
  }

  const allowed = await checkRateLimit(session.user.id); // Upstash/Redis
  if (!allowed) return NextResponse.json({ error: 'Rate limited' }, { status: 429 });

  const r = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.OPENAI_API_KEY}`, // בלי NEXT_PUBLIC_!
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'gpt-4o-mini',
      max_tokens: 500,
      messages: [{ role: 'user', content: text }],
    }),
  });
  const data = await r.json();
  return NextResponse.json({ result: data.choices?.[0]?.message?.content ?? '' });
}
```

## WordPress (AJAX handler מאובטח)

```php
// בתוך plugin — לא ב-theme
add_action('wp_ajax_ai_analyze', 'df_ai_analyze'); // מחוברים בלבד. אין nopriv!

function df_ai_analyze() {
    check_ajax_referer('df_ai_nonce', 'nonce');

    $text = sanitize_textarea_field($_POST['text'] ?? '');
    if (empty($text) || mb_strlen($text) > 2000) {
        wp_send_json_error('Invalid input', 400);
    }

    // Rate limit per-user עם transient
    $user_id = get_current_user_id();
    $key = 'df_ai_limit_' . $user_id;
    $count = (int) get_transient($key);
    if ($count >= 10) wp_send_json_error('Rate limited', 429);
    set_transient($key, $count + 1, MINUTE_IN_SECONDS);

    $response = wp_remote_post('https://api.openai.com/v1/chat/completions', [
        'timeout' => 30,
        'headers' => [
            'Authorization' => 'Bearer ' . (defined('OPENAI_API_KEY') ? OPENAI_API_KEY : getenv('OPENAI_API_KEY')),
            'Content-Type'  => 'application/json',
        ],
        'body' => wp_json_encode([
            'model'      => 'gpt-4o-mini',
            'max_tokens' => 500,
            'messages'   => [['role' => 'user', 'content' => $text]],
        ]),
    ]);

    if (is_wp_error($response)) wp_send_json_error('Upstream error', 502);
    $data = json_decode(wp_remote_retrieve_body($response), true);
    wp_send_json_success(['result' => $data['choices'][0]['message']['content'] ?? '']);
}
```

המפתח ב-`wp-config.php`:
```php
define('OPENAI_API_KEY', 'sk-...'); // מעל שורת ה-"stop editing"
```

## AppDeploy (backend/index.ts)

ב-AppDeploy אין צורך ב-proxy ידני לספקים הנתמכים — feature `ai.generate` עושה את זה עם מפתחות מנוהלים. אם צריך ספק חיצוני אחר: המפתח נכנס דרך `secrets` feature (create_secret_entry), נקרא ב-backend בלבד, וה-frontend קורא דרך `api` מ-`@appdeploy/client`. לעולם לא fetch ישיר מה-frontend לספק.
