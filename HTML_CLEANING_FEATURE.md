# 🧹 HTML Auto-Cleaning Feature

## Popis funkcionalnosti

V Lectio Sources edit stránke je implementované automatické odstraňovanie HTML tagov pri vkladaní textu do týchto polí:

- **Lectio – text**
- **Meditatio – text** 
- **Oratio – text**
- **Contemplatio – text**
- **Actio – text**

## Ako to funguje

### 1. Automatické čistenie pri vložení (Paste)
Keď vložíte text z clipboardu (Ctrl+V / Cmd+V) do ktoréhokoľvek z týchto polí:

1. **Detekcia HTML obsahu:** Systém rozpozná, či clipboard obsahuje HTML formátovaný text
2. **Odstránenie tagov:** Automaticky odstráni všetky HTML tagy (`<p>`, `<div>`, `<span>`, `<strong>`, atď.)
3. **Čistenie entity:** Konvertuje HTML entity (`&amp;` → `&`, `&nbsp;` → medzera, atď.)
4. **Normalizácia medzier:** Odstráni prebytočné medzery a zlúči viacnásobné medzery do jednej
5. **Notifikácia:** Zobrazí potvrdzovaciu správu o odstránení HTML tagov

### 2. Vizuálny indikátor
Každé pole má vedľa názvu indikátor: **🧹 Auto-čistenie HTML**
- Zobrazuje sa pri všetkých 5 textových poliach
- Tooltip vysvetľuje funkčnosť

## Príklady použitia

### Pred (HTML obsah v clipboard):
```html
<p>Toto je <strong>dôležitý</strong> text s <em>kurzívou</em>.</p>
<div>Druhý odstavec s&nbsp;<span style="color: red;">formátovaním</span>.</div>
```

### Po (automaticky vyčistené):
```
Toto je dôležitý text s kurzívou. Druhý odstavec s formátovaním.
```

## Technické detaily

### Implementácia
- **Funkcia:** `stripHtmlTags(html: string)`
- **Handler:** `handlePasteWithHtmlStripping(e, fieldName)`
- **Event:** `onPaste` na každom textarea

### Algoritmus čistenia
1. Vytvorí dočasný DOM element
2. Vloží HTML obsah do elementu
3. Extrahuje len textový obsah (`textContent`)
4. Vyčistí HTML entity
5. Normalizuje medzery

### Podporované HTML entity
- `&amp;` → `&`
- `&lt;` → `<`
- `&gt;` → `>`
- `&quot;` → `"`
- `&#39;` → `'`
- `&nbsp;` → medzera

## User Experience

### Pozitívne stránky
- ✅ **Automatické:** Žiadna manuálna akcia potrebná
- ✅ **Intuitívne:** Funguje pri štandardnom Ctrl+V
- ✅ **Feedback:** Užívateľ dostane potvrdenie o čistení
- ✅ **Vizuálne:** Jasné označenie polí s auto-čistením
- ✅ **Zachováva obsah:** Odstráni len HTML tagy, text zostáva

### Možné scenáre použitia
1. **Kopírovanie z Word dokumentu** - odstráni Word HTML formatting
2. **Kopírovanie z webstránky** - vyčistí web HTML tagy
3. **Kopírovanie z Gmail/Outlook** - odstráni email HTML formatting
4. **Vkladanie z Google Docs** - zjednoduší formátovanie na čistý text

## Rozšírenia (budúce)

Možné vylepšenia:
- **Zachovanie odstavcov:** Konverzia `<p>` na nové riadky
- **Zachovanie zoznamov:** Konverzia `<li>` na bullet points  
- **Selektívne čistenie:** Možnosť zapnúť/vypnúť pre jednotlivé polia
- **Preview:** Zobrazenie pred a po čistení

## Testovanie

Pre testovanie funkčnosti:
1. Otvorte ľubovoľný Lectio Sources záznam na úpravu
2. Skopírujte formátovaný text z web stránky alebo Word dokumentu
3. Vložte do niektorého z označených polí (Lectio, Meditatio, atď.)
4. Overte, že HTML tagy boli odstránené a zobrazila sa správa o čistení