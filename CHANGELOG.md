# @xpayeg/sdk

## 2.0.0
### Major Changes



- [#331](https://github.com/xpayeg/xpay/pull/331) [`bdc6a48`](https://github.com/xpayeg/xpay/commit/bdc6a4865f8c0326a4c0eb3dd20808e1207d682a) Thanks [@Elmosh](https://github.com/Elmosh)! - **Breaking:** `returnUrl` is removed from `confirmPayment()` options. The return destination is always the checkout session's `afterCompletion.redirect.url`, set server-side at session creation.
  
  The URL is sent to the bank during authentication, before the browser leaves your page, so a value passed at confirm time could only ever conflict with what the bank already received. Delete `returnUrl` from `confirmPayment()` and set `afterCompletion.redirect.url` on `createSession`. `redirect: "always"` now follows that destination.
  
  **Breaking:** `afterCompletion` is now required for `uiMode: "embedded"` and `uiMode: "custom"`, and must be `type: "redirect"`. That URL is where the bank returns when authentication takes over the full page, which happens in in-app browsers where an embedded challenge cannot run. `hosted_confirmation` is rejected for these modes since those integrations run on your own site and there is no XPay page to return to.
  
  Also fixed: the 3DS challenge now renders at the size declared to the issuer (previously cropped or oversized), the overlay waits for the issuer's page to paint instead of showing an empty frame, and a challenge that cannot run in an iframe takes over the full tab instead of failing silently.

### Patch Changes



- [#333](https://github.com/xpayeg/xpay/pull/333) [`3dcea81`](https://github.com/xpayeg/xpay/commit/3dcea81b3e3f67267624105560c38f2615c35d96) Thanks [@Elmosh](https://github.com/Elmosh)! - Documentation: the `redirect: "always"` tables and examples now state that the destination is the checkout session's `afterCompletion.redirect.url`, set server-side at session creation.

## 1.0.1
### Patch Changes



- [#103](https://github.com/xpayeg/xpay/pull/103) [`7402e27`](https://github.com/xpayeg/xpay/commit/7402e27cde482148f126b2073694ef38386ea87a) Thanks [@Elmosh](https://github.com/Elmosh)! - Remove `CardElement` from both SDKs — `PaymentElement` is the only supported element going forward. It handles every payment method (Card, ValU, Fawry, etc.) through one unified UI and renders the appropriate fields based on the customer's choice.
  
  This release also republishes both packages through the proper `pnpm publish` flow, which fixes the `workspace:*` protocol leak in 1.0.0's `devDependencies` and `peerDependencies`. `npm install @xpayeg/sdk` and `npm install @xpayeg/react` now work cleanly.
  
  **Migration:** replace `elements.create("card")` with `elements.create("payment")`. If you previously rendered `<CardElement>` only when the user selected "Card" from your own picker, switch to `<PaymentElement>` which provides both the picker and the card form together.
  
  ```ts
  // Before
  const cardElement = elements.create("card");
  cardElement.mount("#card");
  
  // After
  const paymentElement = elements.create("payment");
  paymentElement.mount("#payment");
  ```

## 1.0.0
### Major Changes



- [#100](https://github.com/xpayeg/xpay/pull/100) [`76481af`](https://github.com/xpayeg/xpay/commit/76481af7da2fab85f3557666315374c09cc5dddf) Thanks [@Elmosh](https://github.com/Elmosh)! - Initial 1.0.0 release.
  
  `@xpayeg/sdk` ships the `loadXPay()` loader plus the full public TypeScript surface for embedding XPay payments — `Elements`, `PaymentElement`, `CardElement`, drop-in `checkout()`, `initCheckout()`, action methods (`confirm`, `applyPromotionCode`, `removePromotionCode`, `updateLineItemQuantity`, `submit`, `fetchUpdates`, `changeAppearance`), and the tagged-union `ActionResult` / `XPayError` shapes.
  
  `@xpayeg/react` ships the React bindings: `XPayProvider`, `useCheckout`, `useXPay`, `useElements`, `useConfirmPayment`, `<PaymentElement>`, `<CardElement>`, and `<CheckoutButton>` — all SSR-safe, with stable event listeners and smart option diffing.
  
  The runtime is served from `https://checkout.xpay.app/v1/sdk.js` and auto-loaded by `loadXPay()`. Both packages publish with npm provenance.
