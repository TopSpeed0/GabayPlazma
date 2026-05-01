# 📺 ShulScreen — מסמך תכנון מקיף

> מסך דיגיטלי לבתי כנסת: זמני תפילה, תורמים, חגים, שיעורים
> תאריך: 2026-05-02
> סטטוס: תכנון

---

## 1. חזון המוצר

### מה זה?
אפליקציית web שמציגה על מסך טלוויזיה בבית כנסת:
- 🕐 זמני תפילה (שחרית, מנחה, ערבית, שבת) — מחושבים אוטומטית לפי מיקום
- 🌅 זמנים הלכתיים (הנץ, שקיעה, צאת הכוכבים, זמן ק"ש, חצות)
- 📅 תאריך עברי + פרשת שבוע + חגים + ספירת העומר
- 🎓 שיעורים קבועים (מגיד שיעור, נושא, שעה)
- 💰 תורמים ונדבות (שם + סכום/הקדשה + תאריך)
- 📢 הודעות קהילה (אירועים, מודעות, הזכרות)

### למי זה?
גבאים ורבנים שרוצים מסך מקצועי בבית כנסת, בלי לעדכן ידנית דפים מודפסים.

### מודל עסקי
**SaaS — מנוי חודשי/שנתי per בית כנסת.**
- הלקוח מקבל URL ייחודי למסך שלו
- בלי מנוי תקף → המסך לא עובד
- ניהול דרך פאנל אדמין (מובנה או נפרד)

---

## 2. ארכיטקטורה

### תרשים כללי

```
[טלוויזיה + Stick/מחשב]
    │
    ▼
[https://shulscreen.co.il/s/<ID>]    ← מסך התצוגה (readonly)
    │
    ├── GET /api/screen/<ID>          ← נתוני המסך (JSON)
    ├── GET /api/times/<location>     ← זמנים הלכתיים (חישוב/API)
    └── WebSocket                     ← עדכונים בזמן אמת
    
[https://shulscreen.co.il/admin]      ← פאנל ניהול לגבאי
    │
    ├── POST /api/auth/login          ← התחברות
    ├── CRUD /api/lessons             ← שיעורים
    ├── CRUD /api/donors              ← תורמים
    ├── CRUD /api/announcements       ← הודעות
    └── PUT /api/screen/<ID>/config   ← הגדרות מסך

[https://shulscreen.co.il/superadmin] ← ניהול מנויים (שלנו)
    │
    ├── CRUD /api/subscriptions       ← הנפקה/ביטול מנויים
    ├── GET /api/licenses             ← צפייה ברישיונות
    └── POST /api/license/generate    ← הנפקת רישיון חדש
```

### Stack

| שכבה | טכנולוגיה | הסבר |
|------|-----------|------|
| Frontend — מסך | HTML + CSS + Vanilla JS | קל, מהיר, בלי framework overhead |
| Frontend — אדמין | HTML + CSS + Vanilla JS (או Vue lite) | ממשק ניהול פשוט |
| Backend | Node.js + Express | API + auth + license validation |
| DB | SQLite (לייט) או PostgreSQL (סקייל) | מנויים, משתמשים, נתונים |
| Auth | JWT + bcrypt | גבאי login + API tokens |
| License | RSA signed certificates | רישיון חתום דיגיטלית |
| Hosting | שרת שלנו (nginx) | srv1392988 |
| SSL | Let's Encrypt | HTTPS חובה |
| עיצוב | Black Liquid Glass 🖤 | מסך TV — שחור חוסך חשמל + נראה מדהים |

---

## 3. מערכת רישוי (License System)

### 3.1 עקרון

כל בית כנסת מקבל **רישיון חתום דיגיטלית (RSA)**. בלי רישיון תקף — המסך מציג "מנוי לא פעיל".

### 3.2 זרימה

```
[SuperAdmin] → יוצר רישיון → חותם עם Private Key → שומר ב-DB
    │
    ▼
[Screen loads] → GET /api/license/<ID> → מקבל רישיון חתום
    │
    ▼
[Client validates] → Public Key (embedded) → בודק חתימה + תוקף
    │
    ├── ✅ תקף → מציג מסך
    └── ❌ פג/מזויף → מציג "נא לחדש מנוי"
```

### 3.3 מבנה רישיון

```json
{
  "licenseId": "uuid",
  "screenId": "shul-xyz",
  "orgName": "בית כנסת אהבת ישראל",
  "plan": "yearly",
  "features": ["times", "donors", "lessons", "announcements"],
  "issuedAt": "2026-05-01T00:00:00Z",
  "expiresAt": "2027-05-01T00:00:00Z",
  "issuer": "ShulScreen Ltd"
}
// + RSA signature over the JSON
```

### 3.4 אבטחה

| שכבה | מנגנון |
|------|--------|
| חתימה | RSA-2048+ (Private Key רק אצלנו, Public Key ב-client) |
| תקשורת | HTTPS only |
| API | JWT token per גבאי, API key per מסך |
| Rate limit | לפי IP + API key |
| License check | כל טעינת מסך + כל 6 שעות ברקע |
| Tamper detection | אם הרישיון לא עובר אימות חתימה → מסך חסום |

### 3.5 הצפנת קוד מקור (Client-Side)

הקוד שרץ אצל הלקוח (על ה-TV) צריך להיות **מוצפן/מעורפל**:

| טכניקה | כלי | מה עושה |
|--------|-----|---------|
| **Obfuscation** | javascript-obfuscator | שינוי שמות משתנים, הסרת whitespace, control flow flattening |
| **String encryption** | javascript-obfuscator stringArray | מחרוזות (URLs, keys) מוצפנות ומפוענחות ב-runtime |
| **Anti-debug** | debugger traps + timing checks | מונע DevTools inspection |
| **Domain lock** | בדיקת window.location | הקוד עובד רק מהדומיין שלנו |
| **License binding** | הקוד דורש token מהשרת לפני render | בלי license תקף → אין קוד לראות |

**אסטרטגיה מומלצת:**
1. **Build step**: `javascript-obfuscator` ברמה high + domain lock
2. **Runtime**: הקוד העיקרי נטען מהשרת רק אחרי אימות license (לא סתם script tag סטטי)
3. **API-first**: הלוגיקה הרגישה (חישוב זמנים, אימות) בצד שרת

```
Browser loads index.html (minimal shell)
  → JS validates license with server
  → Server returns encrypted app bundle
  → JS decrypts + evals
  → App renders
```

---

## 4. מסך התצוגה (Display)

### 4.1 Layout (1920×1080, landscape)

```
┌──────────────────────────────────────────────────────────┐
│  🏛️ שם בית הכנסת                    📅 י"ב אייר תשפ"ו  │
│                                      🕐 20:45            │
├──────────────┬───────────────────────┬───────────────────┤
│              │                       │                   │
│  🕐 זמנים    │   💰 תורמים           │  🎓 שיעורים       │
│              │                       │                   │
│  הנץ  5:42  │  ר' משה כהן           │  גמרא  - 19:00   │
│  שחרית 6:30 │  לעילוי נשמת אביו    │  הרב לוי         │
│  מנחה 19:15 │                       │                   │
│  שקיעה 19:32│  ר' דוד לוי           │  הלכה  - 20:00   │
│  ערבית 19:50│  לרפואת שרה בת רחל   │  הרב כהן         │
│  צה"כ 20:10 │                       │                   │
│              │  ר' אברהם ישראלי     │  פרשת שבוע 21:00 │
│  זמן ק"ש    │  נדבה לבית הכנסת     │  הרב ישראלי      │
│  8:45       │                       │                   │
│              │                       │                   │
├──────────────┴───────────────────────┴───────────────────┤
│  📢 הודעות: שבת חזון — דרשה מיוחדת אחרי מוסף           │
│  📢 שיעור נשים — יום שלישי 10:00 בחדר השיעורים          │
├──────────────────────────────────────────────────────────┤
│  פרשת השבוע: בהעלותך  │  ספירת העומר: יום 27           │
└──────────────────────────────────────────────────────────┘
```

### 4.2 תכונות תצוגה

| תכונה | פירוט |
|-------|-------|
| **רוטציה** | תורמים/הודעות מתחלפים כל 10-15 שניות (slider) |
| **זמנים מהבהבים** | 5 דקות לפני תפילה → הזמן מהבהב/מודגש |
| **מצב שבת** | עיצוב מיוחד לשבת, ללא עדכונים (pre-loaded) |
| **מצב חג** | עיצוב חגיגי + ברכות חג |
| **Auto-refresh** | WebSocket או polling כל 5 דקות |
| **Screen saver** | אם אין שינוי 2 שעות → אנימציה עדינה (שומר מסך) |
| **RTL** | הכל בעברית, ימין לשמאל |
| **Responsive** | תומך 1080p, 4K, ו-portrait mode |
| **Full screen** | F11 או auto-fullscreen API |

### 4.3 ערכות נושא (Themes)

| ערכה | תיאור |
|------|--------|
| **Black Glass** (default) | Black Liquid Glass — שחור עמוק, זכוכית, זוהר |
| **Shabbat Gold** | גוונים חמים, זהב, רקע כהה |
| **Yom Tov** | חגיגי, צבעים בהירים יותר |
| **Avel** | מצב אבלות — שחור לבן, מינימלי |
| **Custom** | הגבאי בוחר צבעים |

---

## 5. זמנים הלכתיים

### 5.1 מקורות

| מקור | API | עלות | מה נותן |
|------|-----|------|---------|
| **Hebcal** | hebcal.com/zmanim | חינם | זמני היום, פרשה, חגים |
| **KosherJava/KosherZmanim** | npm package | חינם | חישוב מקומי, כל הזמנים |
| **MyZmanim** | myzmanim.com | חינם | זמנים לפי עיר |

**המלצה**: `KosherZmanim` (npm) לחישוב מקומי + `Hebcal` API לתאריכים עבריים/פרשות.

### 5.2 זמנים שמוצגים

| זמן | חישוב |
|-----|-------|
| עלות השחר | 72 דקות לפני הנץ |
| הנץ החמה | sunrise |
| סוף זמן ק"ש (מג"א) | 3 שעות זמניות מעלות השחר |
| סוף זמן ק"ש (גר"א) | 3 שעות זמניות מהנץ |
| סוף זמן תפילה | 4 שעות זמניות |
| חצות היום | midday |
| מנחה גדולה | 30 דק אחרי חצות |
| מנחה קטנה | 3.5 שעות לפני שקיעה |
| פלג המנחה | 1.25 שעות לפני שקיעה |
| שקיעה | sunset |
| צאת הכוכבים | 13.5 דקות / 8.5° אחרי שקיעה (מותאם אישית) |
| חצות הלילה | midnight |

