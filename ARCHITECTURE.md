# Audiobook Torznab Architecture & Implementation Notes

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Prowlarr (Indexer UI)                        │
│                    http://localhost:9696                             │
└────────────────────────────────────────────────────────────────────┬┘
                              ↑
                              │ Generic Torznab
                              │ HTTP Requests
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│          Audiobook Torznab API Server (FastAPI)                     │
│                    http://localhost:8000                             │
│                                                                       │
│  Routes:                                                              │
│  ├─ GET /                    → HTML Setup Instructions               │
│  ├─ GET /api?t=caps          → Torznab Capabilities                 │
│  ├─ GET /api?t=search&q=...  → Search Results (RSS/XML)            │
│  ├─ GET /api?t=book&q=...    → Book Search Results                 │
│  └─ GET /health              → Health Check                         │
└────────────────────────────────────────┬──────────────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────────┐
                    ↓                    ↓                        ↓
        ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
        │ LibrivoxProvider │  │ TokyBookProvider │  │ RarBGProvider    │
        │  (Active)        │  │  (New - API)     │  │  (Optional)      │
        │                  │  │                  │  │                  │
        │ librivox.org/api │  │tokybook.com/api  │  │torrentapi.org    │
        └──────────────────┘  └──────────────────┘  └──────────────────┘
                    ↓                    ↓                        ↓
        ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
        │ List of Results  │  │ List of Results  │  │ List of Results  │
        │ (AudiobookResult)│  │ (AudiobookResult)│  │ (AudiobookResult)│
        └──────────────────┘  └──────────────────┘  └──────────────────┘

                    ↓                    ↓                        ↓
        ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
        │ZAudiobooksProvider│ │FullLengthAudio..│ │ HDAudiobooksProvider
        │ (New - API)       │ │ (New - Scraping) │ │ (New - Scraping)
        │                  │  │                  │  │                  │
        │zaudiobooks.com   │  │fulllengthaudio..│  │hdaudiobooks.net  │
        └──────────────────┘  └──────────────────┘  └──────────────────┘
                    │                    │                        │
                    └────────────────────┼────────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    ↓                    ↓                    ↓
        ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐
        │BigAudiobooks    │  │GoldenAudiobook  │  │PirateBayProvider│
        │  (New - Scraping)│  │ (New - Scraping) │  │ (Optional)      │
        │                  │  │                  │  │                 │
        │bigaudiobooks.net │  │goldenaudiobook. │  │thepiratebay.org │
        └──────────────────┘  └──────────────────┘  └─────────────────┘
                    │                    │                    │
                    └────────────────────┼────────────────────┘
                                         ↓
                    ┌──────────────────────────────────────────┐
                    │   TorznabFormatter                       │
                    │   (XML/RSS Formatting)                  │
                    │                                          │
                    │ Converts AudiobookResult objects         │
                    │ to Torznab-compliant RSS/XML            │
                    └──────────────────┬───────────────────────┘
                                       ↓
                    ┌──────────────────────────────────────────┐
                    │ Torznab RSS Feed (XML)                  │
                    │                                          │
                    │ <?xml version="1.0"?>                  │
                    │ <rss version="2.0">                     │
                    │   <channel>                              │
                    │     <item>                               │
                    │       <title>...</title>                 │
                    │       <torznab:attr name="...">          │
                    │   </channel>                             │
                    │ </rss>                                   │
                    └──────────────────────────────────────────┘
                                       ↑
                                       │
                    ┌──────────────────┴───────────────────┐
                    │                                      │
           Response to Prowlarr                  Parse by Radarr/Sonarr
```

## 📋 Data Flow

### Search Request Flow

```
1. User searches in Prowlarr
   ↓
2. Prowlarr sends: GET /api?t=search&q=Harry+Potter
   ↓
3. audiobook-torznab-server.py receives request
   ↓
4. Calls AudiobookTorznabSearcher.search(query)
   ↓
5. Searcher dispatches to all providers concurrently
   ├─ LibrivoxProvider.search()
   ├─ TokyBookProvider.search() [NEW]
   ├─ ZAudiobooksProvider.search() [NEW]
   ├─ FullLengthAudiobooksProvider.search() [NEW]
   ├─ HDAudiobooksProvider.search() [NEW]
   ├─ BigAudiobooksProvider.search() [NEW]
   ├─ GoldenAudiobookProvider.search() [NEW]
   └─ RarBGProvider.search()
   ↓
6. Each provider returns List[AudiobookResult]
   ↓
7. Results aggregated and sorted by seeders
   ↓
8. TorznabFormatter.format_results() converts to XML
   ↓
9. XML returned as HTTP response
   ↓
10. Prowlarr parses RSS feed and displays results
    ↓
