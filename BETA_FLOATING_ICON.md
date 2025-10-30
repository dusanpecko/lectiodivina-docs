# BetaFloatingIcon Component

## Popis
Plovúca okrúhla ikona umiestnená v strede pravej strany obrazovky, ktorá indikuje beta verziu aplikácie.

## Funkcionalita
- **Floating Button**: Oranžová ikona s červeným gradientom a výstražným trojuholníkom
- **Animácie**: Pulse animácia pre upútanie pozornosti
- **Pop-up Dialog**: Po kliknutí sa zobrazí informačný dialog o beta verzii
- **Error Reporting**: Integrované nahlasovanie chýb cez existujúci ErrorReportModal
- **Viacjazyčnosť**: Podpora pre SK, CZ, EN, ES

## Použitie
```tsx
<BetaFloatingIcon language={lang} />
```

## Vlastnosti
- `language: string` - Jazyk pre lokalizáciu textov

## Dizajn
- **Veľkosť**: 64x64px okrúhla ikona
- **Farby**: Orange-to-red gradient pozadie
- **Efekty**: Hover scale, shadow, pulse animácia
- **Pozícia**: Fixed right-center (24px od pravého okraja, vertikálne centrované)
- **Z-index**: 50 (nad ostatným obsahom)

## Integrácia
- Integrovaný v `layout.tsx` cez `BetaComponents` wrapper
- Používa existujúci `ErrorReportModal` pre nahlasovanie chýb
- Pripojený na Supabase databázu pre ukladanie reportov