---
name: request-context-multer
description: Restores AsyncLocalStorage request context after multer. Use when routes use upload.array, upload.fields, upload.none, or multipart/form-data and the controller or downstream code needs getMerchantId(), getRequestContextOptional(), or tenant context.
---

# Request Context + Multer

Multer can lose `AsyncLocalStorage` context because multipart parsing uses async/event callbacks outside the original request chain. The request context middleware stores the original context on `req.__requestContext`; controller entries must restore it when needed.

## When To Apply

- A route uses `upload.array(...)`, `upload.fields(...)`, or `upload.none()`.
- The controller, service, repository, upload manager, or validation code calls `getMerchantId()`, `getRequestContextOptional()`, or request-context helpers.
- Multipart FormData has fields only and no files; use `upload.none()` so `req.body` is parsed, then still restore context.
- Errors mention missing request context, missing merchantId, or tenant context after a multipart route.

## Standard Controller Pattern

Use a thin public controller method that restores context, then delegates real work to a private handler.

```ts
async updateXxx(req: Request, res: Response): Promise<void> {
  const context = restoreRequestContextFromReq(req);
  if (!context) {
    res.status(400).json({
      success: false,
      error: { code: 'MISSING_MERCHANT_ID', message: 'Merchant ID is required' },
    });
    return;
  }

  if (!requestContext.getStore()) {
    await runWithContextAsync(context, () => this.doUpdateXxx(req, res));
    return;
  }

  await this.doUpdateXxx(req, res);
}
```

If the local controller still uses `requestContext.run(context, async () => { ... })`, keep the shape consistent with nearby code, but prefer `runWithContextAsync` where it already exists because it is promise-friendly.

## Route Rules

- File uploads: `upload.array('images', n)` or `upload.fields([...])`.
- Multipart fields without files: `upload.none()`.
- Do not rely only on wrapping multer at the route level; multer internals can still cross async boundaries where context is lost.

## Current References

- Middleware stores context: `xituan_backend/src/shared/middleware/request-context.middleware.ts`.
- Helpers: `xituan_backend/src/shared/infrastructure/request-context.ts`.
- Product/category examples: `xituan_backend/src/domains/product/controllers/product.controller.ts`.
- Platform settings uses `runWithContextAsync`: `xituan_backend/src/domains/platform-setting/controllers/platform-setting.controller.ts`.
- Full guide: `xituan_agent/devGuide/request-context-multer-workaround.md`.

## Checklist

- Route has the correct multer middleware.
- Controller entry restores context from `req`.
- Missing context returns a 400 response before business logic.
- If store is missing, business handler runs inside restored context.
- Real create/update logic lives in a private handler so the wrapper remains small.
