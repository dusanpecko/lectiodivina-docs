# Launch Checklist - Návod

## 📋 Čo je Launch Checklist?

Launch Checklist je interaktívna stránka v admin paneli, ktorá ti umožňuje sledovať pokrok projektu a odškrtávať splnené úlohy pri príprave na spustenie aplikácie.

## 🚀 Ako spustiť

### 1. Vytvorenie databázovej tabuľky

Najprv musíš spustiť SQL skript v Supabase, ktorý vytvorí tabuľku a naplní ju úlohami:

1. Otvor **Supabase Dashboard** → Tvoj projekt
2. Klikni na **SQL Editor** v ľavom menu
3. Otvor súbor `/sql/create_launch_checklist_table.sql`
4. Skopíruj celý obsah súboru
5. Vlož do SQL editora
6. Klikni **Run** (alebo Ctrl/Cmd + Enter)

### 2. Prístup k stránke

Po vytvorení tabuľky môžeš pristúpiť k checklist stránke:

1. Prihlás sa do admin panelu: `https://tvoja-domena.com/admin`
2. Na hlavnej stránke uvidíš novú kartu **"Launch Checklist"** s ikonou ✅
3. Klikni na kartu alebo prejdi priamo na: `https://tvoja-domena.com/admin/launch-checklist`

## ✨ Funkce

### Drag & Drop - Presúvanie
- **Presunúť úlohu (zmena poradia)**:
  - Chyť úlohu za grip ikonu (:::)
  - Presuň na inú úlohu v tej istej kategórii
  - Pusť - úlohy sa automaticky vymenia
  
- **Presunúť úlohu do inej kategórie**:
  - Chyť úlohu za grip ikonu
  - Presuň na hlavičku inej kategórie
  - Pusť - úloha sa presunie do novej kategórie
  
- **Presunúť celú kategóriu**:
  - Chyť kategóriu za grip ikonu v hlavičke
  - Presuň na inú kategóriu
  - Potvrď v dialógu - všetky úlohy sa presunú

- **Vizuálny feedback**:
  - Modrý ring pri drag over
  - Grip ikona (:::) sa zvýrazní pri hover
  - Cursor sa zmení na grab/grabbing

### Pridávanie úloh
- Klikni na tlačidlo **"Pridať úlohu"** v pravom hornom rohu
- Alebo klikni na **"Pridať"** pri konkrétnej kategórii (automaticky vyplní kategóriu)
- Vyplň formulár:
  - **Kategória**: Názov kategórie (napr. BRANDING)
  - **Úloha**: Popis úlohy
  - **Týždeň**: Číslo týždňa (1-20)
  - **Poradie**: Poradové číslo úlohy
- Klikni **"Pridať"**

### Úprava úloh
- Nájdi úlohu, ktorú chceš upraviť
- Pri hover (prejdení myšou) sa zobrazia akčné tlačidlá
- Klikni na **modrú ikonu ceruzky** (Edit)
- Uprav údaje v modálnom okne
- Klikni **"Uložiť"**

### Kopírovanie úloh
- Pri hover na úlohu sa zobrazia akčné tlačidlá
- Klikni na **zelenú ikonu kópie** (Copy)
- Vytvorí sa kópia úlohy s textom "(kópia)" na konci
- Kópia bude pridaná na koniec zoznamu

### Mazanie úloh
- Pri hover na úlohu sa zobrazia akčné tlačidlá
- Klikni na **červenú ikonu koša** (Delete)
- Potvrď vymazanie v dialógu
- Úloha bude trvalo odstránená

### Odškrtávanie úloh
- Klikni na kruhové tlačidlo vedľa úlohy na odškrtnutie/zrušenie
- Pri odškrtnutí sa automaticky uloží:
  - Čas dokončenia
  - Používateľ, ktorý úlohu splnil
  
### Kategórie
Úlohy sú rozdelené do 10 kategórií:
- 🎨 **BRANDING** (Týždeň 1)
- 🔧 **TECHNICKÁ INFRAŠTRUKTÚRA** (Týždeň 2)
- 📝 **OBSAH** (Týždeň 3-5)
- 📱 **APP DEVELOPMENT** (Týždeň 3-5)
- 🏪 **APP STORE** (Týždeň 3-4)
- 💰 **FUNDRAISING** (Týždeň 3-8)
- 📢 **MARKETING** (Týždeň 6-9)
- 🧪 **BETA TESTING** (Týždeň 8-9)
- 🎉 **SPUSTENIE** (Týždeň 10)
- 📊 **POST-LAUNCH** (Týždeň 11+)

### Poznámky
- Ku každej úlohe môžeš pridať poznámku
- Klikni na "Pridať poznámku" pod úlohou
- Napíš text a klikni "Uložiť"
- Poznámky sa zobrazia žltým pozadím

