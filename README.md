# @xpayeg/sdk

XPay JavaScript SDK -- loader and TypeScript types for embedding XPay payments on any website.

## Documentation

Full guides, API reference, and live examples: **<https://docs.xpay.app>**

This README is a quick-start. The docs site is the authoritative reference.

## Installation

**Via npm (recommended for bundler projects):**

```bash
npm install @xpayeg/sdk
```

**Via CDN (for non-bundler setups):**

```html
<script src="https://checkout.xpay.app/v1/sdk.js"></script>
```

Both ESM (`import`) and CommonJS (`require`) are supported, with matching TypeScript declarations. Your tooling selects the appropriate entrypoint automatically.

## Step 1: Create a Checkout Session [Server-side]

On your server, create a Checkout Session and return the `clientSecret` to your frontend.

```javascript
// Your server (Node.js example)
app.post("/api/create-checkout", async (req, res) => {
  const response = await fetch("https://api.xpay.app/checkout/sessions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.XPAY_SECRET_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      uiMode: "custom", // 'custom' for Elements SDK, 'embedded' for drop-in modal, 'hosted' for redirect
      lineItems: [
        {
          priceData: {
            unitAmount: 50000, // 500.00 EGP in piasters
            currency: "EGP",
            productData: { name: "Premium Plan" },
          },
          quantity: 1,
        },
      ],
      afterCompletion: {
        type: "redirect",
        redirect: {
          // {CHECKOUT_SESSION_ID} is automatically replaced with the actual session ID
          url: "https://yoursite.com/success?session_id={CHECKOUT_SESSION_ID}",
        },
      },
    }),
  });

  const session = await response.json();
  res.json({ clientSecret: session.clientSecret });
});
```

## Step 2: Initialize Checkout [Client-side]

### `initCheckout` (Recommended)

Returns a single object with session fields and action methods merged together.

```javascript
import { loadXPay } from "@xpayeg/sdk";

const xpay = await loadXPay("pk_test_xxx");

// Fetch client secret from your server
const { clientSecret } = await fetch("/api/create-checkout", { method: "POST" }).then((r) =>
  r.json(),
);

// Initialize checkout -- returns session data + action methods in one object
const checkout = await xpay.initCheckout({ clientSecret });

// Destructure session fields and actions together
const {
  status,
  currency,
  amountTotal,
  amountSubtotal,
  canConfirm,
  paymentMethods,
  confirm,
  applyPromotionCode,
  removePromotionCode,
  updateLineItemQuantity,
} = checkout;

// Display in your UI
document.getElementById("total").textContent = `${currency} ${(amountTotal / 100).toFixed(2)}`;
document.getElementById("methods").textContent =
  `Pay with: ${paymentMethods.map((pm) => pm.displayName).join(", ")}`;
```

`clientSecret` accepts `Promise<string> | string`, so you can pass the fetch directly:

```javascript
const checkout = await xpay.initCheckout({
  clientSecret: fetch("/api/create-checkout", { method: "POST" })
    .then((r) => r.json())
    .then((data) => data.clientSecret),
});
```

**Session fields on the checkout object:**

| Field            | Type                      | Example                             | Description                                   |
| ---------------- | ------------------------- | ----------------------------------- | --------------------------------------------- |
| `id`             | `string`                  | `"cs_test_abc"`                     | Session ID                                    |
| `status`         | `SessionStatus`           | `{ type: "open" }`                  | Structured status (see below)                 |
| `canConfirm`     | `boolean`                 | `true`                              | Whether the session is ready for confirmation |
| `amountTotal`    | `number`                  | `52450`                             | Final total in smallest currency unit         |
| `amountSubtotal` | `number`                  | `50000`                             | Subtotal before fees/discounts                |
| `currency`       | `string`                  | `"EGP"`                             | ISO 4217 currency code                        |
| `merchantName`   | `string`                  | `"My Store"`                        | Merchant display name                         |
| `livemode`       | `boolean`                 | `false`                             | Whether live or test mode                     |
| `paymentMethods` | `PaymentMethodInfo[]`     | `[{ type, displayName, category }]` | Available payment methods                     |
| `lineItems`      | `CheckoutLineItem[]`      |                                     | Line items with prices and quantities         |
| `totalDetails`   | `TotalDetailsResponseDto` |                                     | Breakdown of fees, discounts, VAT             |
| `discounts`      | `DiscountResponseDto[]`   |                                     | Applied discounts                             |

### Session Status

`status` is a structured object, not a plain string:

```javascript
// Check session status
if (checkout.status.type === "open") {
  // Session is active, show payment form
}

if (checkout.status.type === "expired") {
  // Session expired, show message or create a new one
  showMessage("This checkout session has expired.");
}

if (checkout.status.type === "complete") {
  // Payment finished -- check paymentStatus for details
  console.log(checkout.status.paymentStatus); // "paid"
  showMessage("Payment successful!");
}
```

### Alternative: `elements`

For event-driven initialization and lower-level control.

```javascript
const xpay = await loadXPay("pk_test_xxx");
const elements = xpay.elements({ clientSecret });

elements.on("ready", ({ session }) => {
  console.log(session.status); // { type: "open" }
  console.log(session.canConfirm); // true
  console.log(session.amountTotal); // 52450
  console.log(session.paymentMethods); // [{ type: 'card', ... }]
});

elements.on("loaderror", (event) => {
  // Only fires for actual failures (invalid secret, network error, etc.)
  // event: { type: "invalid_request_error" | "api_error" | "network_error", message, code?, param?, docUrl? }
  console.error("Failed to load:", event.message);
});

elements.on("error", (error) => {
  // Fires for unsolicited errors not triggered by a merchant action.
  // Examples: session expired during fee recalculation when switching payment methods, BIN detection failure.
  console.log(error.type); // "invalid_request_error"
  console.log(error.code); // "checkout_session_expired"
  console.log(error.message); // "This checkout session has expired"
  console.log(error.docUrl); // "https://docs.xpay.app/api/errors#checkout_session_expired"
});
```

## Step 3: Mount the Payment Form

### Payment Element (handles method selection + card form)

```javascript
const checkout = await xpay.initCheckout({ clientSecret });
const elements = checkout.getElements();

const paymentElement = elements.create("payment");
paymentElement.mount("#payment-element");

paymentElement.on("change", (event) => {
  submitButton.disabled = !event.complete;
  console.log(event.value.type); // 'card', 'valu', etc.
});
```

## Step 4: Confirm the Payment

`confirm()` returns a tagged union -- check `type` to determine the outcome.

By default, the result is **returned to your code** (`redirect: "if_required"`). Use `redirect: "always"` to redirect to the `afterCompletion.redirect.url` you set on the server.

```javascript
// Default behavior: returns result to your code (no redirect)
const result = await checkout.confirm({
  customerDetails: {
    email: "customer@example.com",
    name: "Ahmed Hassan",
    phone: "+201234567890",
  },
});

if (result.type === "error") {
  errorDiv.textContent = result.error.message;
} else {
  console.log(result.session.status); // { type: "complete", paymentStatus: "paid" }
  window.location.href = "/thank-you";
}
```

To redirect after success instead:

```javascript
await checkout.confirm({
  customerDetails: {
    email: "customer@example.com",
    name: "Ahmed Hassan",
  },
  redirect: "always", // Redirect to afterCompletion.redirect.url
});
// ^ If successful, the page navigates away. Code below only runs on error.
```

**Redirect behavior:**

| `redirect`        | Behavior                                                  |
| ----------------- | --------------------------------------------------------- |
| Not set (default) | `"if_required"` — returns result to your code             |
| `"always"`        | Redirects to the session's `afterCompletion.redirect.url` |
| `"if_required"`   | Returns result to your code — no redirect                 |

Your server sets that URL when it creates the session. XPay navigates there unchanged, appending nothing.

### Pre-validation with `submit()`

Call `submit()` before `confirm()` to validate all fields and get the selected payment method:

```javascript
const { error, selectedPaymentMethod } = await elements.submit();
if (error) {
  errorDiv.textContent = error.message;
  return;
}

// Fields are valid, proceed to confirm
const result = await checkout.confirm({ paymentMethod: selectedPaymentMethod });
```

For deferred Elements (`mode: "payment"`), Apple Pay authorization happens in
`elements.submit()`. Call it directly from the customer's payment click, before
awaiting form validation or creating the checkout session when possible. If an
async caller has already lost browser activation, the SDK shows a top-level
continuation dialog; its Apple Pay button opens the sheet synchronously. The
original `submit()` remains pending and the caller continues automatically after
authorization is stored. Repeated calls share this pending promise. The sheet uses the
preloaded processing quote for the displayed total. For foreign-currency pricing,
Apple Pay shows the converted EGP amount, using XPay's shared currency conversion.
A successful `submit()` means the
authorization was stored, not that payment succeeded.