### 5.3 זמני תפילה

הגבאי מגדיר זמנים קבועים **או** יחסיים:
- שחרית: "6:30" (קבוע) או "הנץ + 0" (וותיקין)
- מנחה: "שקיעה - 20 דקות"
- ערבית: "צאת הכוכבים" או "20:00" (קבוע)
- שבת: זמנים נפרדים (קבלת שבת, מנחה שבת, שחרית שבת)

---

## 6. פאנל ניהול (Admin)

### 6.1 דפים

| דף | תוכן |
|----|-------|
| **Dashboard** | סטטוס מסך, מנוי, סטטיסטיקות |
| **זמני תפילה** | הגדרת זמנים קבועים/יחסיים |
| **תורמים** | CRUD תורמים + הקדשות + תאריך תפוגה |
| **שיעורים** | CRUD שיעורים + ימים + שעות + מגיד שיעור |
| **הודעות** | CRUD הודעות + תאריך תפוגה + עדיפות |
| **עיצוב** | בחירת ערכת נושא + לוגו בית כנסת + צבעים |
| **הגדרות** | מיקום (lat/lon), שם בית כנסת, מנהג (אשכנז/ספרד/תימן) |
| **Preview** | תצוגה מקדימה של המסך |

### 6.2 ניהול תורמים

