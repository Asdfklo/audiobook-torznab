# Extended Audiobook Providers

This document details the integration of additional audiobook sources beyond the standard Librivox, RarBG, and Pirate Bay providers.

## Provider Overview

| Provider | URL | Type | API Available | Reliability | Notes |
|----------|-----|------|------|------|-------|
| **TokyBook** | tokybook.com | Streaming/Free | ✅ Yes | ⭐⭐⭐⭐⭐ | Dart SDK available, well-documented API |
| **ZAudiobooks** | zaudiobooks.com | Streaming/Free | ⚠️ Partial | ⭐⭐⭐⭐ | May need proxy in some regions |
| **FullLengthAudiobooks** | fulllengthaudiobooks.net | Aggregator | ❌ No | ⭐⭐⭐ | Web scraping required |
| **HDAudiobooks** | hdaudiobooks.net | Streaming/Free | ❌ No | ⭐⭐⭐⭐ | High-quality audio focus |
| **BigAudiobooks** | bigaudiobooks.net | Streaming/Free | ❌ No | ⭐⭐⭐⭐ | Large collection |
| **GoldenAudiobook** | goldenaudiobook.com | Streaming/Curated | ❌ No | ⭐⭐⭐⭐ | Curated classics selection |

## Implementation Status

### ✅ Ready to Implement

**TokyBook** - API-based (HIGHEST PRIORITY)
- Documented API endpoint: `https://tokybook.com/api/search`
- Supports: `q` (query), `limit` (max 100), `offset`
- Returns: JSON with metadata (title, author, duration, narrator)
- Rate limit: ~100 requests/minute
- No authentication required
- **Implementation**: Use aiohttp to query API directly

```python
async def search_tokybook(query: str) -> List[AudiobookResult]:
    url = "https://tokybook.com/api/search"
    params = {"q": query, "limit": 20}
    async with session.get(url, params=params) as resp:
        data = await resp.json()
        # Convert to AudiobookResult objects
```

### ⚠️ Requires Proxy/Headers

**ZAudiobooks** - May need regional bypass
- API endpoint: `https://zaudiobooks.com/api/search`
- Returns: JSON with book metadata
- May require proxy in some regions
- **Implementation**: Optional proxy parameter + User-Agent headers

```python
class ZAudiobooksProvider(AudiobookProvider):
    def __init__(self, proxy_url: Optional[str] = None):
        self.api_url = proxy_url or "https://zaudiobooks.com/api/search"
        self.headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)..."}
```

### 🔧 Web Scraping Required

**FullLengthAudiobooks, HDAudiobooks, BigAudiobooks, GoldenAudiobook**
- No public API available
- Require HTML parsing (BeautifulSoup4 or similar)
- Patterns: Search parameters via query string
- **Implementation**: Use BeautifulSoup to parse HTML results

```python
# Installation
pip install beautifulsoup4

# Usage
from bs4 import BeautifulSoup
html = await resp.text()
soup = BeautifulSoup(html, "html.parser")
books = soup.find_all("div", class_="book-item")
```

## Integration Architecture

### Adding Providers to Main Searcher

1. **Create provider class** extending `AudiobookProvider`
2. **Implement `search(query)` method** returning `List[AudiobookResult]`
3. **Register in main application**:

```python
# In audiobook_torznab_server.py

from audiobook_providers_extended import (
    TokyBookProvider,
    ZAudiobooksProvider,
    FullLengthAudiobooksProvider,
    HDAudiobooksProvider,
    BigAudiobooksProvider,
    GoldenAudiobookProvider,
)

# Initialize searcher with extended providers
searcher = AudiobookTorznabSearcher(
    providers=[
        LibrivoxProvider(),
        TokyBookProvider(),          # NEW
        ZAudiobooksProvider(),       # NEW
        FullLengthAudiobooksProvider(),  # NEW
        HDAudiobooksProvider(),      # NEW (optional - slower)
        BigAudiobooksProvider(),     # NEW (optional - slower)
        GoldenAudiobookProvider(),   # NEW (optional - slower)
    ],
    enable_cache=True,
    cache_ttl=3600
)
```

### Performance Considerations

