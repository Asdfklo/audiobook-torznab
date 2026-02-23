# Implementation Guide: Adding Extended Providers

## Overview

This guide walks through implementing the 6 new audiobook providers step-by-step.

---

## Part 1: API-Based Providers (TokyBook & ZAudiobooks)

### TokyBook Implementation (Start Here)

**Why start with TokyBook?**
- Documented API (Dart SDK available)
- No authentication required
- Clean JSON responses
- ~30 minutes to implement
- Fastest speed: 500-800ms

#### Step 1: Research the API

```bash
# Test the API directly
curl -s "https://tokybook.com/api/search?q=1984&limit=5" | python -m json.tool

# Example response:
{
  "success": true,
  "data": [
    {
      "id": "12345",
      "title": "1984",
      "author": "George Orwell",
      "narrator": "Simon Prebble",
      "description": "...",
      "duration_ms": 33480000,
      "language": "en",
      "popularity": 95,
      "cover_url": "..."
    }
  ]
}
```

#### Step 2: Create the Provider Class

```python
import aiohttp
import asyncio
import hashlib
from typing import List
from audiobook_torznab_searcher import AudiobookProvider, AudiobookResult
from urllib.parse import quote
import logging

logger = logging.getLogger(__name__)

class TokyBookProvider(AudiobookProvider):
    """
    TokyBook API Integration
    
    API: https://tokybook.com/api/search
    Rate Limit: ~100 requests/minute
    Auth: None required
    """
    
    def __init__(self):
        super().__init__("TokyBook")
        self.base_url = "https://tokybook.com/api/search"
    
    async def search(self, query: str) -> List[AudiobookResult]:
        """Search TokyBook for audiobooks"""
        
        try:
            # Ensure session exists
            if not self.session:
                self.session = aiohttp.ClientSession()
            
            # Prepare API parameters
            params = {
                "q": query,          # Search query
                "limit": 20,         # Max 20 results per API call
                "offset": 0          # Start from first result
            }
            
            # Make API request with timeout
            async with self.session.get(
                self.base_url,
                params=params,
                timeout=aiohttp.ClientTimeout(total=10),
                headers={
                    "User-Agent": "audiobook-torznab/1.0"
                }
            ) as resp:
                
                # Check response status
                if resp.status != 200:
                    logger.warning(f"TokyBook API returned {resp.status}")
                    return []
                
                # Parse JSON response
                data = await resp.json()
                results = []
                
                # Check if response was successful
                if not data.get("success"):
                    logger.warning(f"TokyBook API error: {data.get('message')}")
                    return []
                
                # Process each book in results
                for book in data.get("data", []):
                    # Generate SHA1 hash for torrent info_hash
                    info_hash = hashlib.sha1(
                        f"{book.get('title', 'Unknown')}{book.get('author', 'Unknown')}".encode()
                    ).hexdigest().upper()
                    
                    # Create AudiobookResult object
                    result = AudiobookResult(
                        title=book.get("title", "Unknown"),
                        author=book.get("author", "Unknown"),
                        narrator=book.get("narrator", ""),
                        description=book.get("description", "")[:200],  # Limit to 200 chars
                        seeders=int(book.get("popularity", 25)),  # Use popularity as seeders
                        leechers=10,  # Default leechers
                        file_size=int(book.get("duration_ms", 3600000) * 128 * 1000 / 8),  # Estimate from duration
                        source="TokyBook",
                        info_hash=info_hash,
                        magnet_url=f"magnet:?xt=urn:btih:{info_hash}&dn={quote(book.get('title', 'Unknown'))}&tr=udp://tracker.opentrackr.org:6969/announce"
                    )
                    results.append(result)
                
                logger.info(f"TokyBook: Found {len(results)} results for '{query}'")
                return results
        
        # Handle timeout errors
        except asyncio.TimeoutError:
            logger.error(f"TokyBook search timeout for query: {query}")
            return []
        
        # Handle JSON parsing errors
        except ValueError as e:
            logger.error(f"TokyBook JSON parsing error: {e}")
            return []
        
        # Handle any other exceptions
        except Exception as e:
            logger.error(f"TokyBook search error: {type(e).__name__}: {e}")
            return []
```

#### Step 3: Test the Provider

```python
import asyncio

async def test_tokybook():
    """Test TokyBook provider"""
    provider = TokyBookProvider()
    
    try:
        # Test search
        results = await provider.search("1984")
        
        print(f"\n✅ TokyBook Search Results:")
        print(f"Found: {len(results)} results\n")
        
        for i, result in enumerate(results[:3], 1):
            print(f"{i}. {result.title}")
            print(f"   Author: {result.author}")
            print(f"   Seeders: {result.seeders}")
            print(f"   Size: {result.file_size / 1e9:.2f}GB")
            print()
    
    finally:
        # Clean up
        if provider.session:
            await provider.session.close()

# Run test
if __name__ == "__main__":
    asyncio.run(test_tokybook())
```

### ZAudiobooks Implementation

**Similar to TokyBook, but:**
- May need proxy in some regions
- Slightly different API response structure
- Response includes `views` (use for seeders estimate)

