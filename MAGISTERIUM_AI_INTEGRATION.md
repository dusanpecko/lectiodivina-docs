# Magisterium AI Integration

## Prehľad

Lectio.one teraz podporuje dva AI modely pre generovanie článkov:

1. **OpenAI GPT-4** - Všeobecný AI asistent
2. **⛪ Magisterium AI** - Špecializovaný na katolícku teológiu a učenie

## Čo je Magisterium AI?

Magisterium AI je špecializovaný veľký jazykový model (LLM) trénovaný na:
- Svätom písme (Biblia)
- Cirkevných otcoch a učiteľoch Cirkvi
- Katechizme Katolíckej cirkvi
- Pápežských encyklikách a dokumentoch
- Katolíckej teológii a spiritualite
- Tradícii Cirkvi

### Výhody použitia Magisterium AI:

✅ **Teologická presnosť** - Odpovede sú v súlade s katolíckym učením  
✅ **Biblická exegéza** - Výklady podľa katolíckej tradície  
✅ **Cirkevné dokumenty** - Odkazuje na oficiálne učenie Cirkvi  
✅ **Duchovná hĺbka** - Hlboké pochopenie katolíckej spirituality  
✅ **Lectio Divina** - Špecializácia na modlitbu so Svätým písmom  

## Použitie v Admin Paneli

### 1. Otvorenie AI Asistenta

V editore článkov (`/admin/news/[id]`) kliknite na:
```
✨ Otvoriť AI Asistent pre generovanie článkov
```

### 2. Výber AI Modelu

V sekcii **"🤖 Vyber AI Model"** máte dve možnosti:

#### OpenAI GPT-4
- **Použitie:** Všeobecné články, novinky, moderné témy
- **Výhody:** Široké všeobecné znalosti, moderný jazyk
- **Vhodné pre:** Novinky Lectio.one, moderné aplikácie spirituality

#### ⛪ Magisterium AI
- **Použitie:** Biblické výklady, teológia, cirkevné učenie
- **Výhody:** Teologická presnosť, cirkevná tradícia
- **Vhodné pre:** 
  - 📖 Biblický výklad
  - ⛪ Teologický článok
  - 🙏 Lectio Divina praktiky
  - ✝️ Sviatosti a liturgia
  - 📜 História cirkvi

### 3. Nastavenie Parametrov

**Téma článku:**
```
Napríklad: Lectio Divina ako cesta k hlbšiemu vzťahu s Bohom
```

**Typ článku:**
- 📖 Biblický výklad
- 🙏 Lectio Divina praktiky
- ⛪ Teologický článok
- 💭 Duchovná meditácia
- 📜 História cirkvi
- ✝️ Sviatosti a liturgia
- 🔔 Novinky Lectio.one

**Dĺžka:**
- 🔹 Krátky (500 slov)
- 🔸 Stredný (1000 slov)
- 🔶 Dlhý (2000+ slov)

**Biblické odkazy (voliteľné):**
```
Ján 3:16, Žalm 23, Mt 5:1-12
```

**Jazyk článku:**
- 🇸🇰 Slovenčina
- 🇨🇿 Čeština
- 🇬🇧 English
- 🇪🇸 Español
- 🇮🇹 Italiano
- 🇵🇹 Português
- 🇩🇪 Deutsch

### 4. Generovanie

Kliknite na **"🚀 Vygenerovať článok s AI"**

Systém automaticky:
1. Vygeneruje nadpis
2. Vytvorí HTML obsah s formátovaním
3. Vygeneruje súhrn (150-200 slov)
4. Voliteľne vygeneruje ilustračný obrázok (WebP)

## Technická implementácia

### API Endpoint

Magisterium AI používa OpenAI-kompatibilné API:

```
POST https://www.magisterium.com/api/v1/chat/completions
```

**Model:** `magisterium-1`  
**Dokumentácia:** https://www.magisterium.com/developers/docs/chat-completions

### Parametre požiadavky

### Environment Variable

V `.env.local` pridajte:
```env
MAGISTERIUM_API_KEY=sk_your_actual_api_key_here
```

### Magisterium AI Model

Používame **magisterium-1** - oficiálny model Magisterium AI trénovaný na katolíckych dokumentoch, Písme a učení Cirkvi.

### Konfigurácia

```typescript
{
  model: "magisterium-1",
  temperature: 0.7,
  max_tokens: 4000,
  response_format: { type: "json_object" },
  return_related_questions: false
}
```

### Príklad API volania

