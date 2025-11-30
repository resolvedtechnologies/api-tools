# Generic Web Scraper Guide with Browser Session Reuse

This repository previously contained Trimble-specific samples. To help you build a **generic, multi-site scraper** that can reuse cached Firefox logins, follow the outline below. Everything is vendor-neutral and focuses on reusable patterns.

## Architecture at a Glance
- **Fetcher layer**: Async HTTP client (e.g., `httpx`/`requests` or `aiohttp`) with retry/backoff, user-agent rotation, and optional proxy pools.
- **Renderer (optional)**: Headless browser path (Playwright) only when JavaScript rendering or cookie-based auth is required.
- **Parser layer**: Fast HTML parsing (`selectolax`/`lxml`) with reusable selector helpers and Pydantic models for validation/normalization.
- **Pipelines**: Output adapters for NDJSON/CSV, PostgreSQL (via upsert), or S3/MinIO Parquet.
- **Scheduler**: URL queue with per-host concurrency limits, retry strategy, and periodic recrawls.

## Reusing Cached Firefox Sessions
1. **Locate the profile**
   - Linux: `~/.mozilla/firefox/<profile>.default-release/`
   - macOS: `~/Library/Application Support/Firefox/Profiles/<profile>.default-release/`
   - Windows: `%APPDATA%\Mozilla\Firefox\Profiles\<profile>.default-release\`
2. **Read cookies**
   - Use the SQLite database `cookies.sqlite` in the profile. Example (Python):
     ```python
     import pathlib, sqlite3, http.cookiejar
     from contextlib import closing

     profile = pathlib.Path("~/.mozilla/firefox/abcd.default-release").expanduser()
     cookie_db = profile / "cookies.sqlite"
     jar = http.cookiejar.CookieJar()

     with closing(sqlite3.connect(cookie_db)) as db:
         for name, value, host, path, expiry in db.execute(
             "SELECT name, value, host, path, expiry FROM moz_cookies"
         ):
             jar.set_cookie(http.cookiejar.Cookie(
                 version=0, name=name, value=value, port=None, port_specified=False,
                 domain=host, domain_specified=True, domain_initial_dot=host.startswith('.'),
                 path=path, path_specified=True, secure=False, expires=expiry,
                 discard=False, comment=None, comment_url=None, rest={}, rfc2109=False,
             ))
     ```
3. **Attach to HTTP client**
   - `requests`: `session = requests.Session(); session.cookies = jar`
   - `httpx`: `client = httpx.Client(cookies=jar)`
4. **With Playwright**
   - Launch Firefox: `async with async_playwright() as p: browser = await p.firefox.launch_persistent_context(profile_path)`
   - The persistent context will load cached sessions (including logins) automatically.
   - Use per-domain throttling and respect robots.txt when rendering.

## Anti-Bot Hygiene
- Rotate UA strings and proxies; pace requests with jittered delays.
- Detect blocks (CAPTCHA pages, 429/403 patterns) and fall back to alternate proxies or the rendering path.
- Respect robots.txt and cache-friendly headers (`If-Modified-Since`, `If-None-Match`).

## Data Validation and Storage
- Normalize dates, prices, and URLs through Pydantic models before persistence.
- Use idempotent keys (URL hash) to deduplicate writes.
- Keep a crawl log table/file with run metadata (counts, failures).

## Quick Task Checklist
- [ ] Define target sites, fields, and crawl cadence.
- [ ] Implement HTTP fetcher with retries, UA/proxy rotation, and per-host rate limits.
- [ ] Add optional Playwright Firefox path using persistent profiles for cached logins.
- [ ] Build parser helpers + Pydantic models for each site.
- [ ] Wire pipelines (NDJSON/CSV, PostgreSQL, S3/Parquet) with upsert semantics.
- [ ] Add integration tests with recorded fixtures (`pytest-recording`/`vcrpy`).

This guide is intentionally generic so it can be applied to any site without Trimble-specific dependencies.