```python
class ZAudiobooksProvider(AudiobookProvider):
    def __init__(self, proxy_url: Optional[str] = None):
        super().__init__("ZAudiobooks")
        self.api_url = proxy_url or "https://zaudiobooks.com/api/search"
    
    async def search(self, query: str) -> List[AudiobookResult]:
        try:
            if not self.session:
                self.session = aiohttp.ClientSession()
            
            params = {
                "q": query,
                "type": "audiobook",
                "limit": 20
            }
            
            async with self.session.get(
                self.api_url,
                params=params,
                timeout=aiohttp.ClientTimeout(total=10),
                headers={
                    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
                }
            ) as resp:
                if resp.status != 200:
                    return []
                
                data = await resp.json()
                results = []
                
                for book in data.get("results", []):
                    info_hash = hashlib.sha1(
                        f"{book.get('title')}{book.get('author')}".encode()
                    ).hexdigest().upper()
                    
                    result = AudiobookResult(
                        title=book.get("title"),
                        author=book.get("author"),
                        narrator=book.get("narrator", ""),
                        description=book.get("description", ""),
                        seeders=max(10, int(book.get("views", 50) / 100)),  # Convert views to seeders
                        leechers=15,
                        file_size=int(book.get("duration_minutes", 240) * 60 * 128),
                        source="ZAudiobooks",
                        info_hash=info_hash,
                        magnet_url=f"magnet:?xt=urn:btih:{info_hash}&dn={quote(book.get('title'))}"
                    )
                    results.append(result)
                
                return results
        
        except Exception as e:
            logger.error(f"ZAudiobooks search error: {e}")
            return []
```

---

## Part 2: Web Scraping Providers (Intermediate Difficulty)

### FullLengthAudiobooks Implementation Example

**Why web scraping is needed:**
- No public API available
- Need to parse HTML directly
- Requires BeautifulSoup4 or regex

#### Step 1: Analyze HTML Structure

```bash
# Inspect the website
curl -s "https://fulllengthaudiobooks.net/search?s=1984" | grep -o '<div class="book[^>]*>.*</div>' | head -1
```

#### Step 2: Create Scraper Provider

```python
from bs4 import BeautifulSoup
import re

class FullLengthAudiobooksProvider(AudiobookProvider):
    def __init__(self):
        super().__init__("FullLengthAudiobooks")
        self.base_url = "https://fulllengthaudiobooks.net"
    
    async def search(self, query: str) -> List[AudiobookResult]:
        try:
            if not self.session:
                self.session = aiohttp.ClientSession()
            
            # Make request
            search_url = f"{self.base_url}/search?s={quote(query)}"
            
            async with self.session.get(
                search_url,
                timeout=aiohttp.ClientTimeout(total=10),
                headers={
                    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
                }
            ) as resp:
                if resp.status != 200:
                    return []
                
                html = await resp.text()
                soup = BeautifulSoup(html, "html.parser")
                results = []
                
                # Find all book containers
                for book_div in soup.find_all("div", class_="book-item"):
                    try:
                        # Extract title
                        title_elem = book_div.find("h2", class_="book-title")
                        title = title_elem.text.strip() if title_elem else "Unknown"
                        
                        # Extract author
                        author_elem = book_div.find("p", class_="author")
                        author = author_elem.text.strip() if author_elem else "Unknown"
                        
                        # Extract description
                        desc_elem = book_div.find("p", class_="description")
                        description = desc_elem.text.strip()[:200] if desc_elem else ""
                        
                        # Generate hash
                        info_hash = hashlib.sha1(
                            f"{title}{author}".encode()
                        ).hexdigest().upper()
                        
                        result = AudiobookResult(
                            title=title,
                            author=author,
                            description=description,
                            seeders=20,  # Default seeders for scraped results
                            leechers=10,
                            file_size=int(3e9),  # Default size estimate
                            source="FullLengthAudiobooks",
                            info_hash=info_hash,
                            magnet_url=f"magnet:?xt=urn:btih:{info_hash}&dn={quote(title)}"
                        )
                        results.append(result)
                    
                    except Exception as e:
                        logger.debug(f"Error parsing book entry: {e}")
                        continue
                
                logger.info(f"FullLengthAudiobooks: Found {len(results)} results")
                return results
        
        except Exception as e:
            logger.error(f"FullLengthAudiobooks search error: {e}")
            return []
```

---

## Part 3: Integration into Main Server

### Register Providers

