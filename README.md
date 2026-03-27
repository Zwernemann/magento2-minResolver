# magento2-minResolver

A drop-in patch for Magento 2 that fixes intermittent 404 errors caused by
RequireJS loading plain `.js` files instead of `.min.js` when JavaScript
minification is enabled in the Admin panel.

---

## The Problem

### What you see

With **Minify JavaScript Files** enabled (Admin → Stores → Config → Advanced →
Developer → JavaScript Settings), Magento Admin works fine most of the time.
But after navigating back and forth several times — for example between
Sales → Orders and Create Order — the browser console suddenly shows:

```
GET /static/version.../adminhtml/.../mage/adminhtml/globals.js
    net::ERR_ABORTED 404 (Not Found)
```

The page may partially break: grids don't load, buttons stop working, or
JavaScript errors cascade. A hard reload fixes it temporarily — until it
happens again.

### Why only after several navigations?

This is not a timing race. It is a **lazy context creation** problem.

RequireJS internally manages one or more *contexts* — isolated module
registries, each with its own URL resolver. Magento ships a small script
called `requirejs-min-resolver.min.js` that is injected into every Admin page.
Its job is to force RequireJS to always request `.min.js` URLs.

The original implementation looks like this:

```javascript
(function() {
    var ctx = require.s.contexts._,   // only the default context
        origNameToUrl = ctx.nameToUrl,
        baseUrl = ctx.config.baseUrl;

    ctx.nameToUrl = function() {
        var url = origNameToUrl.apply(ctx, arguments);
        if (url.indexOf(baseUrl) === 0
                && !url.match(/\/hugerte\//)
                && !url.match(/\/v1\/songbird/)) {
            url = url.replace(/(\.min)?\.js$/, '.min.js');
        }
        return url;
    };
}());
```

This patches exactly **one** context: `_` (the default). That is sufficient for
simple page loads, which is why the bug does not appear immediately.

The problem surfaces when Magento creates a **new named context** — something
it does lazily, only when certain UI components initialise. The Sales order form,
for example, can trigger `require.s.newContext` or `require({context: 'order_'...})`
during its setup. That new context has never seen the resolver patch. It resolves
module URLs as plain `.js`, which do not exist on disk when minification is
enabled — resulting in a 404.

Because this depends on the exact sequence of pages visited, the threshold of
"x navigations" is not fixed. Some sessions stay clean for minutes; others
hit the bug on the third click.

---

## What This Module Does Differently

The original script has two gaps:

| Gap | Effect |
|-----|--------|
| Only patches the `_` context | Any named context is completely unaffected |
| No hook for future contexts | Contexts created after script-load time are never patched |

This module replaces the resolver with a single, self-contained IIFE that
closes both gaps through three layers:

```javascript
(function(){
  function patchCtx(c){
    if(!c||c.__mRF)return;   // idempotency guard
    c.__mRF=true;
    var p=c.nameToUrl;
    c.nameToUrl=function(){
      var u=p.apply(c,arguments),b=c.config&&c.config.baseUrl;
      if(b&&u.indexOf(b)===0
        &&!/\.min\.js$/.test(u)
        &&!/\/hugerte\//.test(u)
        &&!/\/v1\/songbird/.test(u)){
        u=u.replace(/\.js$/,'.min.js');
      }
      return u;
    };
  }

  // Layer 1 — patch every context that exists right now (including _)
  var ctxs=require.s.contexts;
  for(var n in ctxs){
    if(Object.prototype.hasOwnProperty.call(ctxs,n))patchCtx(ctxs[n]);
  }

  // Layer 2 — patch every context created from this point on
  if(!require.s.__mNCF){
    require.s.__mNCF=true;
    var oNC=require.s.newContext;
    require.s.newContext=function(){
      var c=oNC.apply(this,arguments);
      patchCtx(c);
      return c;
    };
  }

  // Layer 3 — rewrite the URL at the last possible moment before the XHR fires
  if(!require.__mRL){
    require.__mRL=true;
    var ol=require.load;
    require.load=function(c,m,u){
      var b=c&&c.config&&c.config.baseUrl;
      if(b&&u&&u.indexOf(b)===0
        &&/\.js$/.test(u)&&!/\.min\.js$/.test(u)
        &&!/\/hugerte\//.test(u)&&!/\/v1\/songbird/.test(u)){
        u=u.replace(/\.js$/,'.min.js');
      }
      return ol.call(this,c,m,u);
    };
  }

  console.info('[rjsFix] v6 active');
}());
```

### Layer overview

| Layer | How | What it catches |
|-------|-----|-----------------|
| Context loop | Iterates `require.s.contexts` at load time | All contexts already created, including `_` |
| `newContext` hook | Wraps `require.s.newContext` | Every context created after the script runs |
| `require.load` hook | Wraps the XHR dispatcher | Any URL that slipped through — last-resort safety net |

### Idempotency

Each layer is guarded by a flag (`__mRF`, `__mNCF`, `__mRL`) so that running
the script multiple times — or loading it from several contexts — never causes
double-patching.

### Exclusions

`hugerte` and `v1/songbird` are third-party libraries that ship **only** as
unminified JS. Their URLs are explicitly excluded from rewriting.

---

## Compatibility

- Magento 2.4.x (tested on 2.4.6 and 2.4.7)
- Theme-independent — works with any Admin theme
- No PHP, no `setup:di:compile`, no Composer dependency
- Only relevant when **Minify JavaScript Files** is enabled in Admin;
  has no effect when minification is off

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
