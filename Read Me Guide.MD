# Setup: capture checkout user data → push to dataLayer on order-received

Single Custom HTML tag now (see `4-gtm-combined-user-data-tag.html`) — it decides what to do based on the current URL, so you only need one tag and one trigger.

*(The earlier two-tag files, `1-gtm-checkout-capture.html` and `2-gtm-order-received-push-userdata.html`, are superseded by this combined version — you can ignore/delete them.)*

## Tag setup

1. GTM → Tags → New → Custom HTML → paste `4-gtm-combined-user-data-tag.html`.
2. Trigger → **All Pages** (Page View, Fires on: All Pages).
3. Save as **"Checkout User Data - Capture & Push"**.

That's it — one tag, one trigger. The script itself checks `window.location.pathname`:
- Path contains `/checkout` (but not `/checkout/order-received/`) → binds a `pointerdown` listener on `#place_order`.
- Path contains `/checkout/order-received/` → reads localStorage and pushes to the dataLayer immediately.
- Any other page → does nothing.

## Critical step — tag sequencing

The purchase/Enhanced Conversions tag(s) on the order-received page still need to fire *after* this tag, otherwise `user_data` won't exist in the dataLayer yet.

Open each purchase-tracking tag (GA4 purchase, Google Ads conversion + Enhanced Conversions, Meta Pixel/CAPI, etc.) → **Advanced Settings → Tag Sequencing → "Fire a tag before this tag fires"** → enable it → select **"Checkout User Data - Capture & Push"**.

## How capture works

`pointerdown` fires before `click`/navigation, so the moment the customer presses "Place order" (button id `#place_order` inside `form.checkout`), the script reads the billing fields straight from the DOM — no per-field listeners, no re-binding needed after WooCommerce's AJAX re-renders. It only saves if `email` and `phone` are both present.

## Phone number normalization

Google's Enhanced Conversions requires phone numbers in **E.164 format**: a leading `+`, then country code, then the number, no spaces/dashes/brackets (e.g. `+61292659333`).

`normalizeAuPhoneNumber()` handles whatever format the customer types and converts it to that format before it's saved or pushed:

| Customer typed | Normalized |
|---|---|
| `0292659333` | `+61292659333` |
| `(02) 9265 9333` | `+61292659333` |
| `02-9265-9333` | `+61292659333` |
| `0412 345 678` | `+61412345678` |
| `61292659333` | `+61292659333` |
| `+61 2 9265 9333` | `+61292659333` |
| `+61292659333` | `+61292659333` (unchanged) |

If a customer enters a non-AU number with its own country code (e.g. a NZ `+64...` number), it's left as-is rather than forced into `+61`.

## Testing

1. Open Preview mode (Tag Assistant) → go through checkout with a test order, filling in email/phone/name/address (try a phone number with spaces/brackets to test normalization).
2. Press "Place order" → open the browser console and confirm the logs: field values, `Phone (normalized E.164)`, then "Checkout user data saved to localStorage". Check Application → Local Storage → `wc_checkout_user_data` has the values, and the phone is in `+61XXXXXXXXX` form.
3. Land on the order-received page → console should show "Pushing user_data to dataLayer" with the same data. Check the dataLayer for the `dl_user_data` event, then confirm your purchase/Enhanced Conversions tag fires *after* it and picks up the values.
4. Reload the order-received page a second time — console should log "No stored checkout user data found" and `dl_user_data` should not push again.
5. Once confirmed working, you can remove the `console.log` lines (or leave them — they only print in DevTools, customers never see them).

## Notes

- Field IDs (`billing_first_name`, `billing_email`, `billing_phone`, etc.) match totalcontacts.com.au's actual checkout markup, including the Select2-wrapped `billing_state` and the hidden, pre-filled `billing_country`.
- Data expires after 1 hour and is deleted immediately after being pushed, so it can't leak into an unrelated order on a shared/public computer.
- If checkout uses a custom domain/subdomain different from the order-received page, localStorage won't carry over (it's per-origin) — let me know if that's the case, this would need a different approach (e.g. passing via URL params or server-side order meta).
