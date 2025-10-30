# 🔊 ElevenLabs Text-to-Speech pre News články

## Prehľad

Systém automatického generovania audio verzií news článkov pomocou **ElevenLabs Text-to-Speech API**. Používa rovnaký systém ako lectio-sources s profesionálnymi hlasmi pre 7 jazykov.

## Implementácia

### 1. API Endpoint

**File:** `/backend/src/app/api/ai-generate-audio/route.ts`

**Funkcie:**

- ✅ Konverzia HTML → plain text (odstránenie tagov, entities)
- ✅ Rozdelenie dlhých textov na chunks (max 1500 znakov)
- ✅ Generovanie audio cez ElevenLabs v3 model
- ✅ Upload do Supabase Storage (`audio-files/news/`)
- ✅ Podpora 7 jazykov (SK, CZ, EN, ES, IT, PT, DE)

**Request:**

```json
{
  "newsId": "123",
  "title": "Nadpis článku",
  "content": "<p>HTML obsah článku...</p>",
  "language": "sk"
}
```

**Response:**

```json
{
  "audioUrl": "https://...supabase.co/storage/.../news_123_sk_1234567890.mp3",
  "filename": "news_123_sk_1234567890.mp3",
  "language": "sk",
  "voiceUsed": "scOwDtmlUjD3prqpp97I",
  "model": "eleven_v3",
  "fileSize": 245678,
  "textLength": 1523,
  "chunksProcessed": 2,
  "duration": 101.5
}
```

### 2. Voice Mapping

| Jazyk | Voice ID               | Model     | Hlas            |
| ----- | ---------------------- | --------- | --------------- |
| 🇸🇰 SK | `scOwDtmlUjD3prqpp97I` | eleven_v3 | Sam (male)      |
| 🇨🇿 CZ | `scOwDtmlUjD3prqpp97I` | eleven_v3 | Sam (male)      |
| 🇬🇧 EN | `21m00Tcm4TlvDq8ikWAM` | eleven_v3 | Rachel (female) |
| 🇩🇪 DE | `jsCqWAovK2LkecY7zXl4` | eleven_v3 | Freya (female)  |
| 🇮🇹 IT | `XB0fDUnXU5powFXDhCwa` | eleven_v3 | Chiara (female) |
| 🇪🇸 ES | `6bNjXphfWPUDHuFkgDt3` | eleven_v3 | Efrayn (male)   |
| 🇵🇹 PT | `6bNjXphfWPUDHuFkgDt3` | eleven_v3 | Efrayn (male)   |

**Voice Settings:**

- `stability`: 0.4-0.6 (konzistentnosť hlasu)
- `similarity_boost`: 0.8-0.9 (vernosť hlasu)
- `style`: 0.0-0.2 (expresivita)
- `use_speaker_boost`: true (vylepšenie kvality)

### 3. UI Integrácia

**File:** `/backend/src/app/admin/news/[id]/page.tsx`

**Pridané:**

- `audio_url?: string` do News interface
- `handleGenerateAudio()` handler
- Tlačidlo "🔊 Generovať Audio" (blue-cyan gradient)
- HTML5 audio prehrávač (zobrazí sa po generovaní)
- Import `Volume2` ikony z lucide-react

**UI Umiestnenie:**
Sekcia "Základné informácie", pod obrázkom, oddelené border-top

### 4. Databáza

**SQL Migrácia:** `/backend/sql/add_audio_url_to_news.sql`

```sql
ALTER TABLE news
ADD COLUMN IF NOT EXISTS audio_url TEXT;
```

**Štruktúra news tabuľky:**

```
- id: number
- title: string
- summary: string (HTML)
- content: string (HTML)
- image_url: string
- audio_url: string ← NOVÝ
- published_at: date
- lang: string
```

## Použitie

### Admin Editor

1. **Vyplňte článok** (nadpis + obsah)
2. **Kliknite "🔊 Generovať Audio"**
3. **Počkajte** (~10-30s podľa dĺžky)
4. **Prehrajte** pomocou audio prehrávača
5. **Uložte článok** (audio_url sa uloží do DB)

### Workflow

```
User klikne "Generovať Audio"
    ↓
HTML → Plain Text konverzia
    ↓
Split na chunks (max 1500 chars)
    ↓
Pre každý chunk:
  - ElevenLabs TTS API (v3 model)
  - 500ms delay (rate limiting)
    ↓
Concat všetky audio buffers
    ↓
Upload do Supabase Storage
    ↓
Vráti public URL
    ↓
Audio player sa zobrazí v UI
```

## Supabase Storage

**Bucket:** `audio-files`
**Path:** `news/{filename}.mp3`
**Filename Format:** `news_{newsId}_{language}_{timestamp}.mp3`

**Príklad:**

```
audio-files/news/news_123_sk_1734567890123.mp3
audio-files/news/news_456_en_1734567901234.mp3
```

**Storage nastavenia:**

- Content-Type: `audio/mpeg`
- Cache-Control: `3600` (1 hodina)
- Verejný prístup: áno (public URL)

## Technické Detaily

### HTML → Plain Text Konverzia

