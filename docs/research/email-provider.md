# Transactional email provider for Cadence

Research for issue #7 (part of map #2, "Notifications for Cadence"). Checked 2026-11-12 against each vendor's own pricing and docs pages. Prices change, so recheck the links before you buy.

**Question:** Which transactional email provider should a small Node / React Router + SQLite app use for notification email? Compare price at low volume, Node SDK, deliverability, templates (including React Email), local dev and test mode, bounce and complaint webhooks, and domain verification.

**Recommendation: Resend.** Postmark is the runner-up. Amazon SES is the choice if cost at high volume ever matters more than setup time.

## At a glance

| | Resend | Postmark | Amazon SES | SendGrid | Mailgun (alternative) |
|---|---|---|---|---|---|
| Free tier | 3,000/mo, capped at 100/day, permanent [1] | 100/mo, permanent, no overages [2] | No ongoing free quota; new AWS accounts get up to $200 in credits for 6 months [3] | 60-day trial, 100/day, then paid only [4] | 100/day, 1 domain, 1 day of logs [5] |
| First paid tier | Pro $20/mo for 50k [1] | Basic $15/mo for 10k, then $1.80 per 1k [2] | $0.10 per 1k, pay as you go [3] | Essentials from $19.95/mo for 50k–100k [4] | Basic $15/mo for 10k [5] |
| Node SDK | `resend`, returns `{ data, error }`, takes a `react` prop [6] | `postmark` (TypeScript), `sendEmailWithTemplate()` [7] | AWS SDK v3 `@aws-sdk/client-sesv2` (generic AWS client) | `@sendgrid/mail` | `mailgun.js` |
| Templates | Send rendered React Email directly; Resend maintains React Email [6][8] | Server-side Mustache-style templates and layouts, plus a validate endpoint [7] | Stored templates via API | Dynamic templates | Stored templates |
| Test mode | `delivered@`, `bounced@`, `complained@`, `suppressed@resend.dev`; counts toward quota [9] | `POSTMARK_API_TEST` token (accepted, never delivered); sandbox servers [10][11] | Mailbox simulator (`success@`, `bounce@`, `complaint@`, `suppressionlist@simulator.amazonses.com`); works in sandbox, billed, does not use up quota [12] | No dedicated sink addresses found | No dedicated sink addresses found |
| Bounce/complaint events | HTTPS webhooks `email.bounced`, `email.complained`, and more, signed with Svix and checked by `resend.webhooks.verify()` [13][14] | HTTPS webhooks for Bounce, Spam complaint, Delivery, and more; **no HMAC signing** (basic auth plus IP allowlist only) [15] | Through SNS topics or event publishing (SNS/Firehose); no direct HTTP webhook, so you confirm an SNS subscription [16] | Event Webhook, ECDSA-signed or OAuth [17] | Webhooks (not checked) |
| Domain setup | DKIM + SPF (TXT and MX/CNAME on a `send.` subdomain); ~15 min typical, up to 72 h; DMARC recommended [18] | DKIM TXT + Return-Path CNAME (`pm_bounces` → `pm.mtasv.net`); up to 48 h [19] | 3 Easy DKIM CNAMEs, optional custom MAIL FROM (MX + SPF); up to 72 h; **plus a sandbox exit request (reply within 96 h)** [20][21] | Domain authentication DNS records | DNS records |
| Account gate | None beyond domain verification | Manual account approval; until then you can only send to your own domain [10] | Sandbox per region: verified recipients only, 200/day, 1/sec [21] | Trial ends at 60 days | — |

## Findings by criterion

### Price at low volume
Cadence's notification volume is small: hundreds to low thousands of emails a month at first.

- **Resend**'s free plan (3,000/mo, 100/day) probably covers Cadence in production at launch. The 100/day cap is the limit you will hit first, for example with a burst of digests. Pro costs $20/mo for 50k [1].
- **Postmark**'s free plan is only 100/mo, and it has no tier between 100 and 10,000 [2]. A real deployment costs at least $15/mo from day one.
- **SES** is by far the cheapest per message: $0.10 per 1k, about $0.10/mo at Cadence's volume [3]. The cost is setup and operations, not money.
- **SendGrid** no longer has a permanent free plan, only a 60-day trial [4]. It is the worst fit for a small app.

### Node SDK quality
- Resend's SDK is small and typed. It returns `{ data, error }` instead of throwing, and accepts a React Email component through `react` (Node only) [6]. It also ships a webhook verification helper [14].
- Postmark's `postmark` library is TypeScript and mature. It covers templates, batch sends (up to 500 per call), and message streams [7].
- SES uses the general-purpose AWS SDK v3. It is capable but verbose, and it needs AWS credentials and IAM in the app's environment.

### Deliverability
No neutral first-party benchmark compares these providers, so treat this section as judgment, not measurement.
- Postmark structurally separates transactional and broadcast mail into "parallel but completely separate sending infrastructures" [2], and it vets accounts before letting them send [10]. Both protect its shared-IP reputation, which is why it has the strongest reputation for transactional mail.
- Resend recommends sending from a dedicated subdomain (for example `updates.example.com`) to isolate reputation [22]. It also documents DMARC and BIMI [18].
- All five require DKIM and SPF alignment. At Cadence's volume, dedicated IPs are irrelevant: Resend sells them only above 3,000/day, Postmark only above 300k/mo [1][2].

### Templates and React Email
- React Email is MIT-licensed and maintained by Resend. It renders to HTML and plain text with `render()` and `toPlainText()`, and its README says emails can be converted to HTML "and sent using any email service provider." The repo ships examples for Resend, Postmark, SES, SendGrid, Mailgun, and Nodemailer [8][23].
- So React Email does **not** lock us in. With Resend you pass the component directly. With anyone else you render to HTML first, which is one extra line.
- Postmark's server-side templates (Mustache-style syntax, layouts, `POST /templates/validate`) [7] are a good choice if someone outside the codebase should edit copy. For a code-first app, templates in the repo are easier to review and test.

