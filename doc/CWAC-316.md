# CWAC-316

This is still just rough notes.

Discover and document all locations where page loading and reloading happens

We want to verify

- Every time we load or reload a page, we wait for at least the window `onload`
  event

every thread has it's own Browser instance. every thread has exactly one Browser
instance

Want to document how our code interacts with the Browser global.

Browser is restarted between each base_url browser instance is shared global
state between all audit plugins

Gareth and Eoin do a detailed review of the logic around page loading and
reloading. Any improvements identified will be implemented as part of this
ticket unless they are big . In that case we will create and size a new ticket
for the change.

Eoin has a hacked version the cwac code that fixed all the false positive
color_contrast issues in CWAC-309. The very rough diff of those changes . will
be expanded upon to implement that fix properly.

Lifecycle of the browser instance

```
cwac.py initialises one Browser instance per thread.
Crawler runs `safe_restart() on the browser after processing each URL from the config.
```

? does crawler ever actually load the page?

TODO: does browser refresh() wait for onload? if so, do we need that delay?
after the refresh between viewports

Selenium page load strategies:
https://www.testmuai.com/blog/selenium-page-load-strategy/

https://www.selenium.dev/documentation/webdriver/waits/ official docs

We use the default which is to wait for `document.readyState = complete` (aka
the firing of the `load` event)

Consider adding implicitly wait

```
driver.implicitly_wait(2)
# default is 0
```

but I don't think this will do anything for axe-core which runs inside the JS
sandbox not in selenium

aside: we can set network throttling as an option to the chrome instance if we
need to throttle
https://www.selenium.dev/documentation/webdriver/browsers/chrome/