```typescript
function htmlToPlainText(html: string): string {
  // Remove HTML tags
  let text = html.replace(/<[^>]*>/g, " ");

  // Decode entities
  text = text
    .replace(/&nbsp;/g, " ")
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<");
  // ...

  // Cleanup
  return text.replace(/\s+/g, " ").trim();
}
```

### Text Chunking

- **Max chunk size:** 1500 characters
- **Split by:** sentences (`.`, `!`, `?`)
- **Reason:** ElevenLabs API limits + avoid timeouts
- **Delay:** 500ms medzi chunks (rate limiting)

### Odhad trvania

```typescript
duration = (textLength / 15) seconds
// Približne 15 znakov za sekundu
```

**Príklad:**

- 1500 znakov ≈ 100 sekúnd ≈ 1:40 min
- 3000 znakov ≈ 200 sekúnd ≈ 3:20 min

## Environment Variables

```env
ELEVENLABS_API_KEY=sk_xxxxxxxxxxxxx
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJxxx...
```

**ELEVENLABS_API_KEY:**

- Získať na https://elevenlabs.io/
- Dashboard → Profile → API Keys
- Free tier: 10,000 chars/month
- Paid: $5/month = 30,000 chars

## Error Handling

| Status | Error              | Riešenie                  |
| ------ | ------------------ | ------------------------- |
| 400    | Missing parameters | Skontrolovať request body |
| 401    | Invalid API key    | Overiť ELEVENLABS_API_KEY |
| 422    | Invalid text       | Text príliš dlhý/krátky   |
| 429    | Rate limit         | Počkať, znížiť frequency  |
| 500    | Generation failed  | Skontrolovať logs         |

## Náklady

**ElevenLabs Pricing:**

- Free: 10,000 chars/month
- Starter: $5/month = 30,000 chars
- Creator: $11/month = 100,000 chars
- Pro: $99/month = 500,000 chars

**Priemerný článok:**

- Krátky (500 slov): ~2,500 chars
- Stredný (1000 slov): ~5,000 chars
- Dlhý (2000 slov): ~10,000 chars

**Odhad spotreby:**

- 10 stredných článkov = 50,000 chars = $11/month
- 20 stredných článkov = 100,000 chars = $11/month

## Výhody

✅ **Accessibility:** Používatelia môžu počúvať články  
✅ **Multitasking:** Počúvanie počas iných aktivít  
✅ **SEO:** Vyššia angažovanosť (dwell time)  
✅ **Reach:** Širšie publikum (zrakovo postihnutí)  
✅ **Professional:** High-quality v3 voices  
✅ **Multi-language:** 7 jazykov automaticky

## Future Improvements

- [ ] Výber hlasu v UI (male/female)
- [ ] Preview audio pred uložením
- [ ] Batch generation (viacero článkov naraz)
- [ ] Audio length estimation pred generovaním
- [ ] Custom voice speed control
- [ ] Pronunciation dictionary (biblické mená)
- [ ] Audio waveform visualization
- [ ] Download audio button
- [ ] Regenerate audio ak sa zmení obsah

## Testing

1. **Vytvoriť nový článok**
2. **Zadať nadpis** (napr. "Test Audio článok")
3. **Vyplniť obsah** (min 100 znakov)
4. **Vybrať jazyk** (napr. SK)
5. **Generovať audio**
6. **Overiť**:
   - ✓ Audio player sa zobrazí
   - ✓ Audio sa prehráva
   - ✓ URL začína na `.../audio-files/news/...`
   - ✓ Po uložení sa audio_url uloží do DB

## Troubleshooting

**Problem:** Audio sa negeneruje  
**Fix:** Skontrolovať ELEVENLABS_API_KEY v .env

**Problem:** "Rate limit exceeded"  
**Fix:** Počkať 60 sekúnd, alebo upgrade plan

**Problem:** Audio je príliš dlhé  
**Fix:** Text je automaticky rozdelený na chunks, malo by fungovať

**Problem:** Audio nie je v správnom jazyku  
**Fix:** Overiť že `news.lang` je správne nastavený

**Problem:** Upload do Supabase zlyhal  
**Fix:** Overiť SUPABASE_SERVICE_ROLE_KEY a bucket permissions

## Poznámky

- Audio sa negeneruje automaticky pri vytvorení článku (manuálne tlačidlo)
- Audio sa ukladá do `audio-files` bucket (rovnaký ako lectio-sources)
- HTML tagy sa automaticky odstránia pred TTS
- Dlhé texty sa rozdeľujú na chunks (seamless pre používateľa)
- Rovnaké voice ID ako v lectio-sources pre konzistenciu

## Related Files

```
backend/
├── src/app/
│   ├── api/
│   │   ├── ai-generate-audio/route.ts  ← NOVÝ TTS endpoint
│   │   └── text-to-speech/route.ts     ← Existujúci (lectio-sources)
│   └── admin/news/[id]/page.tsx        ← UI + handler
├── sql/
│   └── add_audio_url_to_news.sql       ← Migrácia
└── .env
    └── ELEVENLABS_API_KEY              ← Required
```

---

**Autor:** AI Assistant  
**Dátum:** 22.10.2025  
**Verzia:** 1.0  
**Status:** ✅ Production Ready
