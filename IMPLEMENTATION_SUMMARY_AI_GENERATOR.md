# ✨ AI Lectio Divina Generator - Implementácia

## 📋 Zhrnutie

Implementovali sme komplexný AI systém na automatické generovanie a zlepšovanie Lectio Divina textov s využitím OpenAI GPT-4o a GPT-4o-mini.

---

## 🎯 Implementované funkcie

### 1. **AI Generátor Lectio Divina** 🤖
- Automatické generovanie všetkých 4 sekcií (Lectio, Meditatio, Oratio, Contemplatio)
- Výber medzi 2 modelmi:
  - **GPT-4o** - najlepšia kvalita (⭐⭐⭐⭐⭐)
  - **GPT-4o-mini** - rýchlejší & 10x lacnejší (⭐⭐⭐⭐)
- Využitie biblického textu z `biblia_1` + súradníc z `suradnice_pismo`
- Možnosť pridania zdrojového materiálu (komentáre, úvahy)

### 2. **Gramatická kontrola s AI** ✨
- Tlačidlo "Skontrolovať gramatiku" pri každom textovom poli
- Automatická oprava:
  - Gramatických chýb
  - Preklepov
  - Interpunkcie
  - Štylizácie
- Zachovanie originálneho významu a tónu
- Používa GPT-4o-mini (lacnejší)

### 3. **Inteligentné textové polia (AITextField)** 📝
- Nahradili sme staré `<textarea>` s novým `AITextField` komponentom
- Integrovaná gramatická kontrola
- Real-time feedback
- Rôzne typy polí (spiritual, prayer, reference, bible)

---

## 📁 Vytvorené súbory

### API Endpointy
```
/backend/src/app/api/
├── generate-lectio-divina/
│   └── route.ts          # POST endpoint pre generovanie
└── check-grammar/
    └── route.ts          # POST endpoint pre kontrolu gramatiky
```

### Komponenty
```
/backend/src/app/components/
├── LectioAIGenerator.tsx  # Hlavný AI generátor komponent
└── AITextField.tsx        # Textové pole s AI kontrolou
```

### Databáza
```
/backend/sql/
├── add_source_material_column.sql           # Automatická migrácia
└── MANUAL_RUN_add_source_material.sql      # Manuálne spustenie
```

### Dokumentácia
```
/docs/
└── AI_LECTIO_DIVINA_GENERATOR.md           # Kompletná dokumentácia
```

### Testy
```
/backend/
└── test-ai-generator.sh                     # Test script
```

---

## 🔧 Úpravy existujúcich súborov

### `/backend/src/app/admin/lectio-sources/[id]/page.tsx`

**Pridané:**
1. Import nových komponentov (`LectioAIGenerator`, `AITextField`)
2. Nový stĺpec v interface `LectioSource`: `source_material?: string`
3. Handler `handleLectioAIGenerated` pre spracovanie AI výsledkov
4. Nová sekcia "🤖 AI Generátor Lectio Divina" pred Biblické texty
5. Nahradené všetky 4 textové polia s `AITextField`:
   - `lectio_text`
   - `meditatio_text`
   - `oratio_text`
   - `contemplatio_text`

---

## 🗄️ Databázová schéma

### Nový stĺpec v `lectio_sources`

```sql
ALTER TABLE lectio_sources 
ADD COLUMN IF NOT EXISTS source_material TEXT;
```

**Účel:** Uloženie zdrojového materiálu, komentárov a dodatočného textu použitého pri AI generovaní.

**Poznámka:** SQL treba spustiť manuálne v Supabase SQL Editor (súbor je pripravený).

---

## 💰 Ceny OpenAI

### GPT-4o
- **Input:** $2.50 / 1M tokenov
- **Output:** $10.00 / 1M tokenov
- **Cena na 1 generovanie:** ~$0.02 - $0.05

### GPT-4o-mini
- **Input:** $0.15 / 1M tokenov
- **Output:** $0.60 / 1M tokenov
- **Cena na 1 generovanie:** ~$0.001 - $0.003