```
שם תורם: [                    ]
סוג:     ⊙ לעילוי נשמת  ⊙ לרפואת  ⊙ נדבה  ⊙ הקדשה  ⊙ אחר
הקדשה:   [                    ]
סכום:    [        ] ₪  (אופציונלי — להצגה או להסתרה)
תצוגה:   ☑ הצג במסך
מתאריך:  [2026-05-01]
עד:      [2026-06-01]  (אוטומטי — חודש, שבוע, או קבוע)
```

### 6.3 ניהול שיעורים

```
נושא:     [גמרא מסכת ברכות      ]
מגיד שיעור: [הרב משה כהן         ]
ימים:     ☑ ראשון ☑ שלישי ☐ חמישי
שעה:      [19:00]
מיקום:    [בית המדרש              ] (אופציונלי)
פעיל:     ☑
```

---

## 7. SuperAdmin — ניהול מנויים (שלנו)

### 7.1 יכולות

| פעולה | פירוט |
|-------|-------|
| **הנפקת מנוי** | יצירת screenId + license + user credentials |
| **חידוש מנוי** | הארכת תוקף license |
| **ביטול מנוי** | revocation — המסך מפסיק לעבוד |
| **צפייה** | כל המנויים, סטטוסים, תאריכי תפוגה |
| **תמחור** | הגדרת תוכניות (חודשי/שנתי/enterprise) |
| **סטטיסטיקות** | כמה מסכים פעילים, הכנסות, שימוש |

