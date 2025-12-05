# 📚 Cache API Routes - Usage Examples

**Created:** 24. november 2025  
**Status:** ✅ IMPLEMENTED

---

## 🎯 Available Cached API Routes

### 1. News API ✅ IMPLEMENTED IN ADMIN
**Endpoint:** `/api/news`  
**Cache TTL:** 5 minutes (DYNAMIC)  
**Query params:**
- `lang` - Language filter (default: "sk")
- `page` - Page number (default: 1)
- `limit` - Items per page (default: 20)
- `search` - Global search (title, summary, content)
- `title` - Title filter
- `summary` - Summary filter
- `content` - Content filter
- `dateFrom` - Date from (YYYY-MM-DD)
- `dateTo` - Date to (YYYY-MM-DD)

**Admin Panel Integration:**
```typescript
// src/app/admin/news/page.tsx - NOW USES CACHED API! ✅
const fetchNews = useCallback(async () => {
  const params = new URLSearchParams({
    lang: filterLang,
    page: page.toString(),
    limit: PAGE_SIZE.toString(),
  });

  if (globalSearch) {
    params.append('search', globalSearch);
  } else {
    if (filter.title) params.append('title', filter.title);
    if (filter.summary) params.append('summary', filter.summary);
    if (filter.content) params.append('content', filter.content);
    if (filter.dateFrom) params.append('dateFrom', filter.dateFrom);
    if (filter.dateTo) params.append('dateTo', filter.dateTo);
  }

  const response = await fetch(`/api/news?${params.toString()}`);
  const result = await response.json();
  
  setNews(result.data || []);
  setTotal(result.total || 0);
}, [filterLang, globalSearch, filter, page]);

// Cache invalidation after DELETE/BULK_IMPORT
await invalidateAdminCache("news");
```

**Example:**
```bash
# Basic fetch
curl 'http://localhost:3000/api/news?lang=sk&page=1&limit=20'

# Search
curl 'http://localhost:3000/api/news?lang=sk&search=liturgia'

# Filters
curl 'http://localhost:3000/api/news?lang=sk&title=modlitba&dateFrom=2025-01-01'

# Invalidate cache after changes
curl -X POST http://localhost:3000/api/news/invalidate
```

---

### 2. Articles API
**Endpoint:** `/api/articles`  
**Cache TTL:** 5 minutes (DYNAMIC)  
**Query params:**
- `category` - Category ID filter
- `author` - Author ID filter
- `page` - Page number (default: 1)
- `limit` - Items per page (default: 20)
- `search` - Search query

**Example:**
```typescript
// Fetch articles by category
const response = await fetch("/api/articles?category=abc123&page=1");
const { data, total, totalPages } = await response.json();

// Invalidate cache
await fetch("/api/articles/invalidate", { method: "POST" });
```

---

### 3. Lectio Divina API ✅ READY
**Endpoint:** `/api/lectio`  
**Cache TTL:** 15 minutes (SEMI_STATIC)  
**Query params:**
- `date` - Date (YYYY-MM-DD) - **required**
- `lang` - Language (sk, cz, en, es) - default: "sk"

**Features:**
- Automatic liturgical calendar lookup
- Liturgical year cycle detection (A/B/C vs N)
- Multi-language fallback (if requested lang not found, tries Slovak)
- Special day detection (weekday vs Sunday/feast)
- Complete lectio divina data (biblia_1/2/3, lectio, meditatio, oratio, contemplatio, actio)

**Example:**
```bash
# Fetch today's lectio
curl 'http://localhost:3000/api/lectio?date=2025-11-24&lang=sk'

# Response:
{
  "lectioData": {
    "id": 123,
    "hlava": "Ondrej_apostol",
    "biblia_1": "...",
    "lectio_text": "...",
    "meditatio_text": "...",
    "oratio_text": "...",
    "contemplatio_text": "...",
    "actio_text": "...",
    "lang": "sk",
    "rok": "N"
  },
  "celebrationTitle": "Sv. Ondreja Dung-Laca, kňaza, a druhov, mučeníkov"
}

# Invalidate cache after changes
curl -X POST http://localhost:3000/api/lectio/invalidate
```

