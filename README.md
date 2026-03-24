# Zwernemann_MinResolver

Fixes intermittent `*.min.js` 404 errors in Magento 2 caused by two defects in the generated `requirejs-min-resolver.js`.

## The Problem

### Background: How Magento resolves `.min.js` files

In production mode (`MAGE_MODE=production`) Magento deploys minified JavaScript files alongside their non-minified originals (e.g. `globals.min.js` next to `globals.js`). To ensure RequireJS loads the `.min` variant, Magento generates a small script called `requirejs-min-resolver.js` via `Magento\Framework\RequireJs\Config::getMinResolverCode()`. This script monkey-patches RequireJS's internal `nameToUrl()` function — the function that translates a module identifier like `mage/adminhtml/globals` into a URL — so that it appends `.min` before the `.js` extension for any URL that starts with the deployment `baseUrl`.

The generated script looks like this:

```javascript
(function () {
    var ctx = require.s.contexts._,
        origNameToUrl = ctx.nameToUrl,
        baseUrl = ctx.config.baseUrl;          // (A) captured once

    ctx.nameToUrl = function() {
        var url = origNameToUrl.apply(ctx, arguments);
        if (url.indexOf(baseUrl) === 0) {      // (B) compared against stale value
            url = url.replace(/(.\.min)?\.js$/, '.min.js');
        }
        return url;
    };
})();
```

This looks straightforward, but it has two independent defects that both lead to the same symptom: a `.js` file is requested instead of its `.min.js` counterpart, which does not exist in production → HTTP 404.

---

### Defect 1: Stale `baseUrl` in Closure

At point **(A)** the IIFE captures `ctx.config.baseUrl` into a local variable. This happens exactly once, at the moment the script is first evaluated — typically during the initial page load.

The Magento Admin panel re-runs `require.config({ baseUrl: '...' })` during AJAX-based sub-page navigation (e.g. when opening the "New Order" or "Edit Order" flow). RequireJS applies this update to `ctx.config.baseUrl` internally, but the variable captured in the resolver's closure is **never updated** — it still holds the original value from page load.

When a module is then requested on the sub-page:
1. `origNameToUrl()` produces a URL based on the new `baseUrl`
2. The condition `url.indexOf(baseUrl) === 0` at **(B)** compares against the **old** `baseUrl`
3. The condition is `false` → the `.min.js` rewrite is skipped
4. RequireJS requests the bare `.js` file → 404 in production

**Concrete example:**
After navigating to `Sales > Orders > Create New Order`, the network tab shows:

```
GET /pub/static/adminhtml/Magento/backend/en_US/mage/adminhtml/globals.js   → 404
```

instead of the correct:

```
GET /pub/static/adminhtml/Magento/backend/en_US/mage/adminhtml/globals.min.js   → 200
```

---

### Defect 2: Double-Wrapping on AJAX Navigation

The Magento Admin panel uses a pattern where navigating between sub-pages does not trigger a full browser reload. Instead, it fetches new HTML fragments via XHR and injects them into the DOM. If the injected fragment contains a `<script>` block that re-includes `requirejs-min-resolver.js` (which happens on certain layout handles), the IIFE runs a **second time** in the same JavaScript runtime.

Each run of the IIFE:
1. Reads the **current** `ctx.nameToUrl` — which is already the patched function from the previous run
2. Stores it as `origNameToUrl` in a new closure
3. Replaces `ctx.nameToUrl` with a new wrapper that calls this already-wrapped version

After N navigations `ctx.nameToUrl` is a chain of N nested closures. Each wrapper in the chain holds an `origNameToUrl` reference captured at a different point in time. If RequireJS was reconfigured (new `baseUrl`, new `paths` mappings) between any two captures, the intermediate wrappers hold **divergent context snapshots**.

The practical effects are:

- **Wrong URLs:** An inner wrapper applies the `.min.js` rewrite using a `baseUrl` from two navigations ago, producing a double-rewritten path like `globals.min.min.js`.
- **Infinite recursion (rare):** If a re-loaded RequireJS resets `ctx.nameToUrl` to the native implementation while the outer wrapper still references a closure that calls back into the patched version, the call chain can loop until the stack overflows.
- **Silent mis-routing:** More commonly, modules resolve to URLs that exist but are the wrong version (cached non-min file served from a different path), causing subtle JS errors rather than outright 404s.

