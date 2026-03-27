# magento2-minResolver


A drop-in patch for Magento 2 that fixes intermittent 404 errors caused by
RequireJS loading plain `.js` files instead of `.min.js` when JavaScript
minification is enabled in the Admin panel.

Tested on Magento **2.4.7-p9**.

---

## The Problem

### What you see

With **Minify JavaScript Files** enabled
(Admin → Stores → Config → Advanced → Developer → JavaScript Settings),
Magento Admin works fine most of the time. But after navigating back and forth
several times — for example between Sales → Orders and Create Order — the
browser console suddenly shows:

```
GET /static/version.../adminhtml/.../mage/adminhtml/globals.js
    net::ERR_ABORTED 404 (Not Found)
```

The page may partially break: grids don’t load, buttons stop responding, or
JavaScript errors cascade. A hard reload fixes it temporarily — until it
happens again.

### Why is it intermittent?

The bug does not appear on the first page load. It surfaces only after
several AJAX navigations within the same browser session. A single smoke
test is unlikely to catch it.

---

## Root Causes

Magento ships a small script called `requirejs-min-resolver.min.js` that is
injected into every Admin page. Its job is to force RequireJS to always
request `.min.js` URLs. The original implementation contains two defects.

### Defect 1 — Stale `baseUrl` closure

```javascript
// Original Magento code
(function() {
    var ctx = require.s.contexts._,
        origNameToUrl = ctx.nameToUrl,
        baseUrl = ctx.config.baseUrl;  // captured ONCE at script-load time

    ctx.nameToUrl = function() {
        var url = origNameToUrl.apply(ctx, arguments);
        if (url.indexOf(baseUrl) === 0   // compared against the stale value
            && !url.match(/\/hugerte\//)
            && !url.match(/\/v1\/songbird/)) {
            url = url.replace(/(\.min)?\.js$/, '.min.js');
        }
        return url;
    };
}());
```

`baseUrl` is captured once when the script first runs. During AJAX navigation
Magento can reconfigure RequireJS. After reconfiguration `ctx.config.baseUrl` has a new value, but the
closed-over `baseUrl` variable is still the old one. The `indexOf` check fails
for every URL, the `.min.js` rewrite is skipped, and the browser requests the
plain `.js` file — which does not exist — resulting in a 404.

### Defect 2 — Double-wrapping on re-evaluation

On AJAX navigation the Admin can re-inject and re-evaluate
`requirejs-min-resolver.min.js`. Each evaluation runs
`ctx.nameToUrl = function() { ... origNameToUrl ... }` where `origNameToUrl`
was captured from the *current* `ctx.nameToUrl` — which is already the
patched version from the previous evaluation. The result is a growing chain:

```
patch3( patch2( patch1( original ) ) )
```

With no idempotency guard, each additional wrapper can alter the URL
differently, potentially producing wrong paths or double-suffixed filenames
like `globals.min.min.js`.

---

## The Fix

A single self-contained IIFE that closes both gaps:

```javascript
(function(){

  function patchCtx(c){
    if(!c||c.__mRF)return;   // idempotency guard — never patch twice
    c.__mRF=true;
    var p=c.nameToUrl;
    c.nameToUrl=function(){
      // baseUrl read dynamically on every call — never goes stale
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

### How each defect is addressed

| Defect | Original | Fix |
|--------|----------|-----|
| Stale `baseUrl` | Captured once in outer closure | Read as `c.config.baseUrl` inside the function on every call |
| Double-wrapping | No guard | `__mRF` flag on each context prevents patching more than once |

### Additional layers

| Layer | What it covers |
|-------|----------------|
| Context loop at load time | All RequireJS contexts already present, including `_` |
| `newContext` hook | Every new named context created after the script runs |
| `require.load` hook | Last-resort safety net — rewrites the URL just before the XHR fires |
| `__mNCF` / `__mRL` flags | Ensure the `newContext` and `load` hooks are installed exactly once |

### Exclusions

`hugerte` and `v1/songbird` are third-party libraries that ship only as
unminified JS and must not be rewritten.

---

## Compatibility

- Magento 2.4.x (tested on 2.4.7-p9)
- Theme-independent — works with any Admin theme
- No PHP, no `setup:di:compile`, no Composer dependency
- Only active when **Minify JavaScript Files** is enabled; has no effect otherwise

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
