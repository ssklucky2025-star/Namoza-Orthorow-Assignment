# OrthoNow — Namoza Developer Assignment

Submission by Sumit Kumar for the Developer (Client Web + Martech) role.

## Contents
- `task1-gtm-schema.md` — full GTM event schema, booking-funnel dataLayer JSON, and Google Ads conversion recommendation.
- `task2-landing-page.html` — single-file, dependency-free "Book a Consultation" landing page. Open directly in a browser, no server needed.
- `task2-pagespeed-screenshot.png` — PageSpeed Insights Mobile score for the landing page. *(Add this yourself — see submission steps below.)*
- `task3-integration-writeup.md` — HubSpot / WhatsApp (Karix) / Google Ads integration architecture.

## How to run Task 2 locally
1. Download `task2-landing-page.html`.
2. Double-click it, or open it in any browser via `File → Open`.
3. Open the browser console (`Cmd+Option+J` on Mac Chrome, `F12` on Windows) before submitting the form.
4. Fill in a name and a 10-digit number starting with 6–9, click **Book My Consultation**.
5. You'll see the `dataLayer` push logged if you run `console.log(window.dataLayer)` in the console, and the page swaps to a thank-you state with no reload.