# CWAC-316

## Actions proposed

I am proposing the following sequence of PRs for this ticket:

### PR: Better comments

Code comments have been improved as a result of the planning for this ticket.
Those changes will be shipped in a separate PR so they do not distract from
later (more consequential) changes.

Some error messages are also improved.

### PR: Remove dead code

The
[Browser.restart method](https://github.com/eoinkelly/cwac/blob/a98aedef25e8dec2a6b44fb8f4b451651c8aa3eb/src/browser.py#L118)
is never called and should be removed

The leftover code supporting Firefix can also be removed as part of this if the
PO approves.

### PR: Remove get_if_necessary()

Browser.get_if_necessary() introduces complexity

TODO: explain this better

### PR: Audit AuditManager refresh + sleep

Currently there is a webdriver `refresh()` plus a sleep happening at the end of
the audit manager. See
https://github.com/eoinkelly/cwac/blob/a98aedef25e8dec2a6b44fb8f4b451651c8aa3eb/src/audit_manager.py#L386

This seems like it might not be required. Make a decision on this. If it is not
required, remove it. If we are unsure about just removing it, we could wrap it
in a feature flag of some kind.

### PR: Audit the `sleep()` usage

Audit all use of `sleep` in the code. If it's not clear, document what it does.
Verify this sleep is still required.

## Open considerations

### No audits edge case

Seems like the crawler might fail if there are no audits configured. Are we
handling that correctly

## Closed considerations

### Consider adding implicitly wait

Selenium has implicit and explicit wait timeouts which can be set e.g.

```python
driver.implicitly_wait(2)
# default is 0
```

We run axe-core by injecting it into the JS on the page so these waits would not
help axe-core so we do not think setting Selenium waits will be impactful for
CWAC.

### Network throttling Chrome is possible

Unrelated to this ticket but nothing it here for visibility. We can set network
throttling as an option to the chrome instance if we need to throttle how we
load resources from a single page. More details in:
https://www.selenium.dev/documentation/webdriver/browsers/chrome/

## References

- [Selenium page load strategies](https://www.selenium.dev/documentation/webdriver/waits/)
  - We use the "default" page load strategy which is to wait for
    `document.readyState = complete` (aka the firing of the `load` event)
