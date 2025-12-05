# 🚀 Redis Cache Implementation Guide

**Dátum:** 24. november 2025  
**Status:** ✅ DOKONČENÉ

---

## 📋 Prehľad

Redis cache vrstva pre optimalizáciu databázových dotazov pomocou **Upstash Redis** (už existujúci pre rate limiting).

### Výhody:
- ⚡ **10-100x rýchlejšie čítanie** (in-memory vs database)
- 💰 **Nižšie náklady** na Supabase (menej DB queries)
- 📉 **Znížená záťaž** na databázu
- 🎯 **Lepší UX** (rýchlejšie načítanie stránok)

### Použitý Redis:
- **Provider:** Upstash Redis (FREE tier)
- **Location:** Frankfurt, EU
- **Limit:** 500K commands/month (viac než dosť)
- **Použitie:** Rate limiting + Cache (zdieľaný)

---

## 📁 Súbory

### ✅ Vytvorené:
- `src/lib/cache.ts` - cache helper funkcie

### 📝 Na úpravu:
- `src/app/api/news/route.ts` - cached news endpoint
- `src/app/admin/news/page.tsx` - cache invalidation
- Ostatné admin pages podľa potreby

---

## 🔧 Cache Helper API

### 1. Základné funkcie

```typescript
import { getCached, setCached, deleteCached, cacheQuery } from '@/lib/cache';

// Get value
const data = await getCached<NewsItem[]>("cache:news:all");

// Set value with TTL
await setCached("cache:news:all", newsData, 300); // 5 minutes

// Delete value
await deleteCached("cache:news:all");
```

### 2. Cache Query Wrapper (ODPORÚČANÉ)

```typescript
import { cacheQuery, CACHE_TTL, CACHE_PREFIX } from '@/lib/cache';

// Automatic caching
const news = await cacheQuery(
  `${CACHE_PREFIX.NEWS}:all`,
  async () => {
    const { data } = await supabase.from("news").select("*");
    return data;
  },
  CACHE_TTL.DYNAMIC // 5 minutes
);
```

### 3. Cache Invalidation

```typescript
import { invalidateResource, invalidateCache } from '@/lib/cache';

// After CREATE/UPDATE/DELETE
await invalidateResource("NEWS"); // Deletes cache:news:*
await invalidateCache("cache:news:lang:sk"); // Specific pattern
```

---

## 📊 Cache TTL Strategy

| Data Type | TTL | Dôvod |
|-----------|-----|-------|
| **Categories, Translations** | 1 hour | Statické dáta |
| **Lectio Sources, Calendar** | 15 min | Semi-statické |
| **News, Articles, Quotes** | 5 min | Dynamické |
| **User Profiles** | 5 min | Často čítané |
| **Frequently changing** | 1 min | Real-time updates |

---

## 🎯 Implementácia po stránkach

### 1. News (Novinky)

#### ✅ API Route: `src/app/api/news/route.ts`

```typescript
import { cacheQuery, CACHE_TTL, CACHE_PREFIX } from '@/lib/cache';

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url);
  const lang = searchParams.get('lang') || 'sk';
  
  // Cache key includes language
  const cacheKey = `${CACHE_PREFIX.NEWS}:lang:${lang}`;
  
  // Cached query
  const news = await cacheQuery(
    cacheKey,
    async () => {
      const { data } = await supabase
        .from("news")
        .select("*")
        .eq("lang", lang)
        .order("published_at", { ascending: false });
      
      return data;
    },
    CACHE_TTL.DYNAMIC // 5 minutes
  );
  
  return NextResponse.json(news);
}
```

#### ✅ Admin Page: `src/app/admin/news/page.tsx`

```typescript
import { invalidateResource } from '@/lib/cache';

// After DELETE
const handleDelete = async () => {
  // ... delete logic
  
  // Invalidate cache
  await invalidateResource("NEWS");
};

// After UPDATE
const handleUpdate = async () => {
  // ... update logic
  
  // Invalidate cache
  await invalidateResource("NEWS");
};

// After CREATE (bulk import)
const handleBulkImport = async () => {
  // ... import logic
  
  // Invalidate cache
  await invalidateResource("NEWS");
};
```

---

### 2. Articles (Články)

```typescript
// src/app/api/articles/route.ts
const cacheKey = `${CACHE_PREFIX.ARTICLES}:category:${categoryId}`;

const articles = await cacheQuery(
  cacheKey,
  async () => {
    return await supabase
      .from("articles")
      .select("*, article_categories(*), users(full_name)")
      .eq("category_id", categoryId)
      .order("created_at", { ascending: false });
  },
  CACHE_TTL.DYNAMIC
);
```

---

### 3. Lectio Sources (Čítania)

```typescript
// src/app/api/lectio-sources/route.ts
const cacheKey = `${CACHE_PREFIX.SOURCES}:date:${date}`;

const sources = await cacheQuery(
  cacheKey,
  async () => {
    return await supabase
      .from("lectio_sources")
      .select("*")
      .eq("date", date);
  },
  CACHE_TTL.SEMI_STATIC // 15 minutes (zmena len raz denne)
);
```

---

### 4. Categories (Kategórie)