### Gramatická kontrola
- **Cena:** ~$0.0005 - $0.001 na kontrolu

**Odhadované náklady pri 100 generovaniach/mesiac:**
- GPT-4o: ~$2-5/mesiac
- GPT-4o-mini: ~$0.10-0.30/mesiac

---

## 🚀 Ako používať

### 1. Spustite databázovú migráciu

V Supabase SQL Editor spustite:
```sql
-- Obsah súboru: sql/MANUAL_RUN_add_source_material.sql
ALTER TABLE lectio_sources 
ADD COLUMN IF NOT EXISTS source_material TEXT;
```

### 2. Generovanie Lectio Divina

1. Otvorte editáciu lectio source (`/admin/lectio-sources/[id]`)
2. Vyplňte biblický text v `biblia_1`
3. Zadajte súradnice (napr. "Lk 17, 1-6")
4. Vyberte model (GPT-4o alebo GPT-4o-mini)
5. Voliteľne pridajte zdrojový materiál
6. Kliknite "Generovať Lectio Divina"
7. AI automaticky vyplní všetky 4 polia

### 3. Kontrola gramatiky

1. Napíšte text do ľubovoľného poľa
2. Kliknite "✨ Skontrolovať gramatiku"
3. AI opraví text a ukáže zmeny
4. Text sa automaticky aktualizuje

---

## ✅ Checklist implementácie

- [x] API endpoint `/api/generate-lectio-divina`
- [x] API endpoint `/api/check-grammar`
- [x] Komponent `LectioAIGenerator`
- [x] Komponent `AITextField`
- [x] Aktualizácia `page.tsx`
- [x] Interface `LectioSource` s `source_material`
- [x] SQL migrácia
- [x] Dokumentácia
- [x] Test script
- [ ] **Spustenie SQL migrácie v Supabase** ⚠️ (manuálne)

---

## 🔍 Testovanie

### Lokálne testovanie API

```bash
cd /Users/dusanpecko/lectiodivina/backend
./test-ai-generator.sh
```

### Manuálne testovanie cez curl

```bash
curl -X POST http://localhost:3000/api/generate-lectio-divina \
  -H "Content-Type: application/json" \
  -d '{
    "perikopa_ref": "Lk 17, 1-6",
    "perikopa_text": "Potom povedal svojim učeníkom...",
    "model": "gpt-4o-mini"
  }'
```

---

## 🐛 Možné problémy a riešenia

### Chyba: "Prázdna odpoveď z OpenAI"
**Riešenie:** Skontrolujte API kľúč v `.env.local` a kredit na OpenAI účte

### Chyba: Stĺpec `source_material` neexistuje
**Riešenie:** Spustite SQL migráciu v Supabase SQL Editor

### Pomalé generovanie
**Riešenie:** Prepnite na GPT-4o-mini

---

## 🎨 UI/UX vylepšenia

- **Gradient design** pre AI sekciu (purple-indigo)
- **Real-time feedback** pri gramatickej kontrole
- **Loading states** s animáciami
- **Success/error messages** s automatickým zmiznutím
- **Model selector** s vizuálnou indikáciou
- **Tooltips a hints** pre lepšiu orientáciu

---

## 📚 Ďalšie zdroje

- **Kompletná dokumentácia:** `/docs/AI_LECTIO_DIVINA_GENERATOR.md`
- **OpenAI API dokumentácia:** https://platform.openai.com/docs
- **Pricing:** https://openai.com/api/pricing/

---

## 👨‍💻 Autor

Implementácia: GitHub Copilot + Dušan Pecko  
Dátum: 22. október 2025  
Projekt: Lectio Divina

---

## 🎯 Next Steps

1. **Spustiť SQL migráciu** v Supabase
2. **Otestovať generovanie** s reálnym textom
3. **Vyskúšať oba modely** (GPT-4o vs GPT-4o-mini)
4. **Zozbierať feedback** od používateľov
5. **Optimalizovať prompty** na základe výsledkov

---

**Poznámka:** Všetko je pripravené na použitie! Stačí spustiť SQL migráciu a môžete začať generovať. 🚀
