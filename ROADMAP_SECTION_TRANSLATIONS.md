# RoadmapSection - Multilingual Support

## Overview
The RoadmapSection component displays the development timeline/milestones for Lectio Divina app with full multilingual support for SK/CZ/EN/ES languages.

## Files Structure

### Translation File
**Location:** `/backend/src/app/components/roadmapTranslations.ts`

**Supported Languages:**
- 🇸🇰 Slovak (sk)
- 🇨🇿 Czech (cz)
- 🇬🇧 English (en)
- 🇪🇸 Spanish (es)

### Component File
**Location:** `/backend/src/app/components/RoadmapSection.tsx`

## Translation Keys Structure

```typescript
interface RoadmapTranslations {
  badge: string;              // Top badge text
  title: string;              // Main heading
  subtitle: string;           // Subtitle below title
  description: string;        // Inspirational description text
  milestones: {
    start: {
      date: string;           // e.g., "11/2022"
      title: string;          // Milestone title
      description: string;    // Milestone description
    };
    branding: {...};          // New logo & branding milestone
    website: {...};           // New website milestone
    flutter: {...};           // Flutter app milestone
    expansion: {...};         // Language expansion milestone
    portuguese: {...};        // Portuguese language milestone
  };
}
```

## Current Milestones

### 1. Project Start (11/2022)
- **Icon:** Rocket 🚀
- **Color:** Blue to Cyan gradient
- **Position:** Top
- **Key:** `t.milestones.start`

### 2. New Logo & Branding (10/2025)
- **Icon:** Sparkles ✨
- **Color:** Purple to Pink gradient
- **Position:** Bottom
- **Key:** `t.milestones.branding`

### 3. New Website (11/2025)
- **Icon:** Globe 🌐
- **Color:** Amber to Orange gradient
- **Position:** Top
- **Key:** `t.milestones.website`

### 4. Flutter App (Q1/2026)
- **Icon:** Sparkles ✨
- **Color:** Green to Emerald gradient
- **Position:** Bottom
- **Key:** `t.milestones.flutter`

### 5. Language Expansion (Q4/2026)
- **Icon:** Languages 🌍
- **Color:** Teal to Cyan gradient
- **Position:** Top
- **Key:** `t.milestones.expansion`

### 6. Portuguese Language (Q1/2027)
- **Icon:** Globe 🌐
- **Color:** Rose to Red gradient
- **Position:** Bottom
- **Key:** `t.milestones.portuguese`

## How It Works

### 1. Language Context
The component uses the `useLanguage()` hook from `LanguageProvider`:

```tsx
const { lang } = useLanguage();
const t = roadmapTranslations[lang];
```

### 2. Dynamic Content
All text content is dynamically loaded based on the current language:

```tsx
// Header
<span>{t.badge}</span>
<h2>{t.title}</h2>
<p>{t.subtitle}</p>
<p>{t.description}</p>

// Milestones
milestones.map(milestone => (
  <div>
    <span>{milestone.date}</span>
    <h3>{milestone.title}</h3>
    <p>{milestone.description}</p>
  </div>
))
```

### 3. Automatic Updates
When user changes language via language selector:
1. `lang` state updates in LanguageProvider
2. Component re-renders with new translations
3. All text updates automatically

## Adding New Languages

To add a new language (e.g., Italian):

1. **Add to Type Definition:**
```typescript
export type Language = "sk" | "cz" | "en" | "es" | "it";
```

2. **Add Translation Object:**
```typescript
export const roadmapTranslations: Record<string, RoadmapTranslations> = {
  // ... existing languages
  it: {
    badge: "...",
    title: "...",
    subtitle: "...",
    description: "...",
    milestones: {
      start: {
        date: "11/2022",
        title: "...",
        description: "..."
      },
      // ... other milestones
    }
  }
};
```

## Adding New Milestones

To add a new milestone:

1. **Update Interface:**
```typescript
interface RoadmapTranslations {
  // ... existing keys
  milestones: {
    // ... existing milestones
    newMilestone: {
      date: string;
      title: string;
      description: string;
    };
  };
}
```

2. **Add Translations for All Languages:**
```typescript
sk: {
  milestones: {
    // ... existing
    newMilestone: {
      date: "Q2/2027",
      title: "Nový míľnik",
      description: "Popis nového míľnika"
    }
  }
},
// ... repeat for cz, en, es
```

3. **Add to Component:**
```tsx
const milestones: Milestone[] = [
  // ... existing milestones
  {
    date: t.milestones.newMilestone.date,
    title: t.milestones.newMilestone.title,
    description: t.milestones.newMilestone.description,
    icon: <IconComponent className="w-6 h-6" />,
    color: "from-color-500 to-color-500",
    position: "top" // or "bottom"
  }
];
```

## Responsive Design

### Desktop (lg: 1024px+)
- Horizontal timeline with SVG wave line
- Cards alternate top/bottom positions
- Timeline dots visible between cards and line

### Mobile (< 1024px)
- Vertical stack layout
- SVG wave line hidden
- Timeline dots hidden
- Full-width cards with spacing

## Visual Features

- **Background:** #686ea3 (indigo/purple)
- **Animations:** Framer Motion scroll-triggered animations
- **Hover Effects:** Cards scale to 1.05
- **Gradient Line:** Multi-color gradient from blue to red
- **Icons:** Lucide React icons

## Testing Translations

To test different languages:

1. Open the homepage
2. Click language selector in navigation
3. Select language (SK/CZ/EN/ES)
4. Scroll to Roadmap section
5. Verify all text is translated correctly

## Best Practices

✅ **DO:**
- Keep translations consistent across languages
- Use appropriate date formats for each language (Q1/2026 vs 1Q/2026)
- Maintain similar text length across translations
- Test on mobile and desktop

❌ **DON'T:**
- Hardcode text strings in component
- Use machine translation without review
- Skip testing all language variants
- Forget to update all language objects when adding keys

## Related Files

- `/backend/src/app/components/LanguageProvider.tsx` - Language context provider
- `/backend/src/app/i18n.ts` - Main translation file for homepage
- `/backend/src/app/components/adminSidebarTranslations.ts` - Admin sidebar translations
- `/backend/src/app/lectio/translations.ts` - Lectio page translations

## Troubleshooting

### Translations not updating
- Check if `lang` state is updating in LanguageProvider
- Verify `roadmapTranslations[lang]` returns valid object
- Check browser console for errors

### Missing translations
- Ensure all language objects have the same keys
- TypeScript will error if structure doesn't match interface
- Check for typos in translation keys

### Mobile layout issues
- Verify responsive classes (lg: prefixes)
- Test on actual mobile devices or Chrome DevTools
- Check spacing with `space-y-8` on mobile

## Future Enhancements

- [ ] Add German (DE) translations
- [ ] Add Italian (IT) translations
- [ ] Add Portuguese (PT) translations
- [ ] Add French (FR) translations
- [ ] Consider date localization with Intl.DateTimeFormat
- [ ] Add animation preferences (reduced motion support)