### 7.2 תוכניות

| תוכנית | מחיר (הצעה) | כולל |
|---------|-------------|------|
| **Basic** | 49₪/חודש | זמנים + תורמים + שיעורים |
| **Pro** | 99₪/חודש | הכל + הודעות + ערכות נושא + לוגו |
| **Enterprise** | בהתאמה | מספר מסכים + API + התאמה אישית |

---

## 8. אבטחה — סיכום שכבות

```
Layer 1: HTTPS (Let's Encrypt)
    │
Layer 2: License validation (RSA signed cert)
    │
Layer 3: API authentication (JWT + API keys)
    │
Layer 4: Code obfuscation (javascript-obfuscator)
    │
Layer 5: Dynamic code loading (code served only after license check)
    │
Layer 6: Domain lock (code works only from our domain)
    │
Layer 7: Anti-debug (DevTools detection + timing)
    │
Layer 8: Rate limiting (per IP + API key)
    │
Layer 9: CORS (only our domain)
```

---

## 9. מבנה קבצים (תכנון)

```
/var/www/shulscreen.co.il/
├── public/                    # Static files (display)
│   ├── index.html             # Minimal shell (loads app after license check)
│   ├── css/style.css          # Display styles
│   ├── js/
│   │   ├── loader.js          # License check + dynamic app loading (obfuscated)
│   │   └── app.js             # Main display app (served dynamically, obfuscated)
│   └── assets/                # Icons, fonts, default images
│
├── admin/                     # Admin panel
│   ├── index.html
│   ├── css/admin.css
│   └── js/admin.js
│
├── server/                    # Backend
│   ├── index.js               # Express entry point
│   ├── routes/
│   │   ├── auth.js            # Login/register/JWT
│   │   ├── screen.js          # Screen data API
│   │   ├── times.js           # Zmanim calculation
│   │   ├── admin.js           # CRUD operations
│   │   ├── license.js         # License validation
│   │   └── superadmin.js      # Subscription management
│   ├── models/
│   │   ├── user.js
│   │   ├── screen.js
│   │   ├── donor.js
│   │   ├── lesson.js
│   │   ├── announcement.js
│   │   └── license.js
│   ├── middleware/
│   │   ├── auth.js            # JWT verification
│   │   ├── license.js         # License check middleware
│   │   └── rateLimit.js
│   └── keys/
│       ├── private.pem        # RSA private key (SERVER ONLY!)
│       └── public.pem         # RSA public key (embedded in client)
│
├── scripts/
│   ├── generate-license.js    # CLI: generate signed license
│   ├── obfuscate.js           # Build: obfuscate client JS
│   └── seed-db.js             # Dev: seed sample data
│
├── deploy.sh                  # ← web-deploy-skill
├── staging.sh                 # ← web-deploy-skill
├── test.sh                    # ← web-deploy-skill
├── .deploy.env                # ← web-deploy-skill
├── PIPELINE.md                # ← web-deploy-skill
├── CHANGELOG.md               # ← web-deploy-skill
├── BUGS.md                    # ← web-deploy-skill
├── AI.MD                      # Rebuild spec
└── package.json
```

