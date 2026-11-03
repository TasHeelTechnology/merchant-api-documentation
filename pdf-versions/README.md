# PDF versions

Printable merchant guides generated from this documentation. Every change is saved as a new file; older files are kept for reference and never overwritten.

## Naming

`<Guide_Name>_v<version>_<YYYY-MM-DD>.pdf`, where the date is the day the PDF was generated.

Example: `Tasheel_Assisted_Checkout_Integration_Guide_v2.1_2026-11-03.pdf`

If the guide changes again on the same day, keep the date and raise the version (for example `v2.2_2026-11-03`).

## Versions

| File | Date | Changes |
|---|---|---|
| [Tasheel_Assisted_Checkout_Integration_Guide_v2.1_2026-11-03.pdf](./Tasheel_Assisted_Checkout_Integration_Guide_v2.1_2026-11-03.pdf) | 2026-11-03 | **Latest.** Adds step 8: report the booking result (`POST /checkout/booking-status`, `NOT_TICKETED` refund). Adds payments that are not final straight away: `PAYMENT_PENDING`, `PAYMENT_ACTION_REQUIRED`, `PAYMENT_FALLBACK_INITIALIZED`, `PAYMENT_IN_PROGRESS`, status polling, webhook `status:false`, return URL `txnId`/`status`. |
| [Tasheel_Assisted_Checkout_Integration_Guide_v2.0_2026-10-25.pdf](./Tasheel_Assisted_Checkout_Integration_Guide_v2.0_2026-10-25.pdf) | 2026-10-25 | First assisted checkout guide (OTP consent, saved cards, confirm, status). |

The source of these PDFs is kept outside this repository (`Tasheel/Integrations/Assisted Checkout/source/guide/guide.html`); print it with Chromium (A4, backgrounds on).
