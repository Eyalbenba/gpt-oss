# Tavily Integration Guide for SimpleBrowserTool

## Overview

This guide explains how to integrate Tavily's Search and Extract APIs into the SimpleBrowserTool backend, replacing the Exa implementation.

## API Endpoints

### Base URL
```
https://api.tavily.com
```

### Endpoints
- **Search**: `POST /search`
- **Extract**: `POST /extract`

## Authentication

Tavily uses Bearer token authentication:

```python
headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type": "application/json"
}
```

The API key should be prefixed with `tvly-` and can be set via:
- Environment variable: `TAVILY_API_KEY`
- Direct parameter: `api_key`

## Search API Implementation

### Request Format

```python
async def search(self, query: str, topn: int, session: ClientSession) -> PageContents:
    payload = {
        "query": query,
        "max_results": topn,
        "include_raw_content": "text",  
    }
    
    headers = {
        "Authorization": f"Bearer {self._get_api_key()}",
        "Content-Type": "application/json"
    }
    
    async with session.post(
        f"{self.BASE_URL}/search",
        json=payload,
        headers=headers
    ) as resp:
        if resp.status != 200:
            raise BackendError(f"Tavily API error {resp.status}: {await resp.text()}")
        data = await resp.json()
```

### Response Format

```json
{
    "query": "search query",
    "answer": "AI-generated answer to the query",
    "images": ["url1", "url2"],
    "results": [
        {
            "title": "Page Title",
            "url": "https://example.com",
            "content": "Relevant text content from the page",
            "score": 0.95,
            "favicon": "https://example.com/favicon.ico"
        }
    ]
}
```

### Converting to PageContents

```python
# Convert Tavily results to HTML format for compatibility
results = data.get("results", [])
answer = data.get("answer", "")

# Build HTML with answer at top if available
html_parts = ["<html><body>"]

if answer:
    html_parts.append(f"<div><h2>Answer</h2><p>{answer}</p></div>")

html_parts.append("<h1>Search Results</h1><ul>")

for i, result in enumerate(results):
    title = result.get("title", "Untitled")
    url = result.get("url", "")
    content = result.get("content", "")
    
    # Include snippet in the list item
    html_parts.append(
        f'<li><a href="{url}">{title}</a><br/>{content[:200]}...</li>'
    )

html_parts.extend(["</ul>", "</body></html>"])
html_page = "".join(html_parts)

return process_html(
    html=html_page,
    url="",
    title=f"Search: {query}",
    display_urls=True,
    session=session,
)
```

## Extract API Implementation

### Request Format

```python
async def fetch(self, url: str, session: ClientSession) -> PageContents:
    is_view_source = url.startswith(VIEW_SOURCE_PREFIX)
    if is_view_source:
        url = url[len(VIEW_SOURCE_PREFIX):]
    
    payload = {
        "urls": [url]
    }
    
    headers = {
        "Authorization": f"Bearer {self._get_api_key()}",
        "Content-Type": "application/json"
    }
    
    async with session.post(
        f"{self.BASE_URL}/extract",
        json=payload,
        headers=headers
    ) as resp:
        if resp.status != 200:
            raise BackendError(f"Tavily API error {resp.status}: {await resp.text()}")
        data = await resp.json()
```

### Response Format

```json
{
    "results": [
        {
            "url": "https://example.com",
            "raw_content": "Full page text content",
            "metadata": {
                "title": "Page Title",
                "description": "Page description",
                "favicon": "https://example.com/favicon.ico"
            }
        }
    ]
}
```

### Converting to PageContents

```python
results = data.get("results", [])
if not results:
    raise BackendError(f"No content returned for {url}")

result = results[0]
content = result.get("raw_content", "")
metadata = result.get("metadata", {})
title = metadata.get("title", "")

# Since Tavily returns plain text, we need to wrap it in minimal HTML
# to maintain compatibility with the existing process_html function
html_content = f"""
<html>
<head><title>{title}</title></head>
<body>
<pre>{content}</pre>
</body>
</html>
"""

return process_html(
    html=html_content,
    url=url,
    title=title,
    display_urls=True,
    session=session,
)
```

## Complete TavilyBackend Implementation