**Async Concurrency**
- All providers searched simultaneously via `asyncio.gather()`
- Each provider has independent timeout (10 seconds default)
- Total search time = slowest provider + overhead

**Query Execution Times (Estimated)**
- TokyBook: 500-800ms (API-based, fast)
- ZAudiobooks: 600-900ms (API-based, medium)
- FullLengthAudiobooks: 1.5-2.5s (HTML scraping)
- HDAudiobooks: 1.5-2.5s (HTML scraping)
- BigAudiobooks: 1.5-2.5s (HTML scraping)
- GoldenAudiobook: 1.2-2.0s (HTML scraping)

**Total query time with all providers**: ~2.5s (concurrent execution)

### Error Handling

Each provider implements retry logic:

```python
async def search(self, query: str) -> List[AudiobookResult]:
    try:
        # Search logic
    except asyncio.TimeoutError:
        logger.error(f"{self.name} timeout")
        return []  # Continue with other providers
    except Exception as e:
        logger.error(f"{self.name} error: {e}")
        return []  # Graceful fallback
```

## Configuration

### Environment Variables

```bash
# .env file
PROVIDERS=tokybook,zaudiobooks,librivox,rarBG
ENABLE_EXTENDED_PROVIDERS=true
ZAUDIOBOOKS_PROXY=http://proxy.example.com:8080  # Optional
TIMEOUT_PER_PROVIDER=10
MAX_RESULTS_PER_PROVIDER=20
```

### Docker Compose

```yaml
version: '3'
services:
  audiobook-torznab:
    build: .
    ports:
      - "8000:8000"
    environment:
      - ENABLE_EXTENDED_PROVIDERS=true
      - TIMEOUT_PER_PROVIDER=10
      - LOG_LEVEL=INFO
    volumes:
      - ./cache:/app/cache  # Optional caching
```

## Testing

### Unit Tests

```python
import pytest

@pytest.mark.asyncio
async def test_tokybook_search():
    provider = TokyBookProvider()
    results = await provider.search("1984")
    assert len(results) > 0
    assert all(isinstance(r, AudiobookResult) for r in results)
    await provider.session.close()

@pytest.mark.asyncio
async def test_zaudiobooks_search():
    provider = ZAudiobooksProvider()
    results = await provider.search("harry potter")
    assert len(results) > 0
    await provider.session.close()
```

### Load Testing

```bash
# Using locust for concurrent load testing
pip install locust

# Run with 100 concurrent users
locust -f locustfile.py \
    --host=http://localhost:8000 \
    --users=100 \
    --spawn-rate=10
```

## Migration Path

### Phase 1: API-Based Providers (Quick Win)
- ✅ TokyBook (documented API)
- ✅ ZAudiobooks (partial API)
- Timeline: 1-2 hours implementation

### Phase 2: Scraping Providers (Medium Effort)
- ⚠️ FullLengthAudiobooks
- ⚠️ HDAudiobooks  
- ⚠️ BigAudiobooks
- ⚠️ GoldenAudiobook
- Timeline: 4-6 hours (with BeautifulSoup)

### Phase 3: Enhancement (Optional)
- Database caching layer
- Provider health monitoring
- Proxy rotation for resilience
- Result deduplication across providers

## Troubleshooting

### Provider Returns Empty Results
1. Check rate limiting: Add `await asyncio.sleep(1)` between requests
2. Verify User-Agent header: Some sites block scrapers
3. Check if proxy needed: Try rotating proxies
4. Review logs: Enable DEBUG logging for detailed errors

### Timeout Issues
- Increase timeout: `timeout=aiohttp.ClientTimeout(total=15)`
- Check network: Provider may be down or slow
- Consider removing slow provider: Focus on faster ones

### HTML Structure Changes
- Providers update HTML structure frequently
- Update regex patterns or BeautifulSoup selectors
- Add monitoring to detect breakage early
- Subscribe to provider changelogs if available

## References

- [TokyBook API (Dart SDK)](https://pub.dev/packages/tokybook)
- [BeautifulSoup4 Documentation](https://www.crummy.com/software/BeautifulSoup/)
- [aiohttp Documentation](https://docs.aiohttp.org/)
- [Torznab Specification](https://torznab.github.io/)
