# 🚀 Upstash Redis Setup Guide

**Účel:** Rate limiting pre ochranu pred DDoS a cost explosion  
**Čas:** 5 minút  
**Cena:** FREE (10,000 príkazov/deň)

---

## 📝 Setup Steps

### 1. Vytvorte Upstash účet
1. Otvorte: https://upstash.com
2. Kliknite **Sign Up** (alebo Sign in with GitHub)
3. Potvrďte email

### 2. Vytvorte Redis databázu
1. V Upstash dashboarde kliknite **Create Database**
2. Nastavenia:
   - **Name:** `lectio-ratelimit` (alebo iný názov)
   - **Region:** Vyberte najbližší región (napr. `EU-Central-1` pre Európu)
   - **Type:** `Regional` (stačí na development aj production)
   - **TLS:** Enabled (automaticky)
3. Kliknite **Create**

### 3. Skopírujte credentials
Po vytvorení databázy uvidíte:
- **REST URL:** `https://xxxxxxxx.upstash.io`
- **REST TOKEN:** `AxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxQ==`

### 4. Pridajte do .env.local
Otvorte `.env.local` a pridajte:

```bash
# Upstash Redis (Rate Limiting)
UPSTASH_REDIS_REST_URL=https://your-db.upstash.io
UPSTASH_REDIS_REST_TOKEN=AxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxQ==
```

**POZOR:** Nahraďte `your-db` a token skutočnými hodnotami!

### 5. Reštartujte dev server
```bash
npm run dev
```

---

## ✅ Testovanie Rate Limitingu

### Test 1: Denný limit (10 článkov/deň)
```bash
# Pošlite 11 requests na generovanie článkov
# 11. request by mal byť odmietnutý s 429 Too Many Requests
for i in {1..11}; do
  curl -X POST http://localhost:3000/api/ai-generate-article \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer YOUR_TOKEN" \
    -d '{
      "topic": "Test článok '$i'",
      "articleType": "biblical_commentary",
      "length": "short",
      "targetLang": "sk"
    }'
  echo "\n\nRequest $i done\n"
  sleep 1
done
```

Očakávaná odpoveď po 11. requeste:
```json
{
  "error": "Dosiahli ste denný limit generovania článkov",
  "message": "Môžete generovať max 10 článkov za deň. Skúste to zajtra alebo upgradujte na premium.",
  "limit": 10,
  "resetIn": "24 hodín"
}
```

### Test 2: Hodinový limit (10 requests/hodinu)
```bash
# Pošlite 11 requests rýchlo za sebou
# 11. request by mal byť odmietnutý s 429 Too Many Requests
```

Očakávaná odpoveď:
```json
{
  "error": "Rate limit exceeded",
  "message": "You have exceeded the rate limit. Please try again in 60 minutes.",
  "limit": 10,
  "remaining": 0,
  "reset": "2025-11-24T15:30:00.000Z",
  "resetIn": "60 minutes"
}
```

---

## 📊 Monitorovanie v Upstash Dashboard

Po spustení aplikácie môžete v Upstash dashboarde vidieť:
- **Commands/day:** Počet príkazov (každý rate limit check = 1 príkaz)
- **Data:** Uložené rate limit countery
- **Latency:** Rýchlosť odozvy (zvyčajne < 50ms)

### Limity Free Tier:
- ✅ 10,000 príkazov/deň (stačí pre ~5,000 API requestov)
- ✅ 256 MB storage
- ✅ Regional deployment
- ✅ TLS encryption
- ⚠️ Po prekročení 10,000 príkazov sa žiadosti throttle (neplatíte nič navyše)

---

## 🔍 Troubleshooting

### Chyba: "Failed to connect to Upstash Redis"
**Príčina:** Nesprávne environment variables

**Riešenie:**
1. Skontrolujte `.env.local` - sú tam správne URL a TOKEN?
2. Reštartujte dev server: `npm run dev`
3. Skontrolujte, či je databáza v Upstash aktívna

### Chyba: "Rate limit not working"
**Príčina:** Možné cache/storage problémy

**Riešenie:**
1. V Upstash dashboarde kliknite **Data Browser**
2. Vymažte všetky kľúče začínajúce `ratelimit:*`
3. Skúste znova

### Rate limit sa resetuje po reštarte servera?
**Odpoveď:** Nie! Upstash Redis je perzistentný - rate limit countery sa uchovávajú aj po reštarte aplikácie (na rozdiel od in-memory riešení).

---

## 💡 Ďalšie kroky

Po úspešnom nastavení Upstash Redis máte:
- ✅ Rate limiting na AI endpoints (10 článkov/deň)
- ✅ Ochranu pred DDoS útokmi
- ✅ Ochranu pred cost explosion (€15 kredit nemôže byť minutý za pár minút)
- ✅ Hodinový limit (10 requests/hodinu)

**Next:** Spustite SQL migráciu pre AI usage tracking!

```sql
-- Otvorte Supabase SQL Editor a spustite:
-- sql/create_ai_usage_tracking.sql
```

To vytvorí tabuľku `ai_usage_logs` pre monitoring nákladov.

---

## 📚 Dokumentácia

- Upstash Docs: https://docs.upstash.com/redis
- Rate Limit Package: https://github.com/upstash/ratelimit
- Pricing: https://upstash.com/pricing
