# @xpayeg/sdk

## 1.0.0
### Major Changes



- [#100](https://github.com/xpayeg/xpay/pull/100) [`76481af`](https://github.com/xpayeg/xpay/commit/76481af7da2fab85f3557666315374c09cc5dddf) Thanks [@Elmosh](https://github.com/Elmosh)! - Initial 1.0.0 release.
  
  `@xpayeg/sdk` ships the `loadXPay()` loader plus the full public TypeScript surface for embedding XPay payments — `Elements`, `PaymentElement`, `CardElement`, drop-in `checkout()`, `initCheckout()`, action methods (`confirm`, `applyPromotionCode`, `removePromotionCode`, `updateLineItemQuantity`, `submit`, `fetchUpdates`, `changeAppearance`), and the tagged-union `ActionResult` / `XPayError` shapes.
  
  `@xpayeg/react` ships the React bindings: `XPayProvider`, `useCheckout`, `useXPay`, `useElements`, `useConfirmPayment`, `<PaymentElement>`, `<CardElement>`, and `<CheckoutButton>` — all SSR-safe, with stable event listeners and smart option diffing.
  
  The runtime is served from `https://checkout.xpay.app/v1/sdk.js` and auto-loaded by `loadXPay()`. Both packages publish with npm provenance.
