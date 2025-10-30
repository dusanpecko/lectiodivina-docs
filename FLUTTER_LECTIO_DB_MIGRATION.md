# Flutter Lectio Divina - Database Migration

## 📅 Dátum migrácie: 26. október 2025

## 🎯 Prehľad zmien

Flutter aplikácia bola aktualizovaná, aby používala novú 3-tabuľkovú štruktúru databázy namiesto pôvodnej jednej tabuľky `lectio`.

---

## 🗄️ Databázová štruktúra

### **PRED (stará štruktúra)**

```dart
lectio {
  datum: '2025-01-15',
  lang: 'sk',
  hlava: 'Títo',
  lectio_text: '...',
  biblia_1: '...',
  lectio_audio: 'url'
}
```

### **PO (nová štruktúra)**

```dart
// 1️⃣ Liturgický kalendár
liturgical_calendar {
  datum: '2025-01-15',
  locale_code: 'sk',
  celebration_title: '2. nedeľa v Cezročnom období',
  lectio_hlava: '2. NEDEĽA V CEZROČNOM OBDOBÍ',
  celebration_rank_num: 4,
  liturgical_year_id: 2
}

// 2️⃣ Liturgický rok
liturgical_years {
  id: 2,
  lectionary_cycle: 'C', // A, B, alebo C
  start_date: '2024-12-01',
  end_date: '2025-11-30'
}

// 3️⃣ Lectio zdroje
lectio_sources {
  hlava: '2. NEDEĽA V CEZROČNOM OBDOBÍ',
  lang: 'sk',
  rok: 'C', // 'N' pre všedné dni, 'A'/'B'/'C' pre nedele/sviatky
  biblia_1: '...',
  lectio_text: '...',
  lectio_audio: 'url'
}
```

---

## 🔄 Logika načítavania dát

### **Krok 1: Načítaj liturgický kalendár**
```dart
final calendarResponse = await supabase
    .from('liturgical_calendar')
    .select('*, liturgical_years(*)')
    .eq('datum', today)
    .eq('locale_code', lang)
    .maybeSingle();
```

### **Krok 2: Detekcia typu dňa**
```dart
final isWeekday = RegExp(r'(Pondelok|Utorok|Streda|Štvrtok|Piatok|Sobota).+týždňa v Cezročnom období')
    .hasMatch(celebrationTitle);

final isSpecialDay = !isWeekday && (
  celebrationTitle.toLowerCase().contains('nedeľa') ||
  celebrationTitle.toLowerCase().contains('sunday') ||
  (celebrationRankNum != null && celebrationRankNum > 1)
);

final rokToSearch = isSpecialDay ? lectionaryCycle : 'N';
```

**Pravidlá:**
- ✅ **Všedné dni** (pondelok-sobota v cezročnom období) → používame rok `'N'`
- ✅ **Nedele a sviatky** → používame liturgický cyklus `'A'`, `'B'` alebo `'C'`

### **Krok 3: Načítaj lectio source s fallback logikou**

```dart
// 1. Skús primárny jazyk + správny rok
lectioSource = await supabase
    .from('lectio_sources')
    .select()
    .eq('hlava', lectioHlava)
    .eq('lang', lang)
    .eq('rok', rokToSearch)
    .maybeSingle();

// 2. Pre sviatky: skús rok 'N' ak A/B/C nenájdené
if (lectioSource == null && isSpecialDay && rokToSearch != 'N') {
  lectioSource = await supabase
      .from('lectio_sources')
      .eq('hlava', lectioHlava)
      .eq('lang', lang)
      .eq('rok', 'N')
      .maybeSingle();
}

// 3. Fallback na slovenčinu
if (lectioSource == null && lang != 'sk') {
  lectioSource = await supabase
      .from('lectio_sources')
      .eq('hlava', lectioHlava)
      .eq('lang', 'sk')
      .eq('rok', rokToSearch)
      .maybeSingle();
  
  // 4. Pre sviatky v slovenčine: aj tu skús 'N'
  if (lectioSource == null && isSpecialDay && rokToSearch != 'N') {
    lectioSource = await supabase
        .from('lectio_sources')
        .eq('hlava', lectioHlava)
        .eq('lang', 'sk')
        .eq('rok', 'N')
        .maybeSingle();
  }
}
```

---

## 📝 Fallback reťazec (Priority Chain)

1. **Primárne**: `(hlava, lang, rok)` - špecifický jazyk + správny cyklus
2. **Fallback 1**: `(hlava, lang, 'N')` - špecifický jazyk + všeobecný rok (len pre sviatky)
3. **Fallback 2**: `(hlava, 'sk', rok)` - slovenčina + správny cyklus
4. **Fallback 3**: `(hlava, 'sk', 'N')` - slovenčina + všeobecný rok (len pre sviatky)

