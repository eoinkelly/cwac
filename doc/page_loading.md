# Page loading flow

This document explains when CWAC causes the Selenium browser to load or reload a
page during a crawl, and when it only reads from the page that is already
loaded.

The relevant control flow is split across two files:

- `src/crawler.py` decides which URL to audit next.
- `src/audit_manager.py` performs the actual browser navigations needed to run
  the audits.

## Summary

The crawler does not directly open pages in the browser. Instead, it selects a
URL, registers the audits for that URL, and then calls
`AuditManager.run_audits()`. The actual page loads happen inside `run_audits()`.

In practice, the same URL may be loaded multiple times:

1. Once for each viewport.
2. Once again for each audit run within that viewport.
3. Potentially another time after a `WebDriverException` if the browser is
   restarted and the URL is reloaded.

Between viewport passes, the current page is also refreshed once, except after
the last viewport.

## End-to-end sequence for one URL

For a single URL chosen by `Crawler.crawl()`:

1. `Crawler.crawl()` filters and normalises the URL.
2. `Crawler.crawl()` calls `register_audit_plugins()` for that URL.
3. `Crawler.crawl()` calls `audit_manager.run_audits()`.
4. `AuditManager.run_audits()` loops through configured viewports.
5. For each viewport, it sets the browser window size.
6. For each enabled audit in that viewport, it calls `self.browser.get(url)`.
7. After the load completes, it checks for anti-bot pages and optionally opens
   `<details>` elements.
8. The audit plugin runs against the currently loaded page.
9. After all audits for a viewport finish, `driver.refresh()` is called if
   another viewport still needs to run.
10. When all viewport passes are complete, `Crawler.crawl()` calls
    `get_links()`, which reads the current page source to discover more URLs.

## Places that trigger browser navigation

### `AuditManager.run_audits()`

This method is where normal page loading happens.

For each configured viewport:

1. `self.browser.set_window_size(...)` changes the browser size.
2. For each audit, `self.browser.get(audit['kwargs']['url'])` loads the page.

This means page loading is tied to audit execution, not to URL discovery in the
crawler. If three audit plugins run on two viewports, the same URL is loaded up
to six times before failure handling is considered.

The code comment explains why the reload happens after changing viewport size:
some websites return different content for different viewport dimensions, so the
page is re-requested to get the correct variant.

### Between viewport passes

After all audits for one viewport complete, `run_audits()` does this when there
is another viewport remaining:

1. `self.browser.driver.refresh()` refreshes the current page.
2. `time.sleep(self.config.delay_between_viewports)` waits for the browser to
   settle after the refresh.

This refresh happens once per viewport transition. It does not replace the next
viewport's later `self.browser.get(url)` calls; those still happen for each
audit in the next viewport.

### Recovery after `WebDriverException`

If an audit throws `selenium.common.exceptions.WebDriverException`,
`run_audits()` performs recovery:

1. `self.browser.safe_restart()` restarts the browser session.
2. `self.browser.set_window_size(...)` reapplies the current viewport.
3. `self.browser.get(audit['kwargs']['url'])` reloads the same URL.

If that reload fails, the audit is skipped and execution continues.

### Browser restarts between base URLs

`Crawler.iterate_through_base_urls()` calls `self.browser.safe_restart()` after
each top-level site crawl finishes.

This restart does not itself specify a new URL, but it resets browser state
between websites. The next actual page load still happens later inside
`AuditManager.run_audits()` when the next site's first audit starts.

## Places that read the current page without loading a new one

These calls inspect or use the page that is already loaded. They do not trigger
a navigation by themselves.

### `AuditManager.test_for_anti_bot()`

This reads:

1. `self.browser.driver.current_url`
2. `self.browser.driver.page_source`

It uses the current page after `self.browser.get(url)` has already loaded it.

### `AuditManager.check_for_details_elements()`

This inspects the DOM with `find_elements(...)` and may modify open `<details>`
elements with JavaScript. It does not navigate away or reload the page.

### `Crawler.get_links()`

This parses `self.browser.driver.page_source` with BeautifulSoup to extract
links from the page that was most recently loaded by
`AuditManager.run_audits()`.

`Crawler.get_links()` therefore depends on the browser still holding a useful
post-audit page state.

### `Crawler.handle_base_element()`

This calls `self.browser.get_base_uri()` to interpret relative links correctly.
It reads the current document state and does not itself load a page.

## Requests that are not Selenium page loads

`Crawler.crawl()` also performs non-browser network checks before an audit:

1. `is_url_allowed_by_robots_txt()` may fetch `robots.txt` with `requests`.
2. `process_url_headers(...)` may issue HEAD or HTTP requests for header
   validation.

These network requests affect crawl decisions, but they do not load pages into
the Selenium browser.

## Practical implications

1. A page is usually loaded inside `AuditManager`, not inside `Crawler`.
2. The same URL can be loaded many times when multiple audits and viewports are
   enabled.
3. A viewport transition currently causes both a refresh and later fresh
   `browser.get(...)` calls for the same URL.
4. Link extraction happens after audits complete and uses the browser's current
   page source rather than triggering a fresh navigation.
5. Browser restarts happen both during error recovery and between top-level
   sites, but the next explicit URL load still comes from `run_audits()`.
