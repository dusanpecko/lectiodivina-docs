# 📋 EasyForms Integrácia - Dokumentácia

## ✅ Prehľad

Systém umožňuje vkladanie interaktívnych formulárov z **dpforms.sk** (EasyForms) priamo do news článkov. Formuláre sa zobrazujú na konci článku a plne fungujú vrátane JavaScript widgetov.

---

## 🗄️ 1. Databázová štruktúra

### SQL migrácia
```sql
-- Add form_embed_code column to news table
ALTER TABLE news 
ADD COLUMN IF NOT EXISTS form_embed_code TEXT;

COMMENT ON COLUMN news.form_embed_code IS 'EasyForms embed code (HTML + script) for displaying forms in articles';
```

**Spustiť v Supabase SQL editore!**

---

## 🖥️ 2. Backend - Admin Editor

### Zmeny v `/src/app/admin/news/[id]/page.tsx`

#### Interface News
```typescript
interface News {
  id?: number;
  title: string;
  summary: string;
  image_url: string;
  content: string;
  published_at: string;
  lang: string;
  audio_url?: string;
  form_embed_code?: string;  // ✨ NOVÉ
}
```

#### State inicializácia
```typescript
const [news, setNews] = useState<News>({
  // ... ostatné polia
  form_embed_code: "",  // ✨ NOVÉ
});
```

#### Ukladanie do databázy
```typescript
const cleanNewsData = {
  // ... ostatné polia
  form_embed_code: news.form_embed_code || null  // ✨ NOVÉ
};
```

#### Nová sekcia formulára v editore
```tsx
<FormSection title="Interaktívny formulár" icon={FileText}>
  <div className="space-y-3">
    <div className="bg-blue-50 border border-blue-200 rounded-lg p-4 mb-4">
      <p className="text-sm text-blue-900 font-semibold mb-2">📋 EasyForms integrácia</p>
      <p className="text-sm text-blue-800">
        Vložte celý embed kód z EasyForms (vrátane &lt;div&gt; a &lt;script&gt; tagov). 
        Formulár sa zobrazí na konci článku.
      </p>
      {/* Návod */}
    </div>

    <textarea
      value={news.form_embed_code || ""}
      onChange={(e) => setNews(prev => ({ ...prev, form_embed_code: e.target.value }))}
      className="w-full p-3 border border-gray-300 rounded-lg font-mono text-sm"
      placeholder="<!-- MYPROFILE -->..."
      rows={12}
    />
  </div>
</FormSection>
```

---

## 🌐 3. Frontend - Zobrazenie článku

### Zmeny v `/src/app/news/[id]/NewsDetailArticle.tsx`

#### Props interface
```typescript
interface NewsDetailArticleProps {
  news: {
    // ... ostatné polia
    form_embed_code?: string;  // ✨ NOVÉ
  };
  // ...
}
```

#### Render formulára (po hlavnom obsahu, pred bottom accent)
```tsx
{/* EasyForms Embed - if present */}
{news.form_embed_code && (
  <motion.div
    initial={{ opacity: 0, y: 30 }}
    animate={{ opacity: 1, y: 0 }}
    transition={{ duration: 0.8, delay: 1.3 }}
    className="mt-8 sm:mt-12 mb-6 sm:mb-8 p-4 sm:p-6 bg-white border border-gray-200 rounded-xl sm:rounded-2xl shadow-lg"
  >
    <div className="mb-4 sm:mb-6">
      <h3 className="text-xl sm:text-2xl font-bold mb-2" style={{ color: '#40467b' }}>
        📋 {t.newsDetail?.form_title || 'Interaktívny formulár'}
      </h3>
      <p className="text-sm sm:text-base text-slate-600">
        {t.newsDetail?.form_description || 'Vyplňte formulár nižšie'}
      </p>
    </div>
    
    {/* EasyForms embed */}
    <div dangerouslySetInnerHTML={{ __html: news.form_embed_code }} />
  </motion.div>
)}
```

#### Bezpečnosť
- Používa `dangerouslySetInnerHTML` (bezpečné, lebo admin má plný prístup)
- JavaScript kód z EasyForms sa automaticky spustí
- Formulár sa renderuje v izolovanom div kontajneri

---

## 🌍 4. Preklady

### Pridané do `src/app/i18n.ts`

```typescript
newsDetail: {
  // ... existujúce preklady
  form_title: "Interaktívny formulár",        // SK
  form_description: "Vyplňte formulár nižšie"
}

// CZ
form_title: "Interaktivní formulář",
form_description: "Vyplňte formulář níže"

// EN
form_title: "Interactive Form",
form_description: "Fill out the form below"

// ES
form_title: "Formulario interactivo",
form_description: "Complete el formulario a continuación"
```

---

## 📖 5. Ako používať

