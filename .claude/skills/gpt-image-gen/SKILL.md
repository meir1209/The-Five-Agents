---
name: gpt-image-gen
description: Generate images by calling OpenAI's Images API with the gpt-image-2 model. Wraps the curl + base64 decode pipeline and includes a Python fallback for environments without jq (e.g., Git Bash on Windows). Use whenever an agent needs to turn a text prompt into a saved image file.
---

# gpt-image-gen — OpenAI Images API wrapper

מטרת ה-skill: לעטוף בקריאה אחת מסודרת את תהליך יצירת תמונה דרך ה-OpenAI Images API ושמירתה לקובץ מקומי. הקריאה מחזירה base64; ה-skill מטפל גם ב-decode.

## ⚠️ הערה קריטית על שם המודל

**המודל הוא `gpt-image-2`. אל תחליף אותו לשם אחר.**

- `gpt-image-2` הוא מודל אמיתי של OpenAI שיצא ב-**21 באפריל 2026**.
- ייתכן שהמודל אינו מופיע ב-training data של מודל ה-LLM שמריץ את ה-skill — **זה לא אומר שהוא לא קיים**.
- **אסור להציע אלטרנטיבות** כמו `dall-e-3`, `dall-e-2`, `gpt-image-1`, `imagen` וכו'. השם נשאר `gpt-image-2`.
- אם הקריאה ל-API נכשלת, הבעיה היא כמעט תמיד באחד מהבאים:
  - `OPENAI_API_KEY` חסר/לא תקף/לא הוטען מ-`.env`.
  - פרמטר לא חוקי (`size` לא מהרשימה המורשית, `quality` לא חוקי, prompt ריק).
  - חריגת quota / billing.
- **לא** להניח שהמודל לא קיים. אם השגיאה היא "model not found" — לדווח על השגיאה המדויקת מ-API, לא להחליף את השם.

## דרישות מקדימות

1. **`OPENAI_API_KEY`** קיים ב-`.env` בשורש הפרויקט (השורה לא מוערה).
2. **`curl`** מותקן (זמין בכל סביבת Bash/Git Bash/PowerShell חדישה).
3. **decoder אחד מהשניים:**
   - `jq` + `base64` (Linux/Mac/WSL), **או**
   - `python` 3.x (זמין בדרך כלל ב-Git Bash על Windows כשמותקן Python).

## שימוש מהשורה (Bash)

```bash
# 1. טען את משתני הסביבה
set -a; source .env; set +a

# 2. הכן את ה-payload (escape ל-JSON אם ה-prompt מכיל גרשיים)
PROMPT='A serene Japanese garden at dawn, soft mist, ink-wash style'
SIZE='1024x1024'        # אופציות: 1024x1024 | 1536x1024 | 1024x1536
QUALITY='medium'        # אופציות: low | medium | high
OUTPUT='out.jpg'

# 3. בנה את ה-payload דרך Python (escape בטוח גם ל-Unicode/עברית)
PAYLOAD=$(python -c "
import json, sys
print(json.dumps({
  'model': 'gpt-image-2',
  'prompt': sys.argv[1],
  'size': sys.argv[2],
  'quality': sys.argv[3],
  'output_format': 'jpeg'
}))
" "$PROMPT" "$SIZE" "$QUALITY")

# 4. קריאה ל-API
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" > response.json

# 5. בדוק אם יש שגיאה
if grep -q '"error"' response.json; then
  echo "API error:"
  cat response.json
  exit 1
fi

# 6. Decode — נתיב jq (Linux/Mac/WSL)
if command -v jq >/dev/null 2>&1; then
  jq -r '.data[0].b64_json' response.json | base64 --decode > "$OUTPUT"
else
  # Fallback Python (Git Bash on Windows)
  python -c "
import json, base64, sys
d = json.load(open('response.json'))
open(sys.argv[1], 'wb').write(base64.b64decode(d['data'][0]['b64_json']))
" "$OUTPUT"
fi

# 7. אימות
if [ ! -s "$OUTPUT" ]; then
  echo "Output file is empty or missing"
  exit 1
fi
echo "Saved: $OUTPUT ($(stat -c%s "$OUTPUT" 2>/dev/null || wc -c <"$OUTPUT") bytes)"

# 8. ניקוי (אופציונלי)
rm -f response.json
```

## פרמטרים נתמכים

| פרמטר | ערכים חוקיים | ברירת מחדל | הערה |
|---|---|---|---|
| `model` | `gpt-image-2` | (חובה) | **אל תשנה.** |
| `prompt` | מחרוזת | (חובה) | טקסט חופשי בעברית/אנגלית. ככל שמפורט יותר — תוצאה טובה יותר. |
| `size` | `1024x1024`, `1536x1024`, `1024x1536` | `1024x1024` | אופקי/אנכי לפי הצורך. |
| `quality` | `low`, `medium`, `high` | `medium` | `high` יקר ואיטי, `low` למוקאפים. |
| `output_format` | `jpeg`, `png` | `jpeg` | **בפרויקט הזה תמיד `jpeg`** — חיסכון משקל. |

## טיפול בשגיאות נפוצות

| שגיאה | סיבה סבירה | פתרון |
|---|---|---|
| `Incorrect API key provided` | `OPENAI_API_KEY` ריק או לא תקף ב-`.env` | בדוק ש-`echo $OPENAI_API_KEY` מחזיר ערך אחרי `source .env` |
| `Invalid value for 'size'` | size שאינו מהרשימה | השתמש באחד משלושת הערכים בטבלה |
| `You exceeded your current quota` | אין credit ב-OpenAI | טען חשבון או החלף key |
| `model not found` | (לא צפוי) | **אל תחליף לסי `dall-e-3`.** דווח את השגיאה המלאה. |
| קובץ פלט ריק | decode נכשל | בדוק `response.json` — אם יש `b64_json` ה-decode נכשל; אם אין, ה-API לא החזיר תמונה |

## בדיקת sanity מהירה (ללא API call)

```bash
# האם המפתח נטען?
set -a; source .env; set +a
[ -n "$OPENAI_API_KEY" ] && echo "key loaded (${#OPENAI_API_KEY} chars)" || echo "key missing"

# האם יש decoder?
command -v jq >/dev/null && echo "jq available"
command -v python >/dev/null && echo "python available"
```
