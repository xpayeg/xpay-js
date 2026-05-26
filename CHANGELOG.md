# @xpayeg/sdk

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