```typescript
const response = await fetch("https://www.magisterium.com/api/v1/chat/completions", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${process.env.MAGISTERIUM_API_KEY}`,
  },
  body: JSON.stringify({
    model: "magisterium-1",
    messages: [
      {
        role: "system",
        content: "You are a Catholic theologian and spiritual writer..."
      },
      {
        role: "user",
        content: "Write a biblical commentary on Luke 15:11-32..."
      }
    ],
    temperature: 0.7,
    max_tokens: 4000,
    response_format: { type: "json_object" }
  })
});
```

## Príklady Použitia

### Príklad 1: Biblický výklad

**Vstup:**
- AI Model: ⛪ Magisterium AI
- Téma: Výklad podobenstva o milosrdnom Samaritánovi
- Typ: 📖 Biblický výklad
- Dĺžka: Stredný
- Biblické odkazy: Lk 10:25-37
- Jazyk: Slovenčina

**Výstup:**
Článok s:
- Historickým kontextom podobenstva
- Exegézou textu v katolíckej tradícii
- Odkazmi na cirkevných otcov
- Praktickou aplikáciou pre dnešok

### Príklad 2: Lectio Divina praktika

**Vstup:**
- AI Model: ⛪ Magisterium AI
- Téma: Ako praktizovať Lectio Divina s Jánovým evanjeliom
- Typ: 🙏 Lectio Divina praktiky
- Dĺžka: Dlhý
- Biblické odkazy: Ján 1:1-14
- Jazyk: Slovenčina

**Výstup:**
Článok s:
- Úvodom do Lectio Divina
- Štyrmi klasickými krokmi (Lectio, Meditatio, Oratio, Contemplatio)
- Praktickými radami
- Modlitbami a meditáciami

### Príklad 3: Novinky (OpenAI)

**Vstup:**
- AI Model: OpenAI GPT-4
- Téma: Nové funkcie v mobilnej aplikácii Lectio.one
- Typ: 🔔 Novinky Lectio.one
- Dĺžka: Krátky
- Jazyk: Slovenčina

**Výstup:**
Moderný, aktuálny článok o novinkách.

## Porovnanie Modelov

| Vlastnosť | OpenAI GPT-4 | Magisterium AI |
|-----------|--------------|----------------|
| **Všeobecné znalosti** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Katolícka teológia** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Biblická exegéza** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Cirkevné dokumenty** | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Moderný jazyk** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Viacjazyčnosť** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Lectio Divina** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Rýchlosť** | Rýchle | Stredné |
| **Cena** | Stredná | Nízka |

## Odporúčania

### Použite Magisterium AI pre:
- ✝️ Biblické výklady
- 📖 Teologické články
- 🙏 Lectio Divina návody
- ⛪ Sviatosti a liturgia
- 📜 História cirkvi
- 💭 Duchovné meditácie

### Použite OpenAI GPT-4 pre:
- 🔔 Novinky a oznámenia
- 💡 Moderné aplikácie spirituality
- 🌍 Všeobecné témy
- 🚀 Technické články o aplikácii

## Monitoring a Náklady

### Token Usage

Oba modely vracajú usage informácie:
```typescript
{
  usage: {
    promptTokens: 150,
    completionTokens: 1200,
    totalTokens: 1350
  }
}
```

### Cenník (orientačný)

**Magisterium AI:**
- Input: ~$0.40 / 1M tokenov
- Output: ~$0.40 / 1M tokenov
- Článok (1000 slov): ~$0.001

**OpenAI GPT-4:**
- Input: $5.00 / 1M tokenov
- Output: $15.00 / 1M tokenov
- Článok (1000 slov): ~$0.02

## Riešenie Problémov

### Chyba: "Magisterium API nie je nakonfigurované"

**Príčina:** Chýba environment variable `MAGISTERIUM_API_KEY`

**Riešenie:**
1. Zaregistrujte sa na https://magisterium.ai
2. Získajte API kľúč
3. Pridajte do `.env.local`:
   ```env
   MAGISTERIUM_API_KEY=your_api_key_here
   ```
4. Reštartujte development server

### Chyba: "API error: 401"

**Príčina:** Neplatný API kľúč

**Riešenie:**
1. Overte API kľúč na https://magisterium.ai/dashboard
2. Skontrolujte, či kľúč nie je expirovaný
3. Vygenerujte nový kľúč ak je potrebné

### Článok je príliš krátky

**Riešenie:**
- Zmeňte dĺžku na "Dlhý"
- Pridajte viac biblických odkazov
- Buďte konkrétnejší v téme

### Článok nie je v správnom jazyku

**Riešenie:**
- Skontrolujte nastavenie "Jazyk článku" v sekcii Základné informácie
- Téma môže byť po slovensky, ale článok bude v cieľovom jazyku

## Budúce Vylepšenia

- [ ] Batch generovanie viacerých článkov naraz
- [ ] Fine-tuning Magisterium AI na Lectio Divina metodológiu
- [ ] Integrácia s vlastnou databázou biblických textov
- [ ] A/B testovanie kvality medzi modelmi
- [ ] Automatické vylepšovanie na základe feedback

## Odkazy

- **Magisterium AI:** https://magisterium.ai
- **API Dokumentácia:** https://docs.magisterium.ai
- **Príklady:** https://magisterium.ai/examples
- **Support:** support@magisterium.ai

## Autor

**Lectio.one Team**  
Integrácia: 25. októbra 2025

---

💡 **Tip:** Pre najlepšie výsledky kombinujte oba modely - Magisterium AI pre teologický obsah a OpenAI pre moderné, aktuálne články.
