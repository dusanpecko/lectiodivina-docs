# 📦 Backup System Documentation

Systém pre zálohovanie databázy Supabase v projekte Lectio Divina.

## 🚀 Dostupné scripty

### 1. Backup tabuľky lectio_sources
```bash
npm run backup:lectio
```
**Čo robí:**
- Zálohuje tabuľku `lectio_sources` (hlavné lectio dáta)
- Vytvorí 3 formáty: JSON, CSV, SQL
- Súbory sú pomenované s timestamp: `lectio_sources_backup_YYYY-MM-DD_HH-MM-SS.*`

### 2. Kompletný backup databázy
```bash
npm run backup:full
```
**Čo robí:**
- Zálohuje všetky dôležité tabuľky
- Vytvorí individuálne JSON súbory pre každú tabuľku
- Vytvorí jeden kompletný JSON súbor so všetkými dátami
- Vytvorí README.md s dokumentáciou

### 3. Test pripojenia
```bash
node test-db-connection.js
```
**Čo robí:**
- Testuje pripojenie na Supabase
- Zobrazí počet záznamov v hlavných tabuľkách

## 📁 Štruktúra backup súborov

```
backup/
├── lectio_sources_backup_2025-10-21_19-41-06.json    # Lectio data (JSON)
├── lectio_sources_backup_2025-10-21_19-41-06.csv     # Lectio data (CSV) 
├── lectio_sources_backup_2025-10-21_19-41-06.sql     # Lectio data (SQL)
├── full_backup_2025-10-21_19-45-30.json              # Kompletný backup
├── locales_2025-10-21_19-45-30.json                  # Jazyky
├── translations_2025-10-21_19-45-30.json             # Preklady Biblie
└── backup_info_2025-10-21_19-45-30.md               # Dokumentácia
```

## 🔧 Konfigurácia

Script používa environment variables z `.env.local`:
- `NEXT_PUBLIC_SUPABASE_URL` - URL Supabase projektu
- `NEXT_PUBLIC_SUPABASE_ANON_KEY` - Anon/Public kľúč

## 📋 Zálohované tabuľky

### Hlavné backup (`npm run backup:lectio`)
- `lectio_sources` - Hlavné lectio divina dáta

### Kompletný backup (`npm run backup:full`)
- `lectio_sources` - Hlavné lectio dáta
- `locales` - Jazyky (sk, en, es, ...)
- `translations` - Preklady Biblie
- `books` - Biblické knihy
- `bible_verses` - Biblické verše
- `users` - Používatelia
- `articles` - Články
- `calendar_events` - Kalendárne udalosti
- `daily_quotes` - Denné citáty
- `programs` - Programy
- `rosary_mysteries` - Ružencové tajomstvá
- `beta_feedback` - Beta feedback
- `error_reports` - Chybové hlásenia

## 💾 Formáty backup súborov

### JSON formát
```json
{
  "metadata": {
    "table": "lectio_sources",
    "backup_date": "2025-10-21T19:41:06.123Z",
    "total_records": 541,
    "supabase_url": "https://xxx.supabase.co",
    "version": "1.0"
  },
  "data": [
    {
      "id": 1,
      "lang": "sk",
      "kniha": "Matúš",
      // ... všetky stĺpce
    }
  ]
}
```

### CSV formát
Štandardný CSV s hlavičkami, použiteľný v Excel, Google Sheets.

### SQL formát
INSERT statements na obnovenie dát:
```sql
-- Backup tabuľky lectio_sources
-- Vytvorený: 2025-10-21T19:41:06.123Z
-- Počet záznamov: 541

INSERT INTO lectio_sources (id, lang, kniha, ...) VALUES
  (1, 'sk', 'Matúš', ...),
  (2, 'sk', 'Lukáš', ...);
```

## 🔄 Obnovenie dát

### Z JSON súboru
```javascript
const fs = require('fs');
const { createClient } = require('@supabase/supabase-js');

const supabase = createClient(url, key);
const backup = JSON.parse(fs.readFileSync('backup/lectio_sources_backup_xxx.json', 'utf8'));

// Vymazanie existujúcich dát (POZOR!)
await supabase.from('lectio_sources').delete().neq('id', 0);

// Vloženie zálohovaných dát
const { data, error } = await supabase
  .from('lectio_sources')
  .insert(backup.data);
```

### Z SQL súboru
1. Otvor SQL súbor
2. Skopíruj INSERT statements
3. Spusti v Supabase SQL editore

## ⚠️ Bezpečnosť

1. **Backup súbory obsahují citlivé dáta** - neuploaduj ich na verejné repozitáre
2. **Environment variables** - uisti sa, že `.env.local` nie je v gite
3. **Permissions** - script používa anon kľúč, takže môže čítať len verejné dáta
4. **Veľkosť súborov** - backup môže byť veľký (MB), kontroluj disk space

## 🛠 Riešenie problémov

### Chyba pripojenia
```
❌ Chyba: Nenašli sa potrebné environment variables
```
**Riešenie:** Skontroluj `.env.local` súbor

### Prázdna tabuľka
```
⚠️ Tabuľka je prázdna. Backup sa ukončuje.
```
**Riešenie:** Tabuľka naozaj neobsahuje dáta, alebo nemáš oprávnenia na čítanie

### Chyba permissions
```
❌ RLS policy violation
```
**Riešenie:** Anon kľúč nemá prístup k tabuľke. Skontroluj RLS policies v Supabase.

## 📈 Automatizácia

### Cron job (Linux/Mac)
```bash
# Každý deň o polnoci
0 0 * * * cd /path/to/project && npm run backup:lectio

# Každý týždeň kompletný backup  
0 2 * * 0 cd /path/to/project && npm run backup:full
```

### GitHub Actions
```yaml
name: Database Backup
on:
  schedule:
    - cron: '0 2 * * *'  # Každý deň o 2:00
  workflow_dispatch:

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
      - run: npm install
      - run: npm run backup:lectio
        env:
          NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          NEXT_PUBLIC_SUPABASE_ANON_KEY: ${{ secrets.SUPABASE_ANON_KEY }}
```

## 📞 Podpora

V prípade problémov:
1. Skontroluj log výstup scriptu
2. Overte pripojenie cez `node test-db-connection.js`
3. Skontroluj Supabase permissions a RLS policies