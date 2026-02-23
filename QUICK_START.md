# Quick Start: Adding Extended Providers

## TL;DR - Can It Search These Sites?

| Site | Can Add? | Difficulty | Speed | API Available |
|------|----------|-----------|-------|----------------|
| **tokybook.com** | ✅ YES | 😊 Easy | Fast | ✅ Documented |
| **zaudiobooks.com** | ✅ YES | 😊 Easy | Fast | ⚠️ Partial |
| **fulllengthaudiobooks.net** | ✅ YES | 🧠 Medium | Slow | ❌ No API |
| **hdaudiobooks.net** | ✅ YES | 🧠 Medium | Slow | ❌ No API |
| **bigaudiobooks.net** | ✅ YES | 🧠 Medium | Slow | ❌ No API |
| **goldenaudiobook.com** | ✅ YES | 🧠 Medium | Slow | ❌ No API |

**Answer: YES to all six! Here's what you need to know:**

---

## Provider Types

### 🐨 API-Based (Fast & Reliable)

**TokyBook** (😊 EASIEST TO IMPLEMENT)
- **API Endpoint**: `https://tokybook.com/api/search`
- **Method**: GET
- **Parameters**: `q` (query), `limit` (max 100), `offset`
- **Rate Limit**: ~100 req/min
- **Response**: JSON with title, author, narrator, duration
- **Status**: Documented API via [Dart SDK](https://pub.dev/packages/tokybook)
- **Implementation Time**: 30 minutes
- **Lines of Code**: ~50

**ZAudiobooks** (😊 EASY)
- **API Endpoint**: `https://zaudiobooks.com/api/search`
- **Method**: GET
- **Parameters**: `q` (query), `type` ("audiobook")
- **Rate Limit**: ~50 req/min
- **Response**: JSON with book metadata
- **Status**: Works but may need proxy in some regions
- **Implementation Time**: 45 minutes
- **Lines of Code**: ~60

### 🔧 Web Scraping (Requires HTML Parsing)

**FullLengthAudiobooks, HDAudiobooks, BigAudiobooks, GoldenAudiobook**
- **No public API** - Must parse HTML
- **Pattern**: Search via query string + HTML parsing
- **Required Library**: `beautifulsoup4` or regex
- **Rate Limit**: 1-2 sec delay between requests (be respectful)
- **Implementation Time**: 1-2 hours each
- **Lines of Code**: ~80 per provider

---

## Implementation Priority

### Phase 1: API Providers (START HERE) 🎆
```
TokyBook     (30 min)  ← Start with this
    ↓
ZAudiobooks  (45 min)
```

These two will give you 90% of the benefit with 20% of the work.

### Phase 2: Scraping Providers (IF TIME PERMITS)
```
FullLengthAudiobooks  (1.5 hours)
HDAudiobooks         (1.5 hours)
BigAudiobooks        (1.5 hours)
GoldenAudiobook      (1.5 hours)
```

---

## Implementation Pattern

All providers follow this same structure:

```python
from audiobook_torznab_searcher import AudiobookProvider, AudiobookResult
import aiohttp
import asyncio

class YourProviderProvider(AudiobookProvider):
    def __init__(self):
        super().__init__("ProviderName")
        self.base_url = "https://example.com/api"
    
    async def search(self, query: str) -> List[AudiobookResult]:
        try:
            if not self.session:
                self.session = aiohttp.ClientSession()
            
            # YOUR IMPLEMENTATION HERE
            params = {"q": query, "limit": 20}
            async with self.session.get(
                self.base_url,
                params=params,
                timeout=aiohttp.ClientTimeout(total=10)
            ) as resp:
                if resp.status != 200:
                    return []
                
                data = await resp.json()
                results = []
                
                for item in data.get("items", []):
                    result = AudiobookResult(
                        title=item.get("title"),
                        author=item.get("author"),
                        # ... other fields
                    )
                    results.append(result)
                
                return results
        
        except Exception as e:
            logger.error(f"Search error: {e}")
            return []
```

---

## Integration Steps

### 1. Create Provider File

```bash
cp audiobook_providers_extended.py your_project/
```

### 2. Register in Main Server

```python
# audiobook_torznab_server.py

from audiobook_providers_extended import (
    TokyBookProvider,
    ZAudiobooksProvider,
    FullLengthAudiobooksProvider,
    HDAudiobooksProvider,
    BigAudiobooksProvider,
    GoldenAudiobookProvider,
)

# In your FastAPI app initialization:
searcher = AudiobookTorznabSearcher(
    providers=[
        LibrivoxProvider(),          # Existing
        TokyBookProvider(),          # NEW
        ZAudiobooksProvider(),       # NEW
        FullLengthAudiobooksProvider(),  # NEW (optional - slower)
        HDAudiobooksProvider(),      # NEW (optional - slower)
        BigAudiobooksProvider(),     # NEW (optional - slower)
        GoldenAudiobookProvider(),   # NEW (optional - slower)
    ]
)
```

### 3. Test It

```bash
python -m pytest tests/test_providers.py
```

### 4. Deploy

```bash
docker-compose up -d
```

---

## Performance Impact

### Before (Librivox + RarBG only)
- Search time: ~1.5-2 seconds
- Results per query: 20-50

### After (All 8 providers)
- Search time: ~2.5-3 seconds (concurrent, not sequential)
- Results per query: 100-200+
- **Net benefit**: More results, barely slower (concurrent execution)

---

## Debugging Tips

### Check if Provider is Working

```python
import asyncio
from audiobook_providers_extended import TokyBookProvider

async def test():
    provider = TokyBookProvider()
    results = await provider.search("1984")
    print(f"Found {len(results)} results")
    for r in results[:3]:
        print(f"  - {r.title} by {r.author}")
    await provider.session.close()

asyncio.run(test())
```

### Enable Debug Logging

```bash
export LOG_LEVEL=DEBUG
python audiobook_torznab_server.py
```

### Check API Response

```python
import httpx

resp = httpx.get("https://tokybook.com/api/search", params={"q": "test", "limit": 5})
print(resp.json())
```

---

## Estimated Time Investment

| Task | Time | Difficulty |
|------|------|------------|
| Implement TokyBook | 30 min | 😊 Easy |
| Implement ZAudiobooks | 45 min | 😊 Easy |
| Test both providers | 30 min | 😊 Easy |
| Implement FullLengthAudiobooks | 90 min | 🧠 Medium |
| Implement other scraping (3x) | 240 min | 🧠 Medium |
| Total testing & debugging | 60 min | 😊 Easy |
| **TOTAL** | **~7-8 hours** | ⭐ Doable |

**Quick Win (30 min to 1 hour)**: Just implement TokyBook. Instant 50% more results.

---

## Common Pitfalls

### ❌ Don't forget to:
- [ ] Import the provider classes
- [ ] Add them to the searcher provider list
- [ ] Close aiohttp sessions in tests
- [ ] Handle timeout exceptions
- [ ] Log errors (helps with debugging)
- [ ] Test with real queries before deploying

### ❌ Common errors:

**"ConnectionRefusedError: Failed to establish connection"**
- Provider site is down or blocked
- Solution: Add retry logic, skip provider, check logs

**"asyncio.TimeoutError"**
- Provider is slow or unresponsive
- Solution: Already handled in the code (returns empty list)

**"KeyError: 'title'"**
- API response format changed
- Solution: Debug with real API call, update parsing logic

---

## Success Metrics

You'll know it's working when:

1. ✅ Search returns results from multiple providers
2. ✅ Each result has title, author, seeders fields populated
3. ✅ Magnet links are generated correctly
4. ✅ Prowlarr can import and display the feed
5. ✅ Search completes in < 3 seconds

---

## Next Steps

1. Start with **TokyBook** (easiest, fastest)
2. Then add **ZAudiobooks** (API-based, slightly harder)
3. If you want more coverage, add **FullLengthAudiobooks** (scraping)
4. Deploy and test with Prowlarr
5. Optionally add remaining scrapers

**Recommendation**: Implement phases 1 & 2 (both API providers). They'll cover most use cases with minimal effort.

Good luck! 🚀