11. User can download via magnet link
```

## 🔄 Component Relationships

```
AudiobookResult (dataclass)
├─ title: str
├─ author: str
├─ narrator: str
├─ description: str
├─ seeders: int
├─ leechers: int
├─ file_size: int
├─ magnet_url: str
├─ source: str
└─ Methods:
   └─ generate_magnet() → str

AudiobookProvider (abstract)
├─ name: str
├─ session: aiohttp.ClientSession
└─ Abstract method:
   └─ search(query: str) → List[AudiobookResult]

LibrivoxProvider (extends AudiobookProvider)
├─ base_url: "https://librivox.org/api/cache/books"
└─ search() → queries Librivox API

TokyBookProvider (extends AudiobookProvider) [NEW]
├─ base_url: "https://tokybook.com/api/search"
└─ search() → queries TokyBook API

ZAudiobooksProvider (extends AudiobookProvider) [NEW]
├─ base_url: "https://zaudiobooks.com/api/search"
└─ search() → queries ZAudiobooks API (with optional proxy)

FullLengthAudiobooksProvider (extends AudiobookProvider) [NEW]
├─ base_url: "https://fulllengthaudiobooks.net"
└─ search() → scrapes HTML results

HDAudiobooksProvider (extends AudiobookProvider) [NEW]
├─ base_url: "https://hdaudiobooks.net"
└─ search() → scrapes HTML results

BigAudiobooksProvider (extends AudiobookProvider) [NEW]
├─ base_url: "https://bigaudiobooks.net"
└─ search() → scrapes HTML results

GoldenAudiobookProvider (extends AudiobookProvider) [NEW]
├─ base_url: "https://goldenaudiobook.com"
└─ search() → scrapes HTML results

RarBGProvider (extends AudiobookProvider)
├─ base_url: "http://torrentapi.org/mozi_latest_torrents.php"
└─ search() → queries RarBG API

PirateBayProvider (extends AudiobookProvider)
├─ base_url: configurable proxy URL
└─ search() → searches The Pirate Bay

AudiobookTorznabSearcher
├─ providers: List[AudiobookProvider]
├─ search(query) → searches all providers concurrently
└─ Returns: formatted Torznab XML string

TorznabFormatter
├─ AUDIOBOOK_CATEGORY = "5020"
├─ format_results() → converts results to Torznab XML
└─ _add_torznab_attr() → adds Torznab attributes to items
```

## 🔌 Torznab Specification Compliance

### Capabilities (caps)

The API returns a `<caps>` XML document declaring:
- Supported functions: search, book, audio-search
- Maximum results limit: 100
- Default results: 50
- Categories: 5020 (Audio Books), 5050 (Podcasts)

### Search Results Format

Each result includes:
- Standard RSS fields: title, description, link, guid, pubDate
- Torznab attributes (via `<torznab:attr>`):
  - `seeders` - Number of seed peers
  - `peers` - Number of leechers
  - `size` - File size in bytes
  - `infohash` - SHA1 torrent hash
  - `magneturl` - Complete magnet URI
  - `category` - Torznab category code (5020)
  - `source` - Originating provider

### API Endpoints

```
GET /api?t=caps
├─ Response: Torznab capabilities XML
└─ Used for: Discovery, capability verification

GET /api?t=search&q={query}
├─ Response: Torznab search results RSS
└─ Used for: General text search

GET /api?t=book&q={query}
├─ Response: Torznab search results RSS
└─ Used for: Book-specific search (same as search for now)
```

## ⚡ Performance Considerations

### Provider Query Times (Estimated)

| Provider | Type | Speed | Reliability |
|----------|------|-------|-------------|
| TokyBook | API | 500-800ms | ⭐⭐⭐⭐⭐ |
| ZAudiobooks | API | 600-900ms | ⭐⭐⭐⭐ |
| Librivox | API | 700-1000ms | ⭐⭐⭐⭐⭐ |
| RarBG | API | 800-1200ms | ⭐⭐⭐⭐ |
| FullLengthAudiobooks | Scraping | 1.5-2.5s | ⭐⭐⭐ |
| HDAudiobooks | Scraping | 1.5-2.5s | ⭐⭐⭐⭐ |
| BigAudiobooks | Scraping | 1.5-2.5s | ⭐⭐⭐⭐ |
| GoldenAudiobook | Scraping | 1.2-2.0s | ⭐⭐⭐⭐ |

**Total query time (all concurrent)**: ~2.5s (limited by slowest provider)

### Concurrency
- All providers searched simultaneously via `asyncio.gather()`
- Each provider has own aiohttp session
- Timeout per provider: 10 seconds (configurable)
- Failed provider doesn't block others

### Caching Strategy (Future)
```python
# Could implement Redis caching
cache_key = hashlib.md5(query.encode()).hexdigest()
cached_result = redis_client.get(cache_key)
if cached_result:
    return cached_result
else:
    results = await search_all_providers()
    redis_client.setex(cache_key, 3600, results)  # Cache 1 hour
    return results
