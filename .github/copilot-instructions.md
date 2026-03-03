<skills>
Here is a list of skills that contain domain-specific knowledge for this codebase.
Each skill comes with a description of the topic and a file path that contains the detailed instructions.
When a user asks you to perform a task that falls within the domain of a skill, use the 'read_file' tool to acquire the full instructions from the file URI.

<skill>
<name>drizzle-migrations</name>
<description>Generating, applying, and rolling back Drizzle ORM migrations. Covers the full workflow (schema change → generate → review → commit), common failure modes (hash mismatch, relation already exists, column missing), idempotency patterns, jsonb column changes, and rollback strategy. USE FOR: generate migration, apply migration, rollback migration, drizzle migrate, schema change, add column, migration error, migration hash mismatch, db:generate, drizzle journal. DO NOT USE FOR: writing raw SQL queries, database design questions unrelated to migrations.</description>
<file>/Users/terjetyldum/.agents/skills/drizzle-migrations/SKILL.md</file>
</skill>

<skill>
<name>app-tester</name>
<description>Vitest unit and integration test patterns for this stack. Covers test structure, encrypted form submission testing, auth mocking, test database setup, and integration test best practices. USE FOR: write unit test, write integration test, vitest setup, mock auth, test encrypted submission, test API endpoint, test database, test helper, testing patterns. DO NOT USE FOR: E2E browser tests (use playwright-testing), load testing.</description>
<file>/Users/terjetyldum/.agents/skills/app-tester/SKILL.md</file>
</skill>

<skill>
<name>playwright-testing</name>
<description>Playwright end-to-end test setup and patterns. Covers auth state setup and reuse, page object model, critical user path coverage, CI configuration, and debugging flaky tests. USE FOR: playwright test, E2E test, browser test, page object, auth fixture, playwright config, CI playwright, test critical path, end-to-end. DO NOT USE FOR: unit tests or integration tests (use app-tester).</description>
<file>/Users/terjetyldum/.agents/skills/playwright-testing/SKILL.md</file>
</skill>

<skill>
<name>ui-ux-responsive</name>
<description>Responsive, mobile-first UI/UX patterns for this stack. Covers mobile-first CSS, touch target sizing, modal patterns, navigation patterns, and responsive table strategies. USE FOR: mobile layout, responsive design, touch target, modal pattern, nav pattern, responsive table, mobile-first CSS, breakpoint, accessibility sizing. DO NOT USE FOR: backend API design, database schema.</description>
<file>/Users/terjetyldum/.agents/skills/ui-ux-responsive/SKILL.md</file>
</skill>

<skill>
<name>umami-analytics</name>
<description>Umami analytics integration. Covers tracking custom events from the frontend, interpreting the Umami dashboard, and accessing Umami data via its API. USE FOR: track event, custom analytics, umami event, umami dashboard, umami API, page views, conversion tracking, analytics. DO NOT USE FOR: Google Analytics, other analytics platforms.</description>
<file>/Users/terjetyldum/.agents/skills/umami-analytics/SKILL.md</file>
</skill>

<skill>
<name>prod-server-access</name>
<description>Production server operations on the shared Hetzner VPS. Covers SSH access, passwordless sudo setup, tailing logs, running docker exec commands, and safe debugging patterns. USE FOR: SSH to production, tail logs, docker exec, production debugging, server access, prod server, log tailing, restart container, check container status. DO NOT USE FOR: local development setup, CI/CD pipeline configuration.</description>
<file>/Users/terjetyldum/.agents/skills/prod-server-access/SKILL.md</file>
</skill>

<skill>
<name>gh-cli</name>
<description>GitHub CLI (gh) commands for day-to-day repo management. Covers authentication, PR lifecycle (create, review, merge, checks), issue management, triggering and watching workflow runs, managing secrets and variables, and creating releases. USE FOR: gh pr create, gh pr merge, gh run watch, gh secret set, gh release create, gh workflow run, gh issue, gh auth, github cli, trigger workflow, set secret, create release. DO NOT USE FOR: git operations (use git directly), GitHub web UI navigation.</description>
<file>/Users/terjetyldum/.agents/skills/gh-cli/SKILL.md</file>
</skill>

</skills>
