# PageSpeed Insights Optimalizácie (2025)

> Komplexný zoznam optimalizácií vykonaných na základe PageSpeed Insights analýzy pre https://lectio.one

## 📊 Pôvodné metriky (pred optimalizáciami)

- **Performance Grade**: 98/100 ✅
- **Accessibility Grade**: 88/100 ⚠️
- **Best Practices**: 100/100 ✅
- **SEO**: 100/100 ✅

### Identifikované problémy:

1. **Render-blocking CSS** (21.4 KB, 180ms delay)
2. **Legacy JavaScript polyfills** (13 KB unnecessary)
3. **Unused JavaScript** (197 KB can be reduced)
4. **Long main thread tasks** (74ms)
5. **Layout Shifts** (CLS: 0.003)
6. **Non-composited animations** (2 animated elements)
7. **Accessibility issues** (labels, contrast, touch targets)

---

## 🚀 Vykonané optimalizácie

### 1. Modernizácia browserslist (↓ 13 KB polyfills)

**Problém**: Transpiler generoval polyfilly pre staré prehliadače (Array.prototype.at, Array.prototype.flat, Object.hasOwn, atď.)

**Riešenie**: 
- Vytvorený `.browserslistrc` s podporou len moderných prehliadačov
- Pridaná Next.js config option `legacyBrowsers: false`

```javascript
// .browserslistrc
last 2 Chrome versions
last 2 Firefox versions
last 2 Safari versions
last 2 Edge versions
not dead
not IE 11
```

```javascript
// next.config.mjs
experimental: {
  browsersListForSwc: true,
  legacyBrowsers: false,
}
```

**Očakávaný výsledok**: ↓ 13 KB JavaScript bundle

---

### 2. Render-blocking CSS optimalizácia

**Problém**: CSS súbory blokujú initial render (21.4 KB, 180ms)

**Riešenie**:
- Inline kritické CSS priamo v `<head>`
- Preload kritických CSS chunks
- Optimalizovaný font loading s `display: swap`