**Usage in code:**
```typescript
// Fetch lectio for specific date
const date = '2025-11-24';
const response = await fetch(`/api/lectio?date=${date}&lang=sk`);
const { lectioData, celebrationTitle } = await response.json();

// Invalidate cache (after admin updates lectio sources)
await invalidateAdminCache("lectio");
```

---

### 4. Lectio Sources API
**Endpoint:** `/api/lectio-sources`  
**Cache TTL:** 15 minutes (SEMI_STATIC)  
**Query params:**
- `date` - Date filter (YYYY-MM-DD)
- `lang` - Language filter (default: "sk")
- `page` - Page number (default: 1)
- `limit` - Items per page (default: 50)

**Example:**
```typescript
// Fetch today's lectio sources
const today = new Date().toISOString().split("T")[0];
const response = await fetch(`/api/lectio-sources?date=${today}&lang=sk`);
const { data, total } = await response.json();

// Invalidate cache
await fetch("/api/lectio-sources/invalidate", { method: "POST" });
```

---

### 4. Categories API
**Endpoint:** `/api/categories`  
**Cache TTL:** 1 hour (STATIC)  
**Query params:** None

**Example:**
```typescript
// Fetch all categories (cached for 1 hour)
const response = await fetch("/api/categories");
const { data, total } = await response.json();

// Invalidate cache (rarely needed)
await fetch("/api/categories/invalidate", { method: "POST" });
```

---

## 🔧 Admin Panel Integration

### Using the Admin Cache Helper

```typescript
import { invalidateAdminCache } from "@/lib/admin-cache-helper";

// After DELETE
const handleDelete = async (id: number) => {
  await supabase.from("news").delete().eq("id", id);
  
  // Invalidate cache
  await invalidateAdminCache("news");
  
  // Refresh UI
  fetchNews();
};

// After CREATE
const handleCreate = async (newItem: News) => {
  await supabase.from("news").insert(newItem);
  
  // Invalidate cache
  await invalidateAdminCache("news");
  
  fetchNews();
};

// After UPDATE
const handleUpdate = async (id: number, updates: Partial<News>) => {
  await supabase.from("news").update(updates).eq("id", id);
  
  // Invalidate cache
  await invalidateAdminCache("news");
  
  fetchNews();
};

// After BULK IMPORT
const handleBulkImport = async (items: News[]) => {
  await supabase.from("news").insert(items);
  
  // Invalidate cache
  await invalidateAdminCache("news");
  
  fetchNews();
};
```

---

## 🚀 Frontend Usage Examples

### React Component (Client-side)

```typescript
"use client";

import { useState, useEffect } from "react";

function NewsPage() {
  const [news, setNews] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function fetchNews() {
      try {
        const response = await fetch("/api/news?lang=sk&page=1&limit=10");
        const { data } = await response.json();
        setNews(data);
      } catch (error) {
        console.error("Failed to fetch news:", error);
      } finally {
        setLoading(false);
      }
    }

    fetchNews();
  }, []);

  if (loading) return <div>Loading...</div>;

  return (
    <div>
      {news.map((item) => (
        <div key={item.id}>
          <h2>{item.title}</h2>
          <p>{item.summary}</p>
        </div>
      ))}
    </div>
  );
}
```

---

### Server Component (Server-side)

```typescript
// app/news/page.tsx
import { createClient } from "@/app/lib/supabase/server";
import { cacheQuery, CACHE_TTL, CACHE_PREFIX } from "@/lib/cache";

export default async function NewsPage() {
  // Cached server-side data fetching
  const cacheKey = `${CACHE_PREFIX.NEWS}:public:lang:sk`;
  
  const result = await cacheQuery(
    cacheKey,
    async () => {
      const supabase = await createClient();
      const { data } = await supabase
        .from("news")
        .select("*")
        .eq("lang", "sk")
        .order("published_at", { ascending: false })
        .limit(10);
      
      return { data: data || [] };
    },
    CACHE_TTL.DYNAMIC
  );

  return (
    <div>
      {result.data.map((item) => (
        <div key={item.id}>
          <h2>{item.title}</h2>
          <p>{item.summary}</p>
        </div>
      ))}
    </div>
  );
}
```