### Progress Bar
- Hlavná stránka zobrazuje celkový pokrok (%)
- Každá kategória má svoj vlastný progress bar
- Štatistiky kategórií sa zobrazia v hlavičke

### Akčné tlačidlá na úlohách
Pri prejdení myšou (hover) na úlohu sa zobrazia 3 akčné tlačidlá:
- 🔵 **Edit** (modrá ceruzka) - Upraviť úlohu
- 🟢 **Copy** (zelená kópia) - Kopírovať úlohu  
- 🔴 **Delete** (červený kôš) - Vymazať úlohu

Tlačidlá sú skryté a zobrazia sa len pri hover, aby nezavadzali.

### Rozbaľovanie/Zabaľovanie kategórií
- Klikni na názov kategórie pre rozbalenie/zabalenie
- Všetky kategórie sú defaultne rozbalené

## 🗄️ Databázová štruktúra

Tabuľka `launch_checklist`:

```sql
- id: UUID (primary key)
- category: text (názov kategórie)
- task: text (popis úlohy)
- is_completed: boolean (je splnená?)
- completed_at: timestamptz (kedy bola splnená)
- completed_by: uuid (kto ju splnil)
- week_number: integer (týždeň realizácie)
- order_index: integer (poradie úlohy)
- notes: text (poznámky)
- created_at: timestamptz (vytvorené)
- updated_at: timestamptz (aktualizované)
```

## 🔒 Oprávnenia (RLS)

- **Admin**: Plný prístup (čítanie, úprava, mazanie)
- **Všetci**: Len čítanie (môžu vidieť pokrok)

## 🎯 Celkový počet úloh

Checklist obsahuje **91 úloh** rozdelených do 10 kategórií:

1. BRANDING: 8 úloh
2. TECHNICKÁ INFRAŠTRUKTÚRA: 8 úloh
3. OBSAH: 8 úloh
4. APP DEVELOPMENT: 8 úloh
5. APP STORE: 9 úloh
6. FUNDRAISING: 10 úloh
7. MARKETING: 12 úloh
8. BETA TESTING: 10 úloh
9. SPUSTENIE: 10 úloh
10. POST-LAUNCH: 8 úloh

## 📝 Upravenie úloh

Teraz máš plnú CRUD (Create, Read, Update, Delete) funkcionalitu priamo v UI!

### Cez UI (odporúčané):
1. **Pridať**: Klikni na "Pridať úlohu" alebo "Pridať" pri kategórii
2. **Upraviť**: Hover na úlohu → klikni modrú ceruzku
3. **Kopírovať**: Hover na úlohu → klikni zelenú ikonu kópie
4. **Vymazať**: Hover na úlohu → klikni červený kôš

### Cez Supabase Dashboard (alternatíva):
1. Otvor **Supabase Dashboard** → **Table Editor**
2. Vyber tabuľku `launch_checklist`
3. Klikni na **Insert row** pre pridanie novej úlohy
4. Vyplň polia:
   - `category`: Názov kategórie (napr. "BRANDING")
   - `task`: Popis úlohy
   - `week_number`: Číslo týždňa (1-11)
   - `order_index`: Poradie (napr. 92, 93...)
   - `is_completed`: false
   - Ostatné polia nechaj NULL

### Cez SQL:
```sql
INSERT INTO launch_checklist (category, task, week_number, order_index) 
VALUES ('NOVÁ KATEGÓRIA', 'Nová úloha', 1, 92);
```

## 🐛 Troubleshooting

### Stránka zobrazuje prázdny zoznam
- Uisti sa, že si spustil SQL skript
- Skontroluj, či tabuľka `launch_checklist` existuje
- Skontroluj RLS policies (musia byť povolené)

### Nemôžem odškrtávať úlohy
- Skontroluj, či si prihlásený ako admin
- Skontroluj browser console pre chyby
- Overiť RLS policy pre UPDATE operáciu

### Kategória sa nezobrazuje
- Skontroluj, či má kategória aspoň jednu úlohu
- Skontroluj `order_index` - musia byť unikátne

## 🎨 Úprava dizajnu

Súbor na úpravu:
`/src/app/admin/launch-checklist/page.tsx`

Farby:
- Gradient: `from-indigo-500 to-purple-600`
- Kategórie: `border-indigo-300`
- Dokončené úlohy: `bg-green-50 border-green-200`

## 📱 Responzívny dizajn

Stránka je plne responzívna:
- Desktop: 6-column layout pre štatistiky
- Tablet: 4-column layout
- Mobile: 2-column layout, full width kategórie

---

Vytvoril: Lectio Divina Team  
Verzia: 1.0  
Dátum: 18.10.2025