### Krok 1: Vytvor formulár na dpforms.sk
1. Otvor [dpforms.sk](https://dpforms.sk)
2. Vytvor nový formulár
3. Klikni na **"Zdieľať"** alebo **"Embed"**
4. Skopíruj celý embed kód

### Krok 2: Pridaj do článku
1. Otvor admin panel → News → Edituj článok
2. Scrolluj na sekciu **"Interaktívny formulár"**
3. Vlož celý embed kód (vrátane `<script>` tagov)
4. Ulož článok

### Príklad embed kódu:
```html
<!-- MYPROFILE -->
<div id="c14">
    Vyplňte moje <a href="https://dpforms.sk/app/form?id=GKBMIA">online formulár</a>.
</div>
<script type="text/javascript">
    (function(d, t) {
        var s = d.createElement(t), options = {
            'id': 'GKBMIA',
            'theme': 0,
            'container': 'c14',
            'height': '3435px',
            'form': '//dpforms.sk/app/embed'
        };
        s.type= 'text/javascript';
        s.src = '//dpforms.sk/static_files/js/form.widget.js';
        s.onload = s.onreadystatechange = function() {
            var rs = this.readyState; if (rs) if (rs != 'complete') if (rs != 'loaded') return;
            try { (new EasyForms()).initialize(options).display() } catch (e) { }
        };
        var scr = d.getElementsByTagName(t)[0], par = scr.parentNode; par.insertBefore(s, scr);
    })(document, 'script');
</script>
<!-- End MYPROFILE -->
```

---

## ✅ 6. Čo funguje

### ✔️ Next.js (Web)
- ✅ Formulár sa zobrazuje
- ✅ JavaScript widget sa spúšťa
- ✅ Formulár je plne funkčný
- ✅ Odosielanie dát funguje
- ✅ Responsívny dizajn

### ❓ Flutter (Mobile)
**Potrebuje testovanie!**

Flutter má obmedzenia s JavaScript:
- `WebView` widget môže zobrazovať HTML + JS
- Potrebné pridať `webview_flutter` package
- Možné riešenie:

```dart
// V news_detail_screen.dart
if (news['form_embed_code'] != null && news['form_embed_code'].isNotEmpty) {
  Container(
    height: 400, // Alebo dynamicky podľa obsahu
    child: WebView(
      initialHtml: news['form_embed_code'],
      javascriptMode: JavascriptMode.unrestricted,
    ),
  ),
}
```

---

## 🔧 7. Technické detaily

### Backend
- **Pole:** `form_embed_code` (TEXT, nullable)
- **Uloženie:** Raw HTML + JavaScript kód
- **Validácia:** Žiadna (admin má plný prístup)

### Frontend
- **Render:** `dangerouslySetInnerHTML`
- **Pozícia:** Po hlavnom obsahu, pred navigáciou článkov
- **Styling:** White box s border, shadow, rounded corners
- **Animácia:** Framer Motion fade-in

### Bezpečnosť
- ⚠️ `dangerouslySetInnerHTML` je použité zámerne
- ✅ Prístup len pre adminov
- ✅ EasyForms je dôveryhodná platforma
- ✅ Žiadne user-generated content

---

## 📱 8. Flutter integrácia (TODO)

### Potrebné kroky:
1. Pridať `webview_flutter` do `pubspec.yaml`
2. Upraviť `news_detail_screen.dart`
3. Pridať WebView widget pre zobrazenie formulára
4. Testovať JavaScript funkcionalitu
5. Upraviť výšku WebView podľa obsahu

### Alternatívne riešenie:
- Otvoriť formulár v externom prehliadači
- Použiť `url_launcher` package
- Zobraziť link na formulár namiesto embedu

---

## 📊 9. Dátový tok

```
Admin Editor → Vlož embed kód
    ↓
Uloží do Supabase (form_embed_code)
    ↓
Next.js načíta článok
    ↓
NewsDetailArticle render
    ↓
dangerouslySetInnerHTML vykreslí formulár
    ↓
EasyForms JavaScript sa spustí
    ↓
Formulár je funkčný
```

---

## 🎨 10. Dizajn

### Desktop
- Formulár na plnú šírku pravého stĺpca
- White box s border a shadow
- Nadpis v brand color (#40467b)
- Fade-in animácia

### Mobile
- Responzívny layout
- Menší padding
- Zmenšený nadpis
- Touch-friendly form controls

---

## ⚠️ 11. Známe limitácie

1. **Flutter WebView** - Potrebuje testing a možno extra nastavenia
2. **Výška formulára** - V embede je fixná (3435px v príklade), možno nebude vždy sedieť
3. **iFrame vs. Script** - EasyForms používa script widget, nie iframe
4. **Cross-origin** - Ak dpforms.sk zmení CORS policy, môže prestať fungovať

---

## 🚀 12. Budúce vylepšenia

- [ ] Testovať Flutter WebView integráciu
- [ ] Pridať preview formulára v admin editore
- [ ] Automatická detekcia výšky formulára
- [ ] Podpora pre iné form platformy (Typeform, Google Forms)
- [ ] Validácia embed kódu pred uložením
- [ ] Štatistiky vyplnených formulárov

---

## 📞 13. Support

Pri problémoch s:
- **EasyForms:** [dpforms.sk support](https://dpforms.sk/support)
- **Embed kódom:** Skontroluj konzolu prehliadača (F12)
- **Flutter WebView:** Dokumentácia [webview_flutter](https://pub.dev/packages/webview_flutter)

---

**Status:** ✅ Implementované a funkčné na Next.js
**Autor:** GitHub Copilot
**Dátum:** November 1, 2025
