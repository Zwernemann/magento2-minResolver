# MinResolver

Fixes intermittent `*.min.js` 404 errors in Magento 2 caused by two defects in the generated `requirejs-min-resolver.js`.

## The Problem

### When does this apply?

The bug appears when **Minify JavaScript Files** is enabled in Admin
(Stores → Config → Advanced → Developer → JavaScript Settings).
This setting is independent of `MAGE_MODE` — it works in developer, default,
and production mode alike.

When minification is active, Magento generates `requirejs-min-resolver.min.js`
and injects it into every Admin page to ensure RequireJS loads `.min.js` URLs.
If that resolver has a gap (see Root Cause below), a module gets requested as
plain `.js`, which returns a 404.

```
GET /static/.../mage/adminhtml/globals.js net::ERR_ABORTED 404 (Not Found)
```

The error does not appear on the first page load. It surfaces only after
navigating back and forth between Sales → Orders → Create Order several times.

### Why intermittent? Lazy context creation, not a race condition

This is not a timing/async race. It is a **lazy context creation + accumulated
state** problem:

1. On first page load the `_` (default) RequireJS context is patched — modules
   load correctly as `.min.js`.
2. Subsequent navigations reuse the patched `_` context — no errors.
3. After enough navigations, Magento triggers a code path that creates a **new
   named RequireJS context** (e.g. the Sales order form uses
   `require({context: 'order_...'})` or calls `require.s.newContext` internally).
4. That new context was never patched — it resolves URLs as plain `.js`.
5. 404.

The "x navigations" threshold is not fixed. It depends on which combination of
pages was visited and which code paths were exercised. The same session can go
clean for a long time and then suddenly break when a particular state is reached.

### Root Cause in the Original Script

Magento ships `requirejs-min-resolver.min.js` to force `.min.js` resolution.
The original version:

```javascript
(function() {
    var ctx = require.s.contexts._,
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

This has two critical weaknesses:

**1. Only patches the `_` context**
Any named context created via `require({context: ...})` or `require.s.newContext`
is completely unaffected.

**2. No guard against future contexts**
Contexts created after the script runs are never patched.

### What a Previous Fix Attempt Introduced

An intermediate version (v4/v5) added a second IIFE below the original block to
patch all contexts and intercept `newContext`. However, the original first block
was kept unchanged. This caused:

- The `_` context to be **patched twice** (once by the first block without an
  idempotency flag, then again by the second block's loop)
- A wrapped call chain: `secondPatch(firstPatch(original))` — functionally
  harmless due to the regex guards, but unnecessary and fragile

### Current Solution (v6)

The first block is removed entirely. A single self-contained IIFE handles
everything:

```javascript
(function(){
  // Patch one context. __mRF flag prevents double-patching.
  function patchCtx(c){
    if(!c||c.__mRF)return;
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

  // 1. Patch all currently existing contexts (including _)
  var ctxs=require.s.contexts;
  for(var n in ctxs){
    if(Object.prototype.hasOwnProperty.call(ctxs,n))patchCtx(ctxs[n]);
  }

  // 2. Patch contexts created in the future
  if(!require.s.__mNCF){
    require.s.__mNCF=true;
    var oNC=require.s.newContext;
    require.s.newContext=function(){
      var c=oNC.apply(this,arguments);
      patchCtx(c);
      return c;
    };
  }

  // 3. Intercept require.load as a last-resort safety net
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

| Layer | What it does |
|-------|--------------|
| `patchCtx` loop | Patches every context that already exists at script-load time, including `_` |
| `newContext` hook | Ensures every context created after this point is patched immediately |
| `require.load` hook | Final safety net — rewrites the URL just before the XHR fires, catching any context that was somehow missed |
| `__mRF` / `__mNCF` / `__mRL` flags | Idempotency guards — each layer is applied at most once even if the script is loaded multiple times |
| `hugerte` / `v1/songbird` exclusions | Third-party libraries that ship only as unminified JS and must not be rewritten |


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