---

## The Fix

This module appends a second IIFE immediately after the original resolver in the generated output. It does **not** modify or remove the original resolver, so Magento's own exclude-list logic (CDN paths, custom exclusions from `dev/js/minify_exclude`) continues to work as before.

The appended IIFE:

1. **Idempotency guard (`__mRF` flag):** Before patching a context, checks `c.__mRF`. If already set, returns immediately. This makes re-execution on AJAX navigation a no-op, fully preventing double-wrapping.

2. **Dynamic `baseUrl`:** Inside the patched `nameToUrl`, reads `c.config.baseUrl` fresh at every call instead of using a closure-captured value. Any `require.config()` update is picked up automatically.

3. **`require.s.newContext` hook:** RequireJS can create additional named contexts at runtime (e.g. for bundles or isolated module scopes). Without this hook, those contexts would use the unpatched native `nameToUrl`. The hook wraps `require.s.newContext` so every new context is automatically patched — except the `$` context, which is created by `mage/requirejs/mixins.js` specifically to look up non-min mixin module paths and must not be rewritten.

Generated output appended to the original resolver:

```javascript
(function(){
    var a = function(c) {
        if (c.__mRF) return;           // idempotency: skip if already patched
        c.__mRF = true;
        var p = c.nameToUrl;
        c.nameToUrl = function() {
            var u = p.apply(c, arguments),
                b = c.config.baseUrl;  // read dynamically, never stale
            if (u.indexOf(b) === 0 && !/.min.js$/.test(u)) {
                u = u.replace(/.js$/, '.min.js');
            }
            return u;
        };
    };
    a(require.s.contexts._);           // patch the default '_' context
    var n = require.s.newContext;
    require.s.newContext = function(x) {
        var c = n.apply(this, arguments);
        if (x !== '$') { a(c); }       // patch all future contexts except '$'
        return c;
    };
})();
```

---

## Scope: Frontend and Backend

The plugin is registered in the **global `di.xml`** (not scoped to `adminhtml`), so it covers both areas:

| Area | Risk level | Reason |
|------|-----------|--------|
| **Admin panel** | **Critical** | AJAX sub-page navigation re-evaluates scripts and calls `require.config()` → both defects are actively triggered |
| **Frontend (Luma / Blank)** | Low | Standard navigation triggers full page reloads; the resolver is evaluated once per load → defects rarely manifest |
| **Frontend (Hyvä, PWA Studio, custom SPA)** | Medium to high | Same AJAX navigation patterns as the Admin → both defects can be triggered |

Registering globally is safe in all cases: the idempotency guard ensures the patch is applied at most once per context regardless of how many times the script is evaluated.

---

## Confirmed Magento Core Issues

| Issue | Description |
|-------|-------------|
| [#38829](https://github.com/magento/magento2/issues/38829) | JS Minification & RequireJS loading in production 2.4.7 — both `.min` and non-min files are requested simultaneously |
| [#38117](https://github.com/magento/magento2/issues/38117) | `requirejs-min-resolver.min.js` not generated on `setup:static-content:deploy` — navigation fails after first page load |

---

## Installation

### Via Composer

```bash
composer require zwernemann/magento2-minresolver
bin/magento module:enable Zwernemann_MinResolver
bin/magento setup:upgrade
bin/magento setup:static-content:deploy
bin/magento cache:flush
```

### Manually

Copy `app/code/Zwernemann/MinResolver` to your Magento installation, then:

```bash
bin/magento module:enable Zwernemann_MinResolver
bin/magento setup:upgrade
bin/magento setup:static-content:deploy
bin/magento cache:flush
```

## Compatibility

- Magento 2.3.x, 2.4.x
- PHP 7.4+

## License

MIT


## Contact

**Zwernemann Medienentwicklung**\
Martin Zwernemann\
79730 Murg, Germany

[To the website](https://www.zwernemann.de/)

If you have questions, problems, or ideas for new features – feel free to get in touch.
