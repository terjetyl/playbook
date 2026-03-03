# Skill: Playwright E2E Testing

## Stack context

- **Framework**: Playwright with TypeScript
- **App under test**: React/Vite SPA (`apps/forms-web`) + Fastify API (`apps/forms-api`)
- **Auth**: FjordID (Keycloak) — PKCE S256 flow
- **Config**: `apps/forms-web/playwright.config.ts`
- **Test directory**: `apps/forms-web/e2e/`

## Running tests

```bash
# Install browsers (first time)
pnpm --filter @formvault/web exec playwright install

# Run all E2E tests (requires dev server running)
pnpm --filter @formvault/web test:e2e

# Run a specific file
pnpm --filter @formvault/web exec playwright test e2e/form-player.spec.ts

# Run with UI (interactive mode, great for debugging)
pnpm --filter @formvault/web exec playwright test --ui

# Run headed (watch the browser)
pnpm --filter @formvault/web exec playwright test --headed

# Debug a specific test
pnpm --filter @formvault/web exec playwright test --debug e2e/form-player.spec.ts
```

## Handling auth in tests

FjordID uses PKCE — you can't fake a JWT client-side. Two approaches:

### Option 1: storageState (recommended for protected pages)

Log in once, save browser storage, reuse across tests:

```ts
// e2e/fixtures/auth.setup.ts
import { test as setup } from "@playwright/test";

setup("authenticate", async ({ page }) => {
  await page.goto("/");
  // Wait for Keycloak redirect
  await page.waitForURL(/auth\.fjordid\.eu/);
  await page.fill("#username", process.env.E2E_USER!);
  await page.fill("#password", process.env.E2E_PASSWORD!);
  await page.click("#kc-login");
  await page.waitForURL("/app/forms");
  await page.context().storageState({ path: "e2e/.auth/user.json" });
});
```

```ts
// playwright.config.ts
export default defineConfig({
  projects: [
    { name: "setup", testMatch: /.*\.setup\.ts/ },
    {
      name: "authenticated",
      use: { storageState: "e2e/.auth/user.json" },
      dependencies: ["setup"],
    },
  ],
});
```

### Option 2: Public form tests (no auth needed)

The public form player (`/form/:formId`) needs no auth — test it directly:

```ts
test("can submit a public form", async ({ page }) => {
  await page.goto(`/form/${FIXTURE_FORM_ID}`);
  await page.fill('[placeholder="Jane Smith"]', "Test User");
  await page.fill('[type="email"]', "test@example.com");
  await page.click('button[type="submit"]');
  await expect(page.getByText("Thank you")).toBeVisible();
});
```

## Page Object pattern

Keep selectors out of tests:

```ts
// e2e/pages/FormBuilderPage.ts
import { Page, Locator } from "@playwright/test";

export class FormBuilderPage {
  readonly page: Page;
  readonly addFieldButton: Locator;
  readonly saveButton: Locator;
  readonly previewPane: Locator;

  constructor(page: Page) {
    this.page = page;
    this.addFieldButton = page.getByRole("button", { name: "Add field" });
    this.saveButton = page.getByRole("button", { name: "Save" });
    this.previewPane = page.locator('[data-testid="preview-pane"]');
  }

  async addTextField(label: string) {
    await this.addFieldButton.click();
    await this.page.getByRole("option", { name: "Text" }).click();
    await this.page.getByLabel("Field label").fill(label);
  }

  async save() {
    await this.saveButton.click();
    await this.page.getByText("Saved").waitFor();
  }
}
```

## Critical paths to always test

```ts
// e2e/critical-paths.spec.ts

test("create a form and submit it", async ({ page }) => {
  // 1. Create form in builder
  await page.goto("/app/forms/new");
  // ... add fields, save

  // 2. Get the public form URL from Share modal
  await page.getByRole("button", { name: "Share" }).click();
  const formUrl = await page.getByLabel("Form link").inputValue();

  // 3. Submit the public form
  await page.goto(formUrl);
  // ... fill and submit

  // 4. Verify submission appears in dashboard
  await page.goto("/app/forms");
  // ...
});

test("returns 401 for unauthenticated API request", async ({ request }) => {
  const res = await request.get("/api/forms/some-tenant-id");
  expect(res.status()).toBe(401);
});
```

## Waiting strategies — never use `page.waitForTimeout`

```ts
// BAD — brittle, slow
await page.waitForTimeout(2000);

// GOOD — wait for a specific condition
await page.waitForURL("/app/forms");
await expect(page.getByText("Form saved")).toBeVisible();
await page.waitForLoadState("networkidle");
await page.waitForResponse(
  (res) => res.url().includes("/api/forms") && res.status() === 200,
);
```

## API testing with Playwright request context

```ts
test("API: create and fetch form", async ({ request }) => {
  const token = process.env.E2E_API_TOKEN!; // fv_... key

  const created = await request.post("/api/v1/forms", {
    headers: { Authorization: `Bearer ${token}` },
    data: { name: "Test Form", fields: [] },
  });
  expect(created.status()).toBe(201);
  const { id } = await created.json();

  const fetched = await request.get(`/api/v1/forms/${id}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  expect(fetched.status()).toBe(200);
});
```

## Data test IDs

Prefer `data-testid` attributes over text content or CSS classes — they survive refactoring:

```tsx
<button data-testid="save-form">Save</button>
```

```ts
await page.locator('[data-testid="save-form"]').click();
// or:
await page.getByTestId("save-form").click();
```

## CI configuration

Playwright runs in CI against the production build. The `playwright.config.ts` should use `webServer` to spin up the dev server automatically:

```ts
webServer: {
  command: 'pnpm dev',
  url: 'http://localhost:5173',
  reuseExistingServer: !process.env.CI,
  timeout: 120_000,
},
```

Add to `.github/workflows/ci.yml`:

```yaml
- name: Install Playwright browsers
  run: pnpm --filter @formvault/web exec playwright install --with-deps chromium

- name: Run E2E tests
  run: pnpm --filter @formvault/web test:e2e
  env:
    E2E_USER: ${{ secrets.E2E_USER }}
    E2E_PASSWORD: ${{ secrets.E2E_PASSWORD }}
```

## Debugging failures

```bash
# Show trace viewer for a failed test
pnpm --filter @formvault/web exec playwright show-trace test-results/.../trace.zip

# Run with verbose logging
DEBUG=pw:api pnpm --filter @formvault/web exec playwright test
```