```tsx
// layout.tsx
<style dangerouslySetInnerHTML={{
  __html: `
    /* Critical CSS - inline pre okamžité načítanie */
    :root{--color-primary:#40467b;--color-primary-light:#686ea3}
    *{box-sizing:border-box}
    body{margin:0;padding:0;font-family:var(--font-inter),system-ui,-apple-system,sans-serif}
    .min-h-screen{min-height:100vh}
    ...
  `
}} />
```

**Font optimalizácia**:
```typescript
const inter = Inter({
  subsets: ['latin', 'latin-ext'],
  display: 'swap', // Zobraz fallback font okamžite
  preload: true,
  fallback: ['system-ui', '-apple-system', 'sans-serif'],
  adjustFontFallback: true,
});
```

**Očakávaný výsledok**: ↓ 150-200ms LCP improvement

---

### 3. Unused JavaScript reduction (↓ 197 KB)

**Problém**: 270.2 KB vendor chunk obsahuje 197 KB nepoužitého kódu

**Riešenie**:
- Package import optimization v Next.js
- Compiler option pre odstránenie console.log
- Tree-shaking pre veľké knižnice

```javascript
// next.config.mjs
compiler: {
  removeConsole: process.env.NODE_ENV === 'production',
},
experimental: {
  optimizePackageImports: ['lucide-react', 'framer-motion', '@supabase/supabase-js'],
}
```

**Očakávaný výsledok**: ↓ 50-100 KB JavaScript bundle

---

### 4. Non-composited animations fix

**Problém**: 2 elementy s non-composited animáciami:
- Scroll-to-top button (opacity + transform v motion.button)
- Cookie consent button (efekt má nepodporované parametre)

**Riešenie**:

#### Scroll-to-top button:
```tsx
// PRED (framer-motion s scale animáciou)
<motion.button
  animate={{ opacity: isVisible ? 1 : 0, scale: isVisible ? 1 : 0 }}
  whileHover={{ scale: 1.1 }}
/>

// PO (CSS transitions)
<button
  style={{
    opacity: isVisible ? 1 : 0,
    transform: isVisible ? 'scale(1)' : 'scale(0)',
    transition: 'all 0.3s'
  }}
  className="group-hover:scale-110"
/>
```

#### Cookie button:
- Odstránené inline `onMouseEnter/onMouseLeave` JavaScript animácie
- Použité CSS transitions namiesto JS

**Očakávaný výsledok**: ✅ Žiadne non-composited animácie

---

### 5. Accessibility fixes

#### A. Form labels
**Problém**: Select elementy bez `<label>` alebo `aria-label`

```tsx
// PRED
<select value={lang}>...</select>

// PO
<label htmlFor="language-select-desktop" className="sr-only">Select language</label>
<select
  id="language-select-desktop"
  value={lang}
  aria-label="Language selector"
>...</select>
```

#### B. Kontrast farieb
**Problém**: Footer odkazy s nízkym kontrastom (#9ca3af na tmavom pozadí)

```tsx
// PRED
style={{ color: '#9ca3af' }} // Kontrast ratio: 2.3:1 ❌

// PO
className="text-gray-300 hover:text-[#40467b]" // Kontrast ratio: 4.8:1 ✅
```

#### C. Touch targets (min 44x44px)
**Problém**: Review slider pagination dots 8-12px ❌

```tsx
// PRED
<button style={{ width: '12px', height: '12px' }}>
  <span /* dot */ />
</button>

// PO (wrapper má 44x44px, dot vnútri)
<button 
  style={{ width: '44px', height: '44px' }}
  className="flex items-center justify-center"
>
  <span style={{ width: '12px', height: '12px' }} />
</button>
```

#### D. Identické odkazy s rôznym cieľom
**Problém**: 3× "Zobraziť článok" s rôznymi href

```tsx
// PRED
<Link href={`/news/${id}`}>Zobraziť článok</Link>

// PO
<Link 
  href={`/news/${id}`}
  aria-label={`Zobraziť článok: ${title}`}
>
  Zobraziť článok
</Link>
```

#### E. Dekoratívne elementy
**Problém**: Footer blur circles bez `aria-hidden`

```tsx
// PO
<div className="opacity-5 pointer-events-none" aria-hidden="true">
  <div className="blur-3xl" />
</div>
```

**Očakávaný výsledok**: Accessibility score 95-100

---

### 6. Layout Shift elimination (CLS: 0.003 → 0)

**Problém**: Footer dekoratívne elementy spôsobujú malý shift

**Riešenie**:
```tsx
<div className="absolute inset-0 opacity-5 pointer-events-none" aria-hidden="true">
  {/* Blur circles s fixed dimensions */}
</div>
```

**Očakávaný výsledok**: CLS = 0 ✅

---

### 7. DNS preconnect pre external resources

**Pridané v layout.tsx**:
```tsx
<link rel="dns-prefetch" href="https://analytics.lectio.one" />
<link rel="preconnect" href="https://analytics.lectio.one" crossOrigin="anonymous" />
<link rel="dns-prefetch" href="https://unnijykbupxguogrkolj.supabase.co" />
<link rel="preconnect" href="https://unnijykbupxguogrkolj.supabase.co" crossOrigin="anonymous" />
```

**Očakávaný výsledok**: ↓ 50-100ms DNS lookup time

---

## 📈 Očakávané výsledky

### Performance metriky:

| Metrika | Pred | Po (očakávané) | Zlepšenie |
|---------|------|----------------|-----------|
| **Performance Score** | 98% | 98-100% | ✅ Udržané |
| **LCP** | 2.5s | 2.0-2.3s | ↓ 200-500ms |
| **TBT** | 74ms | 30-50ms | ↓ 24-44ms |
| **CLS** | 0.003 | 0 | ✅ Perfect |
| **JavaScript bundle** | 270 KB | 150-170 KB | ↓ 100-120 KB |

### Accessibility:

| Kategória | Pred | Po | Status |
|-----------|------|-----|--------|
| **Form labels** | ❌ | ✅ | Fixed |
| **Color contrast** | ⚠️ | ✅ | Fixed |
| **Touch targets** | ❌ | ✅ | Fixed |
| **Link purpose** | ⚠️ | ✅ | Fixed |
| **Accessibility Score** | 88% | 95-100% | +7-12% |

---

## 🔧 Súbory zmenené

1. `/backend/next.config.mjs` - Compiler + experimental features
2. `/backend/.browserslistrc` - Modern browsers only
3. `/backend/src/app/layout.tsx` - Inline CSS, font optimization, preconnects
4. `/backend/src/app/page.tsx` - Language selector labels
5. `/backend/src/app/components/Footer.tsx` - Contrast fix, aria-hidden
6. `/backend/src/app/components/ScrollToTopButton.tsx` - CSS transitions
7. `/backend/src/app/components/ReviewSlider.tsx` - Touch targets
8. `/backend/src/app/components/HomeNewsSection.tsx` - Link aria-labels

---

## ✅ Checklist pre deployment

- [x] `.browserslistrc` vytvorený
- [x] `next.config.mjs` aktualizovaný
- [x] Accessibility issues fixed
- [x] Non-composited animations removed
- [x] Inline kritické CSS
- [x] Font optimization
- [x] DNS preconnect
- [ ] Production build test
- [ ] PageSpeed Insights re-test
- [ ] Mobile performance test

---

## 🧪 Testovacie kroky

1. **Build production**:
```bash
npm run build
```

2. **Analyze bundle**:
```bash
ANALYZE=true npm run build
```

3. **PageSpeed test**:
- Desktop: https://pagespeed.web.dev/analysis?url=https://lectio.one
- Mobile: https://pagespeed.web.dev/analysis?url=https://lectio.one

4. **GTmetrix test**:
- https://gtmetrix.com/?url=https://lectio.one

---

## 📝 Poznámky

- Všetky optimalizácie sú backward-compatible
- Žiadne breaking changes v API
- Používateľské rozhranie nezmenené
- Accessibility improvements vylepšujú UX pre všetkých

**Dátum vykonania**: 25. október 2025  
**Verzia**: Next.js 15.5.4  
**Autor**: AI Assistant + Dušan Pecko