After `submit()`, create the session on your server and pass its `clientSecret` to
`xpay.confirmPayment()`. Token storage, your server request and confirmation share
one 28-second budget starting when the customer authorizes Apple Pay; server
authorization expiry can shorten it further. Repeated `submit()` calls reuse the
same unexpired authorization. If the session's final EGP total differs from the
earlier quote (for example, because of line-item rounding or a rate change), the
SDK closes the old sheet without charging and asks for fresh approval through
its existing continuation dialog. It then resumes the same `confirmPayment()`
call with the same session. No integration changes are needed. Fee pass-through
still requires session-first Elements. Cart or
payment-method changes invalidate unused authorization and require a new click.
Treat `payment_still_confirming` as an unknown outcome. Retain the existing
checkout session and client secret when retrying: the server resolves the previous
attempt before allowing another charge. Never create a replacement session to
retry an uncertain payment. A definitive refusal closes the sheet and lets the
same Elements instance accept a new preparation and fresh customer consent.
Successful or uncertain dispatched authorizations remain protected against reuse;
restarting the payment UI requires fresh Elements and fresh customer consent,
while retaining the same checkout session. Only the authoritative confirmation
result determines payment success.

If your server cannot create the session after `submit()` succeeds, do not call
`confirmPayment()`. Call `elements.destroy()` to close the sheet immediately and
invalidate unused consent, then create and mount fresh Elements before another
payment click. There is currently no public cancel-only method. Without explicit
cleanup, the original authorization deadline closes the sheet. Destroying Elements
after confirmation was dispatched stops the UI; it does not cancel a bank charge.

## Step 4b: Update Session (Promo Codes, Quantities)

Action methods return `Promise<{ type: "success", session } | { type: "error", error }>`.

```javascript
// Apply promotion code
const result = await checkout.applyPromotionCode("SAVE20");
if (result.type === "error") {
  errorDiv.textContent = result.error.message; // e.g., "Invalid promotion code"
} else {
  console.log("New total:", result.session.amountTotal);
}

// Remove promotion code
await checkout.removePromotionCode();

// Update line item quantity (object param, not positional)
await checkout.updateLineItemQuantity({ lineItem: "li_abc123", quantity: 3 });
```

### Listening for Session Changes

Every update (promo codes, quantity changes, payment method selection, BIN detection) triggers a `change` event with the full updated `CheckoutSession`:

```javascript
// initCheckout -- use on('change', ...)
checkout.on("change", (session) => {
  totalDiv.textContent = `Total: ${session.currency} ${(session.amountTotal / 100).toFixed(2)}`;

  // Amounts breakdown via totalDetails
  console.log(session.totalDetails?.amountPlatformFee); // Processing fee
  console.log(session.totalDetails?.amountCollectedVat); // VAT
  console.log(session.totalDetails?.amountDiscount); // Discount amount
  console.log(session.totalDetails?.amountShipping); // Shipping
  console.log(session.totalDetails?.amountTax); // Tax

  console.log(session.discounts); // Applied discounts
  console.log(session.lineItems); // Updated line items
});
```

With the classic `elements` API:

```javascript
elements.on("change", (session) => {
  console.log(session.amountTotal);
  console.log(session.totalDetails?.amountPlatformFee);
});
```

### Re-sync from Server

Force a fresh session fetch if needed:

```javascript
const result = await elements.fetchUpdates();
if (result.type === "success") {
  console.log("Session refreshed:", result.session.amountTotal);
}
```

## Step 5: Show a Success Page

Retrieve the session from **your server** (using your API key) to display order details.

**Server endpoint:**

```javascript
// Your server -- retrieves session using your API key
app.get("/api/order-status", async (req, res) => {
  const response = await fetch(`https://api.xpay.app/checkout/sessions/${req.query.session_id}`, {
    headers: { Authorization: `Bearer ${process.env.XPAY_SECRET_KEY}` },
  });
  const session = await response.json();
  res.json(session);
});
```

**Client-side success page:**

```javascript
const sessionId = new URLSearchParams(window.location.search).get("session_id");
const session = await fetch(`/api/order-status?session_id=${sessionId}`).then((r) => r.json());

document.getElementById("status").textContent =
  session.paymentStatus === "paid" ? "Payment Confirmed" : "Processing...";
document.getElementById("total").textContent =
  `${session.currency} ${(session.amountTotal / 100).toFixed(2)}`;