### Local dev and test mode
- **Resend:** send to `delivered@resend.dev` / `bounced@resend.dev` / `complained@resend.dev` to simulate each outcome; with a webhook configured, the bounce-handling path can be exercised end to end. Labels such as `bounced+signup@resend.dev` are supported [9]. Before a domain is verified you can send from `onboarding@resend.dev`, which is meant for testing only [6].
- **Postmark:** the `POSTMARK_API_TEST` token validates the payload without delivering anything [10]. This is ideal for CI, but it does not simulate bounces.
- **SES:** the mailbox simulator is the most complete (it adds an out-of-office case), works in the sandbox, and does not affect quota or reputation [12].
- For all of them, the app should keep a local "log/console transport" behind its own `sendEmail()` interface, so dev and tests never need network access.

### Bounce and complaint webhooks
- **Resend:** `email.bounced`, `email.complained`, `email.delivered`, `email.suppressed`, and more [13]. Each request is Svix-signed (`svix-id`/`svix-timestamp`/`svix-signature`) and verified against the raw body [14]. Use `svix-id` for idempotency.
- **Postmark:** full event set, but no HMAC signature verification. The options are basic auth in the URL and IP allowlisting [15]. That is a real weakness for a public endpoint.
- **SES:** events go to SNS (or Firehose through event publishing). Receiving them over HTTP means an SNS HTTPS subscription with a confirmation handshake and SNS signature checks [16]. This is the most moving parts of any option.
- **SendGrid:** ECDSA-signed Event Webhook, or OAuth [17].

### Domain verification
- **Resend:** add the DKIM and SPF records it generates (TXT, and MX or CNAME on the `send.` Return-Path subdomain). Do not proxy CNAMEs behind Cloudflare. Usually about 15 minutes, up to 72 h. Add DMARC afterwards [18].
- **Postmark:** one DKIM TXT record plus a Return-Path CNAME, verified within 48 h [19]. The account must also be approved before it can send to outside domains [10].
- **SES:** three DKIM CNAMEs (auto-published if DNS is on Route 53), an optional custom MAIL FROM, a separate identity per region, and a production-access request that AWS answers within 96 h [20][21].

## Recommendation

**Use Resend.**

1. **Cost:** $0 at Cadence's launch volume (3,000/mo), unlike Postmark ($15/mo minimum) or SendGrid (trial only). The 100/day cap is the thing to watch; Pro at $20/mo removes it.
2. **Best fit for a TypeScript/React codebase:** templates are React Email components that live in the repo, are type-checked, and can be unit-tested by rendering to HTML.
3. **Signed webhooks out of the box:** `resend.webhooks.verify()` makes bounce and complaint handling, which we need to suppress bad addresses, safe to expose publicly. Postmark has no signing.
4. **Test addresses simulate delivered, bounced, complained, and suppressed outcomes,** so bounce and complaint handling can be tested before launch.
5. **Fastest path to sending:** no account approval or sandbox exit, just DNS.

**Hedge against lock-in:** put sending behind a small app-level `EmailSender` interface (`send({ to, subject, react })`) and render React Email ourselves where practical. Switching to Postmark or SES later would then be one adapter.

**When to pick something else:**
- **Postmark** if deliverability reputation becomes the top concern and paying $15/mo is fine. Lock its webhook endpoint down with basic auth and an IP allowlist.
- **SES** if volume grows into hundreds of thousands a month and the team is already on AWS.

## Sources

1. Resend pricing: https://resend.com/pricing
2. Postmark pricing: https://postmarkapp.com/pricing
3. Amazon SES pricing: https://aws.amazon.com/ses/pricing/
4. Twilio SendGrid Email API pricing: https://www.twilio.com/en-us/products/email-api/pricing
5. Mailgun pricing: https://www.mailgun.com/pricing/
6. Resend, Send with Node.js: https://resend.com/docs/send-with-nodejs
7. Postmark Templates API: https://postmarkapp.com/developer/api/templates-api
8. React Email repository (maintainer, license, provider examples): https://github.com/resend/react-email
9. Resend, Send test emails: https://resend.com/docs/dashboard/emails/send-test-emails
10. Postmark API overview (test token, approval error codes 412/413): https://postmarkapp.com/developer/api/overview
11. Postmark sandbox mode: https://postmarkapp.com/developer/user-guide/sandbox-mode
12. Amazon SES mailbox simulator: https://docs.aws.amazon.com/ses/latest/dg/send-an-email-from-console.html
13. Resend webhook event types: https://resend.com/docs/dashboard/webhooks/event-types
14. Resend, Verify webhook requests: https://resend.com/docs/dashboard/webhooks/verify-webhooks-requests
15. Postmark webhooks overview: https://postmarkapp.com/developer/webhooks/webhooks-overview
16. Amazon SES event notifications: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
17. SendGrid Event Webhook security: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/getting-started-event-webhook-security-features
18. Resend, Add a domain: https://resend.com/docs/add-a-domain
19. Postmark, How do I verify a domain: https://postmarkapp.com/support/article/1046-how-do-i-verify-a-domain
20. Amazon SES, Creating and verifying identities: https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html
21. Amazon SES, Request production access: https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html
22. Resend, Domains introduction: https://resend.com/docs/dashboard/domains/introduction
23. React Email `render` utility: https://react.email/docs/utilities/render

Not verified against a first-party page: the SDK package names for SES, SendGrid and Mailgun; the SendGrid and Mailgun template and webhook details; and the SES EventBridge destination. None of these affects the recommendation.
