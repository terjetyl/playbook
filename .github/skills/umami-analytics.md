# Skill: Umami Analytics

## What this skill covers

Using the self-hosted Umami instance at `analytics.fjordid.eu` to understand user behaviour across FormVault and other apps.

## Setup

The tracking snippet goes in `index.html`. Do not add it conditionally — Umami itself excludes localhost automatically when using the hosted script. The `data-website-id` is unique per site registered in the Umami dashboard.

```html
<script
  defer
  src="https://analytics.fjordid.eu/script.js"
  data-website-id="<your-website-id>"
></script>
```

## Custom Events

Umami tracks pageviews automatically. For richer behaviour tracking, fire custom events anywhere in the app:

```ts
// Window method (always available after script loads)
window.umami?.track("form-created", { template: "contact", fieldCount: 4 });
window.umami?.track("form-published");
window.umami?.track("ai-generate-used");
window.umami?.track("submission-viewed");
window.umami?.track("csv-exported");
window.umami?.track("upgrade-prompt-shown", { trigger: "form-limit" });
```

The optional second argument is a flat object of properties — keep values as strings or numbers (no nested objects).

### Key events to instrument in FormVault

| Event name             | Where to fire                         | Properties                                                      |
| ---------------------- | ------------------------------------- | --------------------------------------------------------------- |
| `form-created`         | after POST /forms succeeds            | `{ method: 'manual' \| 'ai' \| 'template', template?: string }` |
| `form-published`       | when share modal opens for first time | `{ fieldCount: number }`                                        |
| `ai-generate-used`     | on AI generate submit                 | —                                                               |
| `ai-field-improve`     | on per-field improve click            | —                                                               |
| `submission-viewed`    | on submission dashboard load          | —                                                               |
| `csv-exported`         | on CSV export click                   | —                                                               |
| `webhook-configured`   | on webhook save                       | —                                                               |
| `member-invited`       | on team invite send                   | `{ role: string }`                                              |
| `upgrade-prompt-shown` | when plan limit hit                   | `{ trigger: string }`                                           |
| `plan-upgrade-clicked` | on upgrade CTA click                  | `{ plan: string }`                                              |

## React helper

Wrap the global call in a typed utility to avoid spreading `window.umami` calls everywhere:

```ts
// src/utils/analytics.ts
export function track(event: string, props?: Record<string, string | number>) {
  window.umami?.track(event, props);
}
```

Add the type to `vite-env.d.ts`:

```ts
interface Window {
  umami?: {
    track: (event: string, props?: Record<string, string | number>) => void;
  };
}
```

## Umami Dashboard — What to look for

### Pages report

- Which pages have the most views? (`/app/forms` vs `/app/builder/:id` vs `/pricing`)
- High `/pricing` visits with low signups → pricing page needs work
- High builder visits but low form publishes → builder UX friction

### Events report

- `form-created` / `form-published` ratio → how many forms are drafted but never published
- `ai-generate-used` trend → is the AI feature being discovered and used
- `upgrade-prompt-shown` vs `plan-upgrade-clicked` → upgrade conversion rate

### Referrers report

- Where is traffic coming from? Direct, search, social?

### UTM tracking

Umami automatically captures UTM params (`utm_source`, `utm_campaign` etc.) on any URL. Add them to outbound links in marketing emails or social posts:

```
https://formvault.eu?utm_source=email&utm_campaign=weekly-digest&utm_medium=email
```

## Umami API (programmatic access)

If you need to pull data into the app (e.g. an admin dashboard):

```ts
// 1. Get a token
const { token } = await fetch("https://analytics.fjordid.eu/api/auth/login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    username: "admin",
    password: process.env.UMAMI_PASSWORD,
  }),
}).then((r) => r.json());

// 2. Fetch pageviews for a site
const stats = await fetch(
  "https://analytics.fjordid.eu/api/websites/<websiteId>/stats?startAt=0&endAt=" +
    Date.now(),
  { headers: { Authorization: `Bearer ${token}` } },
).then((r) => r.json());
```

## Privacy / GDPR notes

- Umami does **not** use cookies and does **not** track individuals — GDPR-compliant by design
- No cookie consent banner needed for Umami (unlike GA4)
- Data stays on `analytics.fjordid.eu` (EU) — no US transfer
- Do **not** send personally identifiable data (email, name) as event properties