---

## 🔍 Debug logy

Implementované debug logy pre sledovanie procesu:

```dart
debugPrint('🔍 Načítavam lectio pre dátum: $today, jazyk: $lang');
debugPrint('🔍 Hľadám rok: "$rokToSearch" (všedný deň: ÁNO/NIE, špeciálny deň: true/false)');
debugPrint('✅ Lectio source nájdený: ${lectioSource['hlava']}');
debugPrint('❌ Lectio source neexistuje pre žiadny jazyk');
debugPrint('🔄 Sviatok nenájdený s rokom A/B/C, skúšam rok N...');
debugPrint('🔄 Skúšam načítať lectio source pre slovenčinu...');
```

---

## ✅ Zmeny v súboroch

### **Upravený súbor:**
- `/mobile/lib/screens/lectio_screen.dart`

### **Metóda:**
- `fetchLectioData()` - kompletne prepísaná

### **Funkčnosť zachovaná:**
- ✅ Navigácia medzi dátumami (predchádzajúci/ďalší deň)
- ✅ Výber dátumu kalendárom
- ✅ Zobrazenie biblických textov (biblia_1, biblia_2, biblia_3)
- ✅ Zobrazenie Lectio Divina sekcií (Lectio, Meditatio, Oratio, Contemplatio, Actio)
- ✅ Audio prehrávanie
- ✅ Pridanie poznámky s biblickým odkazom
- ✅ Multi-jazyková podpora (sk, cz, en, es)
- ✅ Fallback na slovenčinu

---

## 🧪 Testovanie

### **Test scenáre:**

1. **Všedný deň (pondelok-sobota)**
   - ✅ Načíta rok 'N'
   - ✅ Fallback na slovenčinu funguje

2. **Nedeľa**
   - ✅ Načíta správny liturgický cyklus (A/B/C)
   - ✅ Fallback na 'N' ak cyklus nenájdený
   - ✅ Fallback na slovenčinu funguje

3. **Sviatok (celebration_rank_num > 1)**
   - ✅ Načíta správny liturgický cyklus (A/B/C)
   - ✅ Fallback na 'N' ak cyklus nenájdený
   - ✅ Fallback na slovenčinu funguje

4. **Jazykové mutácie**
   - ✅ Slovenčina (sk)
   - ✅ Čeština (cz) → fallback na sk
   - ✅ Angličtina (en) → fallback na sk
   - ✅ Španielčina (es) → fallback na sk

---

## 🚀 Výhody novej štruktúry

1. **Flexibilita**: Podporuje 3-ročný liturgický cyklus (A, B, C)
2. **Škálovateľnosť**: Jednoduché pridávanie nových jazykov
3. **Presnosť**: Správne rozlišovanie medzi všednými dňami a sviatkami
4. **Údržba**: Centralizovaný liturgický kalendár
5. **Multiverzia**: Podpora viacerých biblických prekladov (biblia_1, biblia_2, biblia_3)

---

## 📊 Kompatibilita

- ✅ **Flutter SDK**: 3.x+
- ✅ **Supabase Flutter**: Najnovšia verzia
- ✅ **Dart**: 3.x+
- ✅ **Backwards compatible**: Staré dáta fungujú cez fallback mechanizmus

---

## 🔗 Súvisiace dokumenty

- Backend migrácia: `/backend/src/app/lectio/page.tsx`
- Database schema: Supabase projekt `lectiodivina`
- API dokumentácia: `/backend/src/app/api/lectio/`

---

## 👨‍💻 Autor

**Dušan Pecko**  
Dátum: 26. október 2025

---

## 📌 Poznámky

- **Žiadne breaking changes** pre koncového používateľa
- **Bezproblémová migrácia** - fallback logika zabezpečuje kontinuitu
- **Vylepšený logging** pre jednoduchšie debuggovanie
- **Performance**: Minimálny dopad na výkon (1-2 dodatočné query pri fallbacku)

---

## ⚠️ Dôležité upozornenia

1. **Database Access**: Aplikácia vyžaduje prístup k týmto tabuľkám:
   - `liturgical_calendar`
   - `liturgical_years`
   - `lectio_sources`

2. **RLS (Row Level Security)**: Uistite sa, že Supabase policies povoľujú čítanie týchto tabuliek

3. **Data Migration**: Stará tabuľka `lectio` môže zostať v databáze pre backward compatibility, ale nie je už používaná

---

**Status**: ✅ MIGRÁCIA DOKONČENÁ