```

### Result Aggregation
```python
# Collect all results
all_results = []
for result in await asyncio.gather(*tasks):
    all_results.extend(result)

# Sort by seeders (best first)
all_results.sort(key=lambda x: x.seeders, reverse=True)

# Limit to max results
return all_results[:100]
```

## 🔐 Security & Error Handling

### Error Handling Pattern
```python
try:
    async with self.session.get(url, params=params) as resp:
        if resp.status != 200:
            logger.warning(f"API returned {resp.status}")
            return []
        data = await resp.json()
        # Process data
except asyncio.TimeoutError:
    logger.error(f"Timeout for provider {self.name}")
    return []
except Exception as e:
    logger.error(f"Search error: {e}")
    return []
```

### Rate Limiting (Best Practices)
- Librivox: ~100 requests/minute (no limit documented)
- TokyBook: ~100 requests/minute (estimated)
- ZAudiobooks: ~50 requests/minute (estimated)
- RarBG: 1 request/5 seconds per IP
- Scraping sites: Use delays between requests

Implement delays:
```python
import asyncio
await asyncio.sleep(5)  # Rate limiting
```

## 🚀 Scalability Improvements

### 1. Provider Pooling
```python
class ProviderPool:
    def __init__(self, max_providers=5):
        self.semaphore = asyncio.Semaphore(max_providers)
    
    async def search_limited(self, provider, query):
        async with self.semaphore:
            return await provider.search(query)
```

### 2. Result Caching with TTL
```python
from functools import lru_cache
from datetime import datetime, timedelta

@lru_cache(maxsize=100)
async def cached_search(query, ttl_seconds=3600):
    # Implementation with TTL
    pass
```

### 3. Database Backend
```python
# Store results in SQLite or PostgreSQL
class SearchCache:
    async def get(self, query):
        return await db.query(SearchResult).filter(query=query).all()
    
    async def save(self, query, results):
        await db.add_all([SearchResult(query=query, ...) for r in results])
```

## 🧪 Testing Strategy

### Unit Tests
```python
# Test AudiobookResult
def test_magnet_generation():
    result = AudiobookResult(title="Test", author="Author")
    assert result.generate_magnet().startswith("magnet:")

# Test Formatter
def test_torznab_xml():
    results = [AudiobookResult(...)]
    xml = TorznabFormatter.format_results(results, "test")
    assert "<?xml" in xml
    assert "<item>" in xml

# Test Extended Providers
@pytest.mark.asyncio
async def test_tokybook_search():
    provider = TokyBookProvider()
    results = await provider.search("1984")
    assert len(results) > 0
    await provider.session.close()
```

### Integration Tests
```python
# Test with real providers
async def test_all_providers():
    providers = [
        LibrivoxProvider(),
        TokyBookProvider(),
        ZAudiobooksProvider(),
    ]
    results = await asyncio.gather(*[p.search("test") for p in providers])
    assert all(len(r) >= 0 for r in results)
```

### Load Tests
```bash
# Using locust
locust -f locustfile.py --host=http://localhost:8000
```

## 📝 Configuration Examples

### Docker Compose
```yaml
version: '3'
services:
  audiobook-torznab:
    build: .
    ports:
      - "8000:8000"
    environment:
      - LOG_LEVEL=INFO
      - CACHE_TTL=3600
      - ENABLE_EXTENDED_PROVIDERS=true
      - TIMEOUT_PER_PROVIDER=10
  
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    ports:
      - "9696:9696"
    depends_on:
      - audiobook-torznab
```

### Environment Variables
```bash
# .env file
LOG_LEVEL=INFO
MAX_RESULTS=100
TIMEOUT_PER_PROVIDER=10
ENABLE_CACHE=true
CACHE_TTL=3600
BIND_ADDRESS=0.0.0.0
BIND_PORT=8000
ENABLE_EXTENDED_PROVIDERS=true
ZAUDIOBOOKS_PROXY=  # Optional proxy for ZAudiobooks
```

## 🔮 Future Enhancements

1. **Database Backend** - Store and cache results
2. **Authentication** - API key support for rate limiting
3. **Advanced Filtering** - By narrator, language, year
4. **Result Scoring** - Weighted by seeders, source reliability
5. **Proxy Support** - Rotate through proxies for reliability
6. **Webhook Notifications** - Alert on new releases
7. **Web UI** - Dashboard for management and statistics
8. **Metrics Collection** - Prometheus-compatible metrics endpoint
9. **Provider Health Monitoring** - Detect broken providers
10. **Result Deduplication** - Across multiple providers

---

**Architecture Version:** 2.0.0 (Extended Providers)
**Last Updated:** February 23, 2026
**New Providers:** 6 (TokyBook, ZAudiobooks, FullLengthAudiobooks, HDAudiobooks, BigAudiobooks, GoldenAudiobook)