---

## 📊 Cache Performance Monitoring

### Check Cache Status

```typescript
import { getCacheStats } from "@/lib/cache";

// Get Redis connection status
const stats = await getCacheStats();
console.log(stats.connected); // true/false
```

### Upstash Dashboard
1. Open https://console.upstash.com
2. Go to your Redis database
3. Check **Commands** tab - see cache hit/miss rates
4. Check **Data Browser** - see cached keys
5. Monitor **Daily Commands** - should stay under 500K/month (FREE tier)

---

## 🔍 Cache Key Patterns

### News Cache Keys
```
cache:news:lang:sk:page:1:limit:20:search:
cache:news:lang:en:page:1:limit:20:search:python
cache:news:lang:cs:page:2:limit:10:search:
```

### Articles Cache Keys
```
cache:articles:cat:abc123:page:1:limit:20:search::author:
cache:articles:cat::page:1:limit:20:search:react:author:user123
```

### Lectio Sources Cache Keys
```
cache:sources:date:2025-11-24:lang:sk:page:1:limit:50
cache:sources:date:2025-11-25:lang:en:page:1:limit:50
```

### Categories Cache Keys
```
cache:categories:all
```

---

## ⚡ Performance Expectations

### Before Cache (Direct DB Query)
- News load: ~200-500ms
- Articles load: ~300-800ms (with joins)
- Lectio sources: ~150-400ms
- Categories: ~100-200ms

### After Cache (Redis)
- News load: ~10-50ms ⚡ (**10-20x faster**)
- Articles load: ~20-80ms ⚡
- Lectio sources: ~15-60ms ⚡
- Categories: ~5-30ms ⚡

### Cache Hit Rate Target
- **Goal:** >80% cache hits
- **Monitor:** Upstash Commands dashboard
- **Optimize:** Adjust TTL if hit rate is low

---

## 🧪 Testing Cache

### 1. Test Cache HIT
```bash
# First request (CACHE MISS)
curl http://localhost:3000/api/news?lang=sk

# Check server console:
# [Cache MISS] cache:news:lang:sk:page:1:limit:20:search:
# [Cache SET] cache:news:lang:sk:page:1:limit:20:search: (TTL: 300s)

# Second request (CACHE HIT)
curl http://localhost:3000/api/news?lang=sk

# Check server console:
# [Cache HIT] cache:news:lang:sk:page:1:limit:20:search:
```

### 2. Test Cache Invalidation
```bash
# Invalidate news cache
curl -X POST http://localhost:3000/api/news/invalidate

# Check response:
# {"success":true,"message":"Invalidated 3 cache keys","deletedKeys":3}

# Next request will be CACHE MISS again
curl http://localhost:3000/api/news?lang=sk
```

### 3. Monitor Upstash Dashboard
- Open https://console.upstash.com
- Go to **Redis** → Your database
- **Data Browser** → See cached keys in real-time
- **Commands** → See cache operations count

---

## 🎯 Best Practices

### ✅ DO:
- **Always invalidate cache** after CREATE/UPDATE/DELETE in admin panels
- **Use specific cache keys** with all relevant params (lang, page, category)
- **Monitor cache hit rates** in Upstash dashboard
- **Adjust TTL** based on data change frequency
- **Use server components** when possible (better caching)

### ❌ DON'T:
- **Don't cache user-specific sensitive data** without encryption
- **Don't use too long TTL** for frequently changing data
- **Don't forget to invalidate** after mutations
- **Don't cache real-time data** (chat, live updates)

---

## 📈 Next Steps

### Optional Improvements:
1. [ ] Add cache warming (cron job) for popular pages
2. [ ] Implement cache tags for better invalidation
3. [ ] Add cache analytics dashboard in admin panel
4. [ ] Implement stale-while-revalidate pattern
5. [ ] Add cache compression for large responses

---

**Status:** ✅ Fully implemented and documented  
**Ready for production:** YES  
**Monitoring:** Upstash Dashboard