// Amounts breakdown
// session.totalDetails.amountDiscount       -- 0
// session.totalDetails.amountPlatformFee    -- 1750 (Processing Fee, if fees pass-through)
// session.totalDetails.amountCollectedVat   -- 700  (VAT)
// session.totalDetails.amountShipping       -- 0
// session.totalDetails.amountTax            -- 0

// Line items
session.lineItems?.forEach((item) => {
  console.log(item.price.product.name, "x", item.quantity, "=", item.amountTotal);
});
```

## Step 6: Handle Webhooks [Server-side]

Listen for webhook events on your server. This is the source of truth for order fulfillment.

```javascript
app.post("/webhooks/xpay", (req, res) => {
  const event = req.body;

  if (event.type === "checkout.session.completed") {
    fulfillOrder(event.data); // Ship product, send email, update DB
  }

  res.json({ received: true });
});
```

## Drop-in Checkout (Modal)

The simplest integration -- opens the full checkout in a modal overlay. No form needed.

```html
<button id="checkout-button">Pay Now</button>

<script src="https://checkout.xpay.app/v1/sdk.js"></script>
<script>
  document.getElementById("checkout-button").addEventListener("click", async () => {
    const xpay = XPay("pk_test_xxx");

    const checkout = xpay.checkout({
      clientSecret: "cs_test_abc_secret_xyz",
      mode: "modal",
      onComplete: (result) => {
        window.location.href = "/success";
      },
      onClose: () => {
        console.log("Customer closed checkout");
      },
    });

    checkout.open();
  });
</script>
```

## React Integration

The `@xpayeg/react` package provides a `useCheckout()` hook that wraps `initCheckout` with loading/error states:

```javascript
import { useCheckout } from "@xpayeg/react";

function CheckoutPage() {
  const checkoutState = useCheckout();

  if (checkoutState.type === "loading") {
    return <Spinner />;
  }

  if (checkoutState.type === "error") {
    return <div>Error: {checkoutState.error.message}</div>;
  }

  const { currency, lineItems, amountTotal, confirm } = checkoutState.checkout;

  return (
    <div>
      <h2>
        Total: {currency} {(amountTotal / 100).toFixed(2)}
      </h2>
      {lineItems?.map((item) => (
        <div key={item.id}>
          {item.price.product.name} x {item.quantity}
        </div>
      ))}
      <button onClick={() => confirm({ customerDetails: { email } })}>Pay</button>
    </div>
  );
}
```

## Appearance

Override the session's `brandingSettings` at runtime:

```javascript
const checkout = await xpay.initCheckout({
  clientSecret,
  appearance: {
    colorMode: "dark",
    borderStyle: "pill",
    inputStyle: "filled",
    colors: { primary: "#FF6B35" },
  },
});

// Update later with changeAppearance()
checkout.changeAppearance({ colorMode: "light" });
```

With the classic API:

```javascript
const elements = xpay.elements({ clientSecret, appearance: { colorMode: "dark" } });
elements.changeAppearance({ colorMode: "light" });
```

## API Reference

### `loadXPay(publishableKey)`

Loads the XPay SDK from CDN. Returns a Promise. Call at module level, not inside components.

```javascript
import { loadXPay } from "@xpayeg/sdk";
const xpay = await loadXPay("pk_test_xxx");
```

### `xpay.initCheckout(options)`

Initialize a checkout session. Resolves with a merged object containing session fields and action methods.

```javascript
const checkout = await xpay.initCheckout({
  clientSecret: "cs_test_abc_secret_xyz", // string | Promise<string>
  appearance: { colorMode: "dark" },
  locale: "ar",
});

// Session fields: status, canConfirm, amountTotal, currency, paymentMethods, lineItems, totalDetails, ...
// Action methods: confirm(), applyPromotionCode(), removePromotionCode(), updateLineItemQuantity(),
//                 submit(), fetchUpdates(), changeAppearance(), on(), getElements()
```

### `xpay.elements(options)`

Create an Elements instance for mounting payment elements.

### `xpay.checkout(options)`

Create a drop-in checkout instance (modal or inline).

### `xpay.confirmPayment(options)`

Confirm and submit a payment (classic API).

## TypeScript

Full TypeScript support with all types exported:

```typescript
import type {
  XPayInstance,
  Elements,
  PaymentMethodInfo,
  CheckoutSession,
  SessionStatus,
  CustomerDetails,
  Appearance,
  InitCheckoutResult,
  CheckoutActions,
  ActionResult,
  XPayError,
  CheckoutLineItem,
  CheckoutTotalDetails,
  CheckoutFees,
  CheckoutDiscount,
} from "@xpayeg/sdk";
```
