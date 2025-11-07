# Generovanie liturgického kalendára zo slovenských zdrojov

## Prehľad
Nová funkcionalita **"🇸🇰 Zo SK zdrojov"** v liturgical-calendar umožňuje vytvoriť liturgický kalendár priamo zo slovenských lectio_sources namiesto generovania z českého API a následného prekladu.

## Proces

### Pôvodný workflow:
```
České API → AI preklad → Slovenský kalendár → Mapovanie na lectio_sources
```

### Nový optimalizovaný workflow:
```
České API (štruktúra/dátumy) + Slovenské lectio_sources → Priamo slovenský kalendár
```

## Implementácia

### Kľúčové funkcie:

1. **`handleGenerateFromSlovakSources()`**
   - Hlavná funkcia pre generovanie
   - Načíta slovenské lectio_sources
   - Získa štruktúru roku z českého API (len pre dátumy/liturgické info)
   - Namapuje české názvy na slovenské lectio_sources
   - Vytvorí priamo slovenský kalendár

2. **`findMatchingLectioSource()`**
   - Inteligentné mapovanie českých názvov na slovenské
   - Rozpoznáva všedné dni vs nedele
   - Mapuje liturgické obdobia (cezročné, pôstne, atď.)
   - Porovnáva čísla týždňov a dní

3. **`normalizeTextForMapping()`**
   - Normalizuje text pre porovnávanie
   - Odstráni diakritiku a extra medzery

### Mapovanie českých výrazov:

| Česky | Slovensky |
|-------|-----------|
| `Pondělí` | `Pondelok` |
| `Úterý` | `Utorok` |
| `Středa` | `Streda` |
| `Čtvrtek` | `Štvrtok` |
| `Pátek` | `Piatok` |
| `Sobota` | `Sobota` |
| `Neděle` | `Nedeľa` |
| `Cezročním období` | `Cezročnom období` |
| `postní` | `pôstne/pôstnej/pôstna` |
| `velikonoční` | `veľkonočn*` |
| `adventní` | `adventn*` |

### Logika cyklov:

- **Nedele**: Musia mať správny lekcionárny cyklus (A/B/C) alebo 'ABC'
- **Všedné dni**: Musia mať rok 'N' alebo 'ABC' (rovnaké pre všetky cykly)

## Výhody novej functionality:

1. **Rýchlosť**: Eliminuje potrebu AI prekladu každého dňa
2. **Presnosť**: Priame mapovanie na overené slovenské texty
3. **Konzistentnosť**: Používa existujúce lectio_sources ako autoritný zdroj
4. **Efektivita**: Menej API volaní a rýchlejšie spracovanie

## Použitie:

1. Otvorte `/admin/liturgical-calendar`
2. Kliknite na **"🇸🇰 Zo SK zdrojov"** (zelené tlačidlo)
3. Potvrďte generovanie
4. Systém automaticky:
   - Načíta slovenské lectio_sources
   - Získa štruktúru roku z API
   - Namapuje dni na slovenské texty
   - Vytvorí kompletný slovenský kalendár

## Testovanie:

Mapovacia logika bola testovaná s 100% úspešnosťou na vzorových dátach:
- ✅ `Pondělí 1. týdne v Cezročním období` → `Pondelok 1. týždňa v Cezročnom období`
- ✅ `2. neděle v Cezročním období` → `2. nedeľa v Cezročnom období`  
- ✅ `Středa 1. týdne postního` → `Streda 1. pôstneho týždňa`

---

*Implementované: 1. november 2025*
*Status: ✅ Pripravené na produkčné použitie*