```python
@chz.chz(typecheck=True)
class TavilyBackend(Backend):
    """Backend that uses the Tavily Search API."""

    source: str = chz.field(doc="Description of the backend source")
    api_key: str | None = chz.field(
        doc="Tavily API key. Uses TAVILY_API_KEY environment variable if not provided.",
        default=None,
    )

    BASE_URL: str = "https://api.tavily.com"

    def _get_api_key(self) -> str:
        key = self.api_key or os.environ.get("TAVILY_API_KEY")
        if not key:
            raise BackendError("Tavily API key not provided")
        return key

    def _get_headers(self) -> dict:
        return {
            "Authorization": f"Bearer {self._get_api_key()}",
            "Content-Type": "application/json"
        }

    async def search(
        self, query: str, topn: int, session: ClientSession
    ) -> PageContents:
        payload = {
            "query": query,
            "max_results": topn,
            "include_answer": True,
            "include_raw_content": False,
            "search_depth": "basic"
        }
        
        async with session.post(
            f"{self.BASE_URL}/search",
            json=payload,
            headers=self._get_headers()
        ) as resp:
            if resp.status != 200:
                raise BackendError(
                    f"Tavily API error {resp.status}: {await resp.text()}"
                )
            data = await resp.json()
        
        # Convert Tavily format to HTML for compatibility
        results = data.get("results", [])
        answer = data.get("answer", "")
        
        html_parts = ["<html><body>"]
        
        if answer:
            html_parts.append(f"<div><h2>Summary</h2><p>{answer}</p></div><hr/>")
        
        html_parts.append("<h1>Search Results</h1><ul>")
        
        for result in results:
            title = result.get("title", "Untitled")
            url = result.get("url", "")
            content = result.get("content", "")
            
            html_parts.append(
                f'<li><a href="{url}">{title}</a><br/>'
                f'<small>{content[:200]}...</small></li>'
            )
        
        html_parts.extend(["</ul>", "</body></html>"])
        html_page = "".join(html_parts)
        
        return process_html(
            html=html_page,
            url="",
            title=f"Search: {query}",
            display_urls=True,
            session=session,
        )

    async def fetch(self, url: str, session: ClientSession) -> PageContents:
        is_view_source = url.startswith(VIEW_SOURCE_PREFIX)
        if is_view_source:
            url = url[len(VIEW_SOURCE_PREFIX):]
        
        payload = {"urls": [url]}
        
        async with session.post(
            f"{self.BASE_URL}/extract",
            json=payload,
            headers=self._get_headers()
        ) as resp:
            if resp.status != 200:
                raise BackendError(
                    f"Tavily API error {resp.status}: {await resp.text()}"
                )
            data = await resp.json()
        
        results = data.get("results", [])
        if not results:
            raise BackendError(f"No content returned for {url}")
        
        result = results[0]
        content = result.get("raw_content", "")
        metadata = result.get("metadata", {})
        title = metadata.get("title", "")
        
        # Wrap plain text in HTML for compatibility
        html_content = f"""
        <html>
        <head><title>{title}</title></head>
        <body>
        <article>{content}</article>
        </body>
        </html>
        """
        
        return process_html(
            html=html_content,
            url=url,
            title=title,
            display_urls=True,
            session=session,
        )
```

## Key Differences from Exa

1. **Authentication**: Tavily uses `Bearer` token format vs Exa's `x-api-key` header
2. **Response Format**: Tavily provides structured JSON with `answer` field
3. **Content Type**: Tavily returns plain text content, not HTML
4. **No HTML Tags**: Tavily's Extract API returns clean text, eliminating the need for HTML parsing

## Usage Example

```python
from gpt_oss.tools.simple_browser import SimpleBrowserTool, TavilyBackend

# Initialize with Tavily backend
backend = TavilyBackend(source="web")
browser_tool = SimpleBrowserTool(backend=backend)

# Use in system message
system_content = system_content.with_tools(browser_tool.tool_config)
```

## Environment Setup

```bash
export TAVILY_API_KEY="tvly-YOUR_API_KEY_HERE"
```

## Advantages

1. **Cleaner Content**: Tavily returns pre-processed text, reducing processing overhead
2. **AI-Optimized**: Designed specifically for LLM consumption
3. **Answer Generation**: Provides AI-generated summaries for search queries
4. **Simpler Integration**: Less HTML parsing required

## Error Handling

Common error codes:
- `401`: Invalid API key
- `429`: Rate limit exceeded
- `500`: Server error

Always wrap API calls in try-except blocks and provide meaningful error messages to the user.