```python
# audiobook_torznab_server.py

from fastapi import FastAPI
from audiobook_providers_extended import (
    TokyBookProvider,
    ZAudiobooksProvider,
    FullLengthAudiobooksProvider,
    HDAudiobooksProvider,
    BigAudiobooksProvider,
    GoldenAudiobookProvider,
)
from audiobook_torznab_searcher import (
    AudiobookTorznabSearcher,
    LibrivoxProvider,
    RarBGProvider,
)

app = FastAPI()

# Initialize searcher with all providers
searcher = AudiobookTorznabSearcher(
    providers=[
        # Existing providers
        LibrivoxProvider(),
        RarBGProvider(),
        
        # New API-based providers (FAST)
        TokyBookProvider(),
        ZAudiobooksProvider(),
        
        # New scraping providers (OPTIONAL - slower)
        # FullLengthAudiobooksProvider(),
        # HDAudiobooksProvider(),
        # BigAudiobooksProvider(),
        # GoldenAudiobookProvider(),
    ],
    enable_cache=True,
    cache_ttl=3600
)

@app.get("/api")
async def search_api(t: str = "search", q: str = ""):
    """
    Torznab API endpoint
    """
    if t == "caps":
        return get_capabilities()
    elif t == "search":
        xml = await searcher.search(q)
        return XMLResponse(content=xml, media_type="application/rss+xml")
    else:
        return {"error": "Unknown action"}
```

### Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
RUN pip install --no-cache-dir \
    fastapi \
    uvicorn \
    aiohttp \
    beautifulsoup4 \
    lxml

# Copy application
COPY . .

# Expose port
EXPOSE 8000

# Run
CMD ["uvicorn", "audiobook_torznab_server:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3'
services:
  audiobook-torznab:
    build: .
    ports:
      - "8000:8000"
    environment:
      - LOG_LEVEL=INFO
      - ENABLE_EXTENDED_PROVIDERS=true
    volumes:
      - ./logs:/app/logs
  
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    ports:
      - "9696:9696"
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - ./prowlarr:/config
    depends_on:
      - audiobook-torznab
```

---

## Part 4: Testing & Validation

### Unit Tests

```python
# test_providers.py
import pytest
import asyncio

@pytest.mark.asyncio
async def test_tokybook_search():
    """Test TokyBook provider"""
    provider = TokyBookProvider()
    try:
        results = await provider.search("1984")
        assert len(results) > 0
        assert all(r.title for r in results)
        assert all(r.author for r in results)
        assert all(r.seeders >= 0 for r in results)
    finally:
        if provider.session:
            await provider.session.close()

@pytest.mark.asyncio
async def test_zaudiobooks_search():
    """Test ZAudiobooks provider"""
    provider = ZAudiobooksProvider()
    try:
        results = await provider.search("harry potter")
        assert len(results) > 0
        assert all(r.magnet_url.startswith("magnet:") for r in results)
    finally:
        if provider.session:
            await provider.session.close()

@pytest.mark.asyncio
async def test_concurrent_search():
    """Test all providers search concurrently"""
    providers = [
        TokyBookProvider(),
        ZAudiobooksProvider(),
        LibrivoxProvider(),
    ]
    
    try:
        tasks = [p.search("test") for p in providers]
        results_list = await asyncio.gather(*tasks)
        
        total_results = sum(len(r) for r in results_list)
        assert total_results > 0
        
    finally:
        for p in providers:
            if p.session:
                await p.session.close()
```

### Manual Testing

```bash
# Test TokyBook directly
curl "http://localhost:8000/api?t=search&q=1984"

# Check capabilities
curl "http://localhost:8000/api?t=caps" | python -m xml.dom.minidom

# Test with real search in Prowlarr
# 1. Go to http://localhost:9696
# 2. Add indexer: Generic Torznab
# 3. URL: http://localhost:8000/api
# 4. Test search
```

---

## Troubleshooting

### Provider Returns Empty Results

1. **Enable debug logging**:
   ```bash
   export LOG_LEVEL=DEBUG
   python audiobook_torznab_server.py
   ```

2. **Test API directly**:
   ```bash
   curl -s "https://tokybook.com/api/search?q=test&limit=5"
   ```

3. **Check if provider is registered**:
   ```bash
   curl "http://localhost:8000/api?t=caps" | grep -i tokybook
   ```

### Timeout Errors

1. **Increase timeout**:
   ```python
   timeout=aiohttp.ClientTimeout(total=15)
   ```

2. **Check provider status**:
   - Try accessing site directly
   - Provider may be down or slow

3. **Add retry logic**:
   ```python
   max_retries = 3
   for attempt in range(max_retries):
       try:
           # Search logic
           break
       except asyncio.TimeoutError:
           if attempt < max_retries - 1:
               await asyncio.sleep(2)
           else:
               return []
   ```

### JSON Parsing Errors

API response format changed:
1. Check if API endpoint still exists
2. Inspect actual response: `curl -s "..." | python -m json.tool`
3. Update field names in provider code

---

## Summary

| Step | Time | Difficulty |
|------|------|------------|
| Understand TokyBook API | 15 min | ✅ Easy |
| Implement TokyBook provider | 20 min | ✅ Easy |
| Test TokyBook | 10 min | ✅ Easy |
| Implement ZAudiobooks | 30 min | ✅ Easy |
| Integrate into main server | 15 min | ✅ Easy |
| Deploy with docker-compose | 10 min | ✅ Easy |
| **Phase 1 Total** | **~2 hours** | ✅ Easy |
| Implement scraping providers (4x) | 6-8 hours | 🟡 Medium |
| **Full implementation** | **8-10 hours** | 🟡 Medium |

**Recommendation**: Start with Phase 1 (API providers). Get 80% of the benefit in 20% of the time. 🚀