---

## 10. דומיין ותשתית

| פריט | ערך |
|------|-----|
| **דומיין** | shulscreen.co.il (לרכוש) או תת-נתיב זמני |
| **שרת** | srv1392988 (Hostinger) — להתחלה |
| **SSL** | Let's Encrypt |
| **Web server** | nginx reverse proxy → Node.js |
| **DB** | SQLite (phase 1) → PostgreSQL (scale) |
| **Process manager** | pm2 או systemd |
| **Backup** | daily DB backup → gdrive |
| **Monitoring** | uptime check + error logging |

---

## 11. שלבי פיתוח (Roadmap)

### Phase 1: MVP (2-3 שבועות)
- [ ] מסך תצוגה בסיסי (זמנים + תורמים + שיעורים)
- [ ] חישוב זמנים הלכתיים (KosherZmanim)
- [ ] תאריך עברי + פרשת שבוע (Hebcal)
- [ ] Backend API בסיסי (Express + SQLite)
- [ ] Admin panel פשוט (CRUD)
- [ ] License system (RSA)
- [ ] deploy pipeline (web-deploy-skill)
- [ ] Black Liquid Glass design

### Phase 2: Polish (1-2 שבועות)
- [ ] ערכות נושא (Shabbat Gold, Yom Tov, Avel)
- [ ] רוטציית תורמים/הודעות
- [ ] הבהוב זמני תפילה
- [ ] מצב שבת (pre-loaded, no network)
- [ ] WebSocket live updates
- [ ] Code obfuscation build step
- [ ] SuperAdmin panel

### Phase 3: Production (1 שבוע)
- [ ] דומיין shulscreen.co.il
- [ ] מסך דמו לשיווק
- [ ] דף נחיתה (landing page)
- [ ] תיעוד לגבאים (PDF)
- [ ] תמחור + תשלום (PayBox/Tranzila/Stripe)

### Phase 4: Scale
- [ ] אפליקציית ניהול (PWA)
- [ ] Multi-screen per בית כנסת
- [ ] התראות WhatsApp/Telegram לגבאי
- [ ] אינטגרציה עם Google Calendar
- [ ] Marketplace ערכות נושא
- [ ] White-label לרשתות בתי כנסת

---

## 12. סיכום טכנולוגיות

| צורך | פתרון |
|------|-------|
| זמנים הלכתיים | KosherZmanim (npm) — חישוב מקומי |
| תאריך עברי | Hebcal API — פרשות, חגים, מועדים |
| License | RSA-2048 signed JSON certificates |
| Auth | JWT + bcrypt |
| Code protection | javascript-obfuscator + dynamic loading + domain lock |
| Real-time | WebSocket (ws) / SSE |
| DB | better-sqlite3 (phase 1) |
| Deploy | web-deploy-skill (deploy.sh + staging.sh + test.sh) |
| Design | Black Liquid Glass (RTL, Hebrew) |

---

## 13. שאלות פתוחות

1. **דומיין** — shulscreen.co.il? shulscreen.com? synagogue-screen.co.il?
2. **תשלום** — PayBox (ישראלי)? Tranzila? Stripe?
3. **אחסון** — להישאר על Hostinger או לעבור ל-VPS יותר חזק?
4. **מנהגים** — כמה וריאציות לזמנים? (אשכנז/ספרד/תימן/חב"ד)
5. **Portrait mode** — יש בתי כנסת עם מסך אנכי?
6. **Offline mode** — מה קורה אם אין אינטרנט? cache מקומי?
7. **שם המוצר** — ShulScreen? שלט הקהילה? מסך בית הכנסת?