```typescript
// src/app/api/categories/route.ts
const cacheKey = `${CACHE_PREFIX.CATEGORIES}:all`;

const categories = await cacheQuery(
  cacheKey,
  async () => {
    return await supabase.from("article_categories").select("*");
  },
  CACHE_TTL.STATIC // 1 hour (zmena veľmi zriedka)
);
```

---

### 5. User Profile

```typescript
// src/app/api/profile/[id]/route.ts
const cacheKey = `${CACHE_PREFIX.PROFILE}:${userId}`;

const profile = await cacheQuery(
  cacheKey,
  async () => {
    return await supabase
      .from("users")
      .select("*, subscriptions(*), orders(*)")
      .eq("id", userId)
      .single();
  },
  CACHE_TTL.USER // 5 minutes
);
```

---

## 🧪 Testovanie

### 1. Manuálne testovanie

```bash
# Spusti server
npm run dev

# Otvor stránku 2x (druhé načítanie by malo byť rýchlejšie)
# Skontroluj console.log:
# [Cache MISS] cache:news:all
# [Cache SET] cache:news:all (TTL: 300s)
# [Cache HIT] cache:news:all  <-- Rýchlejšie!
```

### 2. Cache invalidation test

```typescript
// V admin paneli vymaž novinku
// Console by mal ukázať:
// [Cache INVALIDATE] Deleted 3 keys matching: cache:news:*
```

### 3. Redis dashboard

- Otvor **Upstash Console**: https://console.upstash.com
- Choď na **Redis** → Tvoja databáza
- **Data Browser** → Vidíš cache keys
- **Commands** → Počet cache operácií

---

## 📈 Monitoring

### Cache Hit Rate

```typescript
// src/app/api/admin/cache-stats/route.ts
import { getCacheStats } from '@/lib/cache';

export async function GET() {
  const stats = await getCacheStats();
  return NextResponse.json(stats);
}
```

### Upstash Dashboard

- **Daily Commands:** Koľko cache operácií
- **Storage Used:** Veľkosť cached dát
- **Latency:** Rýchlosť Redis responses

---

## 🚀 Deployment Checklist

### ✅ Development:
1. [x] Vytvorený `src/lib/cache.ts`
2. [x] Vytvorený `src/lib/admin-cache-helper.ts`
3. [x] Created cached API routes:
   - [x] `/api/news` - News with caching
   - [x] `/api/articles` - Articles with caching
   - [x] `/api/lectio-sources` - Lectio sources with caching
   - [x] `/api/categories` - Categories with caching
4. [x] Admin invalidation implemented in `news` admin panel
5. [ ] Admin invalidation in `articles` admin panel (optional)
6. [ ] Admin invalidation in `lectio-sources` admin panel (optional)

### ✅ Production:
1. [ ] Testované cache hit rates
2. [ ] Monitorovať Upstash usage (mali by byť << 500K commands/month)
3. [ ] Nastaviť cache warming (cron job?)

---

## 💡 Best Practices

### ✅ DO:
- Cachuj **často čítané**, **zriedka menené** dáta
- Používaj **špecifické cache keys** (s lang, date, id)
- **Invaliduj cache** po UPDATE/DELETE/CREATE
- Používaj **cacheQuery()** wrapper (automatický caching)

### ❌ DON'T:
- Necachuj **real-time data** (chat, live updates)
- Necachuj **user-specific sensitive data** bez encryption
- Necachuj **príliš veľké objekty** (>1MB)
- Nepoužívaj príliš **dlhé TTL** na dynamické dáta

---

## 🔧 Troubleshooting

### Cache not working?

1. **Check env vars:**
   ```bash
   echo $UPSTASH_REDIS_REST_URL
   echo $UPSTASH_REDIS_REST_TOKEN
   ```

2. **Check console logs:**
   - `[Cache HIT]` = cache funguje ✅
   - `[Cache MISS]` = cache miss (normálne pri prvom requeste)
   - `[Cache ERROR]` = problém s Redis ❌

3. **Test Redis connection:**
   ```typescript
   import { getCacheStats } from '@/lib/cache';
   const stats = await getCacheStats();
   console.log(stats.connected); // Should be true
   ```

### Cache not invalidating?

1. **Check invalidation calls:**
   ```typescript
   await invalidateResource("NEWS"); // After DELETE/UPDATE
   ```

2. **Manual invalidation:**
   - Upstash Console → Data Browser → Delete keys manually

---

## 📊 Expected Performance

### Before Cache:
- News load: ~200-500ms (DB query)
- Articles load: ~300-800ms (with joins)
- Profile load: ~400-1000ms (multiple tables)

### After Cache:
- News load: ~10-50ms (Redis) ⚡
- Articles load: ~20-80ms (Redis) ⚡
- Profile load: ~30-100ms (Redis) ⚡

**Improvement: 10-20x rýchlejšie!** 🚀

---

## 🎯 Ďalšie kroky

1. [ ] Implementovať caching do `news` API route
2. [ ] Pridať invalidation do admin panels
3. [ ] Testovať cache hit rates
4. [ ] Monitorovať Upstash usage
5. [ ] Optimalizovať TTL hodnoty podľa usage patterns

---

**Status:** ✅ Cache helper ready  
**Next:** Implementovať do API routes + admin panels
