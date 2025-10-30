# AI Lectio Divina Generator - Dokumentácia

## Prehľad

Nová funkcia umožňuje automatické generovanie Lectio Divina textov pomocou OpenAI GPT-4o/GPT-4o-mini.

## Funkcie

### 1. **AI Generátor Lectio Divina**
- Automatické spracovanie biblického textu
- Generovanie všetkých 4 sekcií (Lectio, Meditatio, Oratio, Contemplatio)
- Výber medzi GPT-4o (najlepšia kvalita) a GPT-4o-mini (rýchlejší & lacnejší)
- Možnosť pridania zdrojového materiálu (komentáre, úvahy)

### 2. **Gramatická kontrola s AI**
- Každé textové pole má tlačidlo "Skontrolovať gramatiku"
- Oprava preklepov, gramatiky a interpunkcie
- Zachovanie originálneho významu a tónu
- Používa GPT-4o-mini (lacnejší model)

### 3. **Inteligentné textové polia (AITextField)**
- Veľké textové polia s integrovanou AI kontrolou
- Real-time feedback
- Rôzne typy polí (spiritual, prayer, reference, bible)

## Použitie

### Generovanie Lectio Divina

1. Vyplňte biblický text v poli `biblia_1`
2. Zadajte súradnice Písma (napr. "Lk 17, 1-6")
3. Voliteľne pridajte zdrojový materiál (komentáre, úvahy)
4. Vyberte model AI (GPT-4o alebo GPT-4o-mini)
5. Kliknite na "Generovať Lectio Divina"
6. AI automaticky vyplní všetky 4 sekcie

### Kontrola gramatiky

1. Napíšte alebo upravte text v ľubovoľnom poli
2. Kliknite na tlačidlo "✨ Skontrolovať gramatiku"
3. AI opraví text a ukáže zmeny
4. Text sa automaticky aktualizuje

## API Endpointy

### POST `/api/generate-lectio-divina`

**Request:**
```json
{
  "source_material": "Dodatočný text...",
  "perikopa_ref": "Lk 17, 1-6",
  "perikopa_text": "Potom povedal svojim učeníkom...",
  "model": "gpt-4o" // alebo "gpt-4o-mini"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "lectio": "Text lectio...",
    "meditatio": "Text meditatio...",
    "oratio": "Text oratio...",
    "contemplatio": "Text contemplatio..."
  },
  "usage": {
    "model": "gpt-4o",
    "tokens": 1234
  }
}
```

### POST `/api/check-grammar`

**Request:**
```json
{
  "text": "Text na kontrolu...",
  "fieldType": "spiritual" // spiritual | prayer | reference | bible
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "corrected_text": "Opravený text...",
    "changes_made": "Popis zmien alebo 'Žiadne zmeny potrebné'"
  },
  "usage": {
    "tokens": 234
  }
}
```

## Databázová schéma

### Nový stĺpec v `lectio_sources`

```sql
ALTER TABLE lectio_sources 
ADD COLUMN source_material TEXT;
```

**Popis:**
- `source_material` - Zdrojový materiál, komentáre a dodatočný text použitý pri AI generovaní

## Komponenty

### `LectioAIGenerator`
**Props:**
- `bibliaText: string` - Biblický text z biblia_1
- `suradnicePismo: string` - Súradnice Písma
- `onGenerated: (data) => void` - Callback po vygenerovaní
- `disabled?: boolean` - Zakázať generovanie

### `AITextField`
**Props:**
- `label: string` - Popis poľa
- `name: string` - Názov poľa
- `value: string` - Hodnota
- `onChange: (value: string) => void` - Handler zmeny
- `placeholder?: string` - Placeholder
- `rows?: number` - Počet riadkov (default: 10)
- `height?: string` - Výška (default: '15rem')
- `fieldType?: string` - Typ poľa (default: 'spiritual')
- `disabled?: boolean` - Zakázať editáciu
- `showGrammarCheck?: boolean` - Zobraziť tlačidlo kontroly (default: true)

## Ceny OpenAI (približne)

### GPT-4o
- Input: $2.50 / 1M tokenov
- Output: $10.00 / 1M tokenov
- **Približná cena na 1 generovanie:** $0.02 - $0.05

### GPT-4o-mini
- Input: $0.15 / 1M tokenov
- Output: $0.60 / 1M tokenov
- **Približná cena na 1 generovanie:** $0.001 - $0.003

### Gramatická kontrola (GPT-4o-mini)
- **Približná cena:** $0.0005 - $0.001 na kontrolu

## Systémový Prompt

AI používa detailný systémový prompt s pokynmi pre:
- Štýl a tón (jasný, tichý, povzbudzujúci)
- Dĺžku textov (Lectio: 60-90 slov, Meditatio: 90-140 slov, atď.)
- Teologické zásady (katolícka citlivosť, vyhnutie sa polemikám)
- Štruktúru výstupu (JSON formát)

Viac detailov v súbore: `/backend/src/app/api/generate-lectio-divina/route.ts`

## Bezpečnosť

- API kľúč uložený v `.env.local`
- Rate limiting na API endpointoch (odporúčané)
- Validácia vstupov
- Error handling s user-friendly správami

## Možné vylepšenia

1. **Cache výsledkov** - uložiť vygenerované texty pre opakované použitie
2. **History/versioning** - uchovávať predchádzajúce verzie textov
3. **Batch processing** - generovať viacero lectio naraz
4. **Custom prompts** - možnosť upraviť systémový prompt v UI
5. **A/B testing** - porovnávať rôzne modely a ich výsledky

## Troubleshooting

### Chyba: "Prázdna odpoveď z OpenAI"
- Skontrolujte API kľúč v `.env.local`
- Skontrolujte kredit na OpenAI účte

### Chyba: "Neplatná štruktúra odpovede z AI"
- Model nevrátil správny JSON formát
- Skúste znovu alebo prepnite na iný model

### Pomalé generovanie
- GPT-4o je pomalší než GPT-4o-mini
- Prepnite na GPT-4o-mini pre rýchlejšie výsledky

## Kontakt

Pre otázky alebo problémy kontaktujte: info@lectiodivina.sk
