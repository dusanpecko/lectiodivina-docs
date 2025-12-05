# Shop Enable/Disable Configuration

## Ako vypnúť shop na produkcii

Shop sekcia je kontrolovaná cez environment premennú `NEXT_PUBLIC_SHOP_ENABLED`.

### Development (lokálne)
V development móde je shop automaticky povolený.

### Production
Na produkcnom serveri pridajte do `.env` alebo nastavte environment premennú:

```bash
NEXT_PUBLIC_SHOP_ENABLED=false
```

Toto zablokuje prístup k:
- `/shop` - hlavná stránka shopu
- `/shop/[slug]` - detail produktu  
- `/cart` - košík

### Admin sekcia
Admin sekcia `/admin/shop/**` **ostane vždy prístupná**, nezávisle od nastavenia.

## Ako povoliť shop

Ak chcete shop povoliť späť na produkcii:

```bash
NEXT_PUBLIC_SHOP_ENABLED=true
```

## Middleware

Blokovanie je implementované v `/src/middleware.ts`. Pri pokuse o prístup k shop route s vypnutým shopom bude používateľ automaticky presmerovaný na homepage (`/`).

## Testing

### Lokálne testovanie production nastavenia:

1. V `.env.local` nastavte:
```bash
NEXT_PUBLIC_SHOP_ENABLED=false
```

2. Reštartujte development server:
```bash
npm run dev
```

3. Skúste pristúpiť na `/shop` - mali by ste byť presmerovaní na homepage

### Kontrola v prehliadači:
```javascript
console.log(process.env.NEXT_PUBLIC_SHOP_ENABLED)
```

## Deployment

Pri nasadzovaní na produkciu nastavte environment premennú vo vašom hosting provideri:
- Vercel: Project Settings → Environment Variables
- Railway: Variables tab
- Netlify: Site Settings → Environment Variables
- Docker: pridajte do `.env` alebo docker-compose

```yaml
# docker-compose.yml príklad
environment:
  - NEXT_PUBLIC_SHOP_ENABLED=false
```
