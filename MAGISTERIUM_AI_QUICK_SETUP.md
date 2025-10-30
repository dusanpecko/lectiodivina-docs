# Magisterium AI - Quick Setup Guide

## 🚀 Rýchle Nastavenie

### 1. Získanie API Kľúča

1. Navštívte https://www.magisterium.com/developers
2. Vytvorte účet (Sign Up)
3. Prejdite do API Console → API Keys
4. Vygenerujte nový API kľúč
5. Skopírujte kľúč (začína na `sk_...`)

### 2. Konfigurácia v Lectio.one

Pridajte do `.env.local`:

```env
# Magisterium AI
MAGISTERIUM_API_KEY=sk_your_api_key_here
```

### 3. Reštart Servera

```bash
cd backend
npm run dev
```

### 4. Testovanie

1. Otvorte `/admin/news/new`
2. Kliknite na "✨ Otvoriť AI Asistent"
3. Vyberte "⛪ Magisterium AI"
4. Zadajte tému, napríklad: "Výklad podobenstva o stratenom synovi"
5. Typ: "📖 Biblický výklad"
6. Kliknite "🚀 Vygenerovať článok s AI"

## ✅ Overenie Integrácie

Ak všetko funguje správne, uvidíte:
```
✨ Článok vygenerovaný pomocou ⛪ Magisterium AI! (1350 tokenov)
```

## ❌ Časté Problémy

**Problem:** "Magisterium API nie je nakonfigurované"
```bash
# Skontrolujte .env.local
cat .env.local | grep MAGISTERIUM

# Reštartujte server
npm run dev
```

**Problem:** "API error: 401"
- Skontrolujte správnosť API kľúča
- Overte, že kľúč nie je expirovaný

## 📊 Cenník

Magisterium AI je veľmi cenovo dostupný:
- ~$0.40 / 1M tokenov (input + output)
- Článok 1000 slov: ~$0.001
- 1000 článkov: ~$1

## 🎯 Odporúčané Použitie

**Magisterium AI:**
- ✝️ Biblické výklady
- 📖 Teologické články
- 🙏 Lectio Divina
- ⛪ Sviatosti

**OpenAI GPT-4:**
- 🔔 Novinky
- 💡 Moderné témy
- 🌍 Všeobecné články

## 📚 Ďalšie Zdroje

- [Plná dokumentácia](./MAGISTERIUM_AI_INTEGRATION.md)
- [Magisterium AI Docs](https://www.magisterium.com/developers/docs)
- [Chat Completions API](https://www.magisterium.com/developers/docs/chat-completions)

---

**Status:** ✅ Ready for Production  
**Tested:** 25.10.2025  
**Version:** 1.0.0
