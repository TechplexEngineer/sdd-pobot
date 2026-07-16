# PO Bot Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a deployable PO Bot skeleton — SvelteKit on Cloudflare Workers, the full D1 schema, Google OAuth admin login gated by an allowlist, and a public read-only board — so every later phase (Slack intake, email ingestion, tracking, inventory) has infrastructure to build on.

**Architecture:** SvelteKit with `@sveltejs/adapter-cloudflare`, deployed as a Worker with static assets. D1 holds all relational data via a single migration covering the full schema from the design spec. Admin auth is a hand-rolled Google OAuth 2.0 authorization-code flow (no library dependency) plus an HMAC-signed session cookie, checked against an `admins` allowlist table on every `/admin/*` request. The public board at `/` requires no auth.

**Tech Stack:** SvelteKit (Svelte 5, TypeScript), `@sveltejs/adapter-cloudflare`, Cloudflare D1, Vitest + `@cloudflare/vitest-pool-workers` for tests that touch Workers runtime APIs (D1, Web Crypto).

## Global Constraints

- SvelteKit deployed to Cloudflare Workers via `adapter-cloudflare` (per spec Architecture & Tech Stack).
- D1 (SQLite) is the only datastore for relational data (per spec Architecture & Tech Stack).
- Google OAuth authenticates admin users against the `admins` allowlist table; the public board requires no login (per spec Architecture & Tech Stack, Dashboard).
- All application code is TypeScript.
- The full D1 schema (10 tables) is defined now, in this phase, even though later phases populate most of it — the schema is already fully specified and approved in the design spec, so building it once avoids repeated migration churn across phases.

---

### Task 1: Project Scaffold & Cloudflare Adapter

**Files:**
- Create: `package.json`, `svelte.config.js`, `vite.config.ts`, `tsconfig.json`, `src/app.html`, `src/routes/+page.svelte`, `.gitignore` (modify existing scaffolded one)
- Create: `wrangler.jsonc`

**Interfaces:**
- Produces: a running `npm run dev` dev server and a deployable Worker build at `.svelte-kit/cloudflare/_worker.js`. No application code yet — later tasks depend on this scaffold existing.

- [ ] **Step 1: Scaffold the SvelteKit project**

Run from the project root (`/Users/techplex/projects/sdd-pobot`, which already contains `.claude/` and `docs/`, so directory-emptiness checks must be skipped):

```bash
npx sv create . --template minimal --types ts --no-add-ons --install npm --no-dir-check
```

- [ ] **Step 2: Install the Cloudflare adapter and dev dependencies**

```bash
npm install -D @sveltejs/adapter-cloudflare @cloudflare/workers-types wrangler
```

- [ ] **Step 3: Configure `svelte.config.js` to use the Cloudflare adapter**

Replace the contents of `svelte.config.js` with:

```js
import adapter from '@sveltejs/adapter-cloudflare';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

export default {
	preprocess: vitePreprocess(),
	kit: {
		adapter: adapter({
			platformProxy: {
				configPath: undefined,
				environment: undefined,
				persist: undefined
			}
		})
	}
};
```

- [ ] **Step 4: Write `wrangler.jsonc`**

Create `wrangler.jsonc` in the project root:

```jsonc
{
	"$schema": "./node_modules/wrangler/config-schema.json",
	"name": "po-bot",
	"main": ".svelte-kit/cloudflare/_worker.js",
	"compatibility_date": "2026-07-16",
	"compatibility_flags": ["nodejs_als"],
	"assets": {
		"binding": "ASSETS",
		"directory": ".svelte-kit/cloudflare"
	}
}
```

- [ ] **Step 5: Add Cloudflare-specific ignores**

Append to `.gitignore` (created by `sv create`) if not already present:

```
.wrangler
.dev.vars
.dev.vars.*
```

- [ ] **Step 6: Add `deploy` script to `package.json`**

In the `"scripts"` section of `package.json`, add:

```json
"deploy": "npm run build && wrangler deploy"
```

- [ ] **Step 7: Verify the dev server boots**

```bash
npm run dev &
sleep 3
curl -s -o /dev/null -w "%{http_code}" http://localhost:5173
kill %1
```

Expected: `200`

- [ ] **Step 8: Commit**

```bash
git add package.json package-lock.json svelte.config.js vite.config.ts tsconfig.json wrangler.jsonc .gitignore src
git commit -m "Scaffold SvelteKit project with Cloudflare Workers adapter"
```

---

### Task 2: D1 Schema Migration

**Files:**
- Create: `migrations/0001_initial_schema.sql`
- Create: `vitest.config.ts`
- Create: `test/apply-migrations.ts`
- Create: `test/tsconfig.json`
- Create: `src/lib/server/schema.test.ts`
- Modify: `wrangler.jsonc` (add `d1_databases`)
- Modify: `package.json` (add `test` script)

**Interfaces:**
- Produces: a D1 binding named `DB`, and a fully migrated schema with tables `vendors`, `settings`, `requests`, `orders`, `order_line_items`, `shipments`, `webhook_logs`, `admins`, `inventory_items`, `inventory_history`. All later tasks in this and future phases query through this schema.

- [ ] **Step 1: Create the D1 database (requires your own Cloudflare account)**

```bash
npx wrangler login
npx wrangler d1 create po-bot-db
```

Copy the `database_id` from the command output — you'll need it in the next step.

- [ ] **Step 2: Add the D1 binding to `wrangler.jsonc`**

Add this key to `wrangler.jsonc` (replace `<DATABASE_ID>` with the value from Step 1):

```jsonc
	"d1_databases": [
		{
			"binding": "DB",
			"database_name": "po-bot-db",
			"database_id": "<DATABASE_ID>",
			"migrations_dir": "./migrations"
		}
	]
```

- [ ] **Step 3: Generate the migration file**

```bash
npx wrangler d1 migrations create po-bot-db initial_schema
```

This creates `migrations/0001_initial_schema.sql`, empty apart from a header comment.

- [ ] **Step 4: Install test dependencies**

```bash
npm install -D vitest @cloudflare/vitest-pool-workers
```

- [ ] **Step 5: Write `test/apply-migrations.ts`**

```ts
import { applyD1Migrations } from 'cloudflare:test';
import { env } from 'cloudflare:workers';

await applyD1Migrations(env.DB, env.TEST_MIGRATIONS);
```

- [ ] **Step 6: Write `test/tsconfig.json`**

```jsonc
{
	"extends": "../tsconfig.json",
	"compilerOptions": {
		"moduleResolution": "bundler",
		"types": ["@cloudflare/vitest-pool-workers/types"]
	},
	"include": ["./**/*.ts", "../src/**/*.ts"]
}
```

- [ ] **Step 7: Write `vitest.config.ts`**

```ts
import path from 'node:path';
import { cloudflareTest, readD1Migrations } from '@cloudflare/vitest-pool-workers';
import { defineConfig } from 'vitest/config';

export default defineConfig(async () => ({
	plugins: [
		cloudflareTest({
			wrangler: { configPath: './wrangler.jsonc' },
			miniflare: {
				bindings: {
					TEST_MIGRATIONS: await readD1Migrations(path.join(__dirname, 'migrations'))
				}
			}
		})
	],
	test: {
		include: ['src/lib/server/**/*.test.ts'],
		setupFiles: ['./test/apply-migrations.ts']
	}
}));
```

- [ ] **Step 8: Add the `test` script to `package.json`**

```json
"test": "vitest run"
```

- [ ] **Step 9: Write the failing schema test**

Create `src/lib/server/schema.test.ts`:

```ts
import { env } from 'cloudflare:workers';
import { describe, it, expect } from 'vitest';

const EXPECTED_TABLES = [
	'vendors',
	'settings',
	'requests',
	'orders',
	'order_line_items',
	'shipments',
	'webhook_logs',
	'admins',
	'inventory_items',
	'inventory_history'
];

describe('D1 schema', () => {
	it('creates every table defined in the design spec', async () => {
		const { results } = await env.DB.prepare(
			"SELECT name FROM sqlite_master WHERE type = 'table' AND name NOT LIKE 'sqlite_%' AND name NOT LIKE '_cf_%' AND name != 'd1_migrations'"
		).all<{ name: string }>();
		const tableNames = results.map((row) => row.name).sort();
		expect(tableNames).toEqual([...EXPECTED_TABLES].sort());
	});
});
```

- [ ] **Step 10: Run the test and verify it fails**

```bash
npm test
```

Expected: FAIL — `migrations/0001_initial_schema.sql` is still empty, so `tableNames` is `[]`, not the 10 expected tables.

- [ ] **Step 11: Write the full schema into the migration file**

Replace the contents of `migrations/0001_initial_schema.sql` with:

```sql
CREATE TABLE vendors (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	name TEXT NOT NULL,
	domain_patterns TEXT,
	part_number_regex TEXT,
	notes TEXT
);

CREATE TABLE settings (
	key TEXT PRIMARY KEY,
	value TEXT NOT NULL,
	updated_at TEXT NOT NULL
);

CREATE TABLE requests (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	slack_user_id TEXT NOT NULL,
	slack_thread_ts TEXT NOT NULL,
	raw_text TEXT NOT NULL,
	item_name TEXT NOT NULL,
	part_number TEXT,
	link TEXT,
	quantity INTEGER NOT NULL,
	estimated_price REAL,
	status TEXT NOT NULL CHECK (status IN ('requested', 'ordered', 'cancelled')),
	llm_confidence REAL NOT NULL,
	created_at TEXT NOT NULL,
	ordered_at TEXT,
	delivered_at TEXT
);

CREATE TABLE orders (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	vendor_id INTEGER REFERENCES vendors(id),
	gmail_message_id TEXT NOT NULL UNIQUE,
	order_number TEXT,
	order_date TEXT,
	raw_email_snapshot TEXT NOT NULL,
	parse_status TEXT NOT NULL CHECK (parse_status IN ('parsed', 'low_confidence', 'manual')),
	created_at TEXT NOT NULL
);

CREATE TABLE order_line_items (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	order_id INTEGER NOT NULL REFERENCES orders(id),
	item_name TEXT NOT NULL,
	part_number TEXT,
	quantity INTEGER NOT NULL,
	price REAL,
	matched_request_id INTEGER REFERENCES requests(id),
	added_manually INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE shipments (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	order_id INTEGER NOT NULL REFERENCES orders(id),
	tracking_number TEXT NOT NULL,
	carrier TEXT,
	easypost_tracker_id TEXT,
	status TEXT NOT NULL DEFAULT 'unknown',
	estimated_delivery TEXT,
	last_event_at TEXT,
	added_manually INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE webhook_logs (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	source TEXT NOT NULL,
	event_type TEXT NOT NULL,
	related_shipment_id INTEGER REFERENCES shipments(id),
	related_order_id INTEGER REFERENCES orders(id),
	received_at TEXT NOT NULL,
	request_payload TEXT NOT NULL,
	response_status INTEGER,
	response_payload TEXT,
	processing_status TEXT NOT NULL CHECK (processing_status IN ('success', 'error', 'ignored')),
	error_message TEXT,
	processing_duration_ms INTEGER
);

CREATE TABLE admins (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	google_email TEXT NOT NULL UNIQUE
);

CREATE TABLE inventory_items (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	part_number TEXT UNIQUE,
	item_name TEXT NOT NULL,
	quantity_on_hand INTEGER NOT NULL DEFAULT 0,
	created_at TEXT NOT NULL,
	updated_at TEXT NOT NULL
);

CREATE TABLE inventory_history (
	id INTEGER PRIMARY KEY AUTOINCREMENT,
	inventory_item_id INTEGER NOT NULL REFERENCES inventory_items(id),
	change_type TEXT NOT NULL CHECK (change_type IN ('delivery', 'cycle_count_set', 'cycle_count_delta')),
	quantity_before INTEGER NOT NULL,
	quantity_after INTEGER NOT NULL,
	delta INTEGER NOT NULL,
	source_request_id INTEGER REFERENCES requests(id),
	admin_google_email TEXT,
	note TEXT,
	created_at TEXT NOT NULL
);

CREATE INDEX idx_requests_status ON requests(status);
CREATE INDEX idx_order_line_items_order_id ON order_line_items(order_id);
CREATE INDEX idx_order_line_items_matched_request_id ON order_line_items(matched_request_id);
CREATE INDEX idx_shipments_order_id ON shipments(order_id);
CREATE INDEX idx_inventory_history_inventory_item_id ON inventory_history(inventory_item_id);
```

- [ ] **Step 12: Run the test and verify it passes**

```bash
npm test
```

Expected: `schema.test.ts` passes, confirming all 10 tables exist.

- [ ] **Step 13: Apply the migration to your local `wrangler dev` database**

This is a separate storage context from the isolated D1 instance the test above just ran against — `wrangler dev` and `wrangler d1 execute --local` read from `.wrangler/state` on disk, so they need the migration applied here too:

```bash
npx wrangler d1 migrations apply po-bot-db --local
```

Expected: output confirms `0001_initial_schema.sql` applied.

- [ ] **Step 14: Commit**

```bash
git add migrations wrangler.jsonc vitest.config.ts test package.json src/lib/server/schema.test.ts
git commit -m "Add D1 schema migration and Workers-runtime test setup"
```

---

### Task 3: Public Board Page

**Files:**
- Create: `src/lib/server/db.ts`
- Create: `src/lib/server/db.test.ts`
- Modify: `src/routes/+page.server.ts` (create if `sv create` didn't scaffold one)
- Modify: `src/routes/+page.svelte`
- Create: `src/app.d.ts`

**Interfaces:**
- Consumes: D1 binding `DB` from Task 2's schema (table `requests`).
- Produces: `getBoardRequests(db: D1Database): Promise<BoardRequest[]>` in `src/lib/server/db.ts`, and the `BoardRequest` type — Task 5 (Google OAuth) will add `isAdmin` to this same file.

- [ ] **Step 1: Write the failing test**

Create `src/lib/server/db.test.ts`:

```ts
import { env } from 'cloudflare:workers';
import { describe, it, expect, beforeEach } from 'vitest';
import { getBoardRequests } from './db';

async function insertRequest(overrides: Partial<{ item_name: string; status: string }> = {}) {
	await env.DB.prepare(
		`INSERT INTO requests (slack_user_id, slack_thread_ts, raw_text, item_name, quantity, status, llm_confidence, created_at)
		 VALUES (?, ?, ?, ?, ?, ?, ?, ?)`
	)
		.bind(
			'U123',
			'111.222',
			'need a gear',
			overrides.item_name ?? '60 tooth gear',
			5,
			overrides.status ?? 'requested',
			0.92,
			new Date().toISOString()
		)
		.run();
}

describe('getBoardRequests', () => {
	beforeEach(async () => {
		await env.DB.exec('DELETE FROM requests');
	});

	it('returns requests that are not cancelled', async () => {
		await insertRequest({ item_name: '60 tooth gear', status: 'requested' });
		await insertRequest({ item_name: 'bearing', status: 'cancelled' });

		const results = await getBoardRequests(env.DB);

		expect(results).toHaveLength(1);
		expect(results[0].item_name).toBe('60 tooth gear');
	});
});
```

- [ ] **Step 2: Run the test and verify it fails**

```bash
npm test
```

Expected: FAIL — `./db` has no exported member `getBoardRequests`.

- [ ] **Step 3: Implement `src/lib/server/db.ts`**

```ts
import type { D1Database } from '@cloudflare/workers-types';

export interface BoardRequest {
	id: number;
	item_name: string;
	quantity: number;
	part_number: string | null;
	link: string | null;
	slack_user_id: string;
	status: 'requested' | 'ordered' | 'cancelled';
}

export async function getBoardRequests(db: D1Database): Promise<BoardRequest[]> {
	const { results } = await db
		.prepare(
			`SELECT id, item_name, quantity, part_number, link, slack_user_id, status
			 FROM requests
			 WHERE status != 'cancelled'
			 ORDER BY created_at ASC`
		)
		.all<BoardRequest>();
	return results;
}
```

- [ ] **Step 4: Run the test and verify it passes**

```bash
npm test
```

Expected: PASS

- [ ] **Step 5: Declare the `Platform` type**

Create `src/app.d.ts`:

```ts
import type { D1Database } from '@cloudflare/workers-types';

declare global {
	namespace App {
		interface Platform {
			env: {
				DB: D1Database;
				GOOGLE_CLIENT_ID: string;
				GOOGLE_CLIENT_SECRET: string;
				SESSION_SECRET: string;
			};
		}
	}
}

export {};
```

- [ ] **Step 6: Wire up `src/routes/+page.server.ts`**

```ts
import type { PageServerLoad } from './$types';
import { getBoardRequests } from '$lib/server/db';

export const load: PageServerLoad = async ({ platform }) => {
	const requests = await getBoardRequests(platform!.env.DB);
	return { requests };
};
```

- [ ] **Step 7: Replace `src/routes/+page.svelte`**

```svelte
<script lang="ts">
	import type { PageData } from './$types';

	let { data }: { data: PageData } = $props();

	const columns = ['requested', 'ordered', 'shipped', 'delivered'] as const;
	const columnLabels: Record<(typeof columns)[number], string> = {
		requested: 'Requested',
		ordered: 'Ordered',
		shipped: 'Shipped',
		delivered: 'Delivered'
	};

	function itemsFor(column: (typeof columns)[number]) {
		if (column === 'shipped' || column === 'delivered') return [];
		return data.requests.filter((r) => r.status === column);
	}
</script>

<h1>PO Bot</h1>

<div class="board">
	{#each columns as column (column)}
		<section>
			<h2>{columnLabels[column]}</h2>
			{#if itemsFor(column).length === 0}
				<p>No items yet.</p>
			{:else}
				<ul>
					{#each itemsFor(column) as request (request.id)}
						<li>{request.quantity}x {request.item_name}</li>
					{/each}
				</ul>
			{/if}
		</section>
	{/each}
</div>

<style>
	.board {
		display: grid;
		grid-template-columns: repeat(4, 1fr);
		gap: 1rem;
	}
</style>
```

- [ ] **Step 8: Verify locally with `wrangler dev`**

```bash
npm run build
npx wrangler dev &
sleep 3
curl -s http://localhost:8787 | grep -o "PO Bot"
kill %1
```

Expected: `PO Bot`

- [ ] **Step 9: Commit**

```bash
git add src/lib/server/db.ts src/lib/server/db.test.ts src/routes/+page.server.ts src/routes/+page.svelte src/app.d.ts
git commit -m "Add public read-only board page backed by D1"
```

---

### Task 4: Session Token Helpers

**Files:**
- Create: `src/lib/server/session.ts`
- Create: `src/lib/server/session.test.ts`

**Interfaces:**
- Produces: `signSession(payload: SessionPayload, secret: string): Promise<string>`, `verifySession(token: string, secret: string): Promise<SessionPayload | null>`, and the `SessionPayload` type (`{ email: string; exp: number }`) — consumed by Task 5 (OAuth callback) and Task 6 (admin route guard).

- [ ] **Step 1: Write the failing tests**

Create `src/lib/server/session.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { signSession, verifySession } from './session';

const SECRET = 'test-secret-do-not-use-in-prod';

describe('signSession / verifySession', () => {
	it('round-trips a valid, unexpired session', async () => {
		const exp = Math.floor(Date.now() / 1000) + 3600;
		const token = await signSession({ email: 'mentor@example.com', exp }, SECRET);
		const payload = await verifySession(token, SECRET);

		expect(payload).toEqual({ email: 'mentor@example.com', exp });
	});

	it('rejects a token signed with a different secret', async () => {
		const exp = Math.floor(Date.now() / 1000) + 3600;
		const token = await signSession({ email: 'mentor@example.com', exp }, SECRET);
		const payload = await verifySession(token, 'wrong-secret');

		expect(payload).toBeNull();
	});

	it('rejects a tampered payload', async () => {
		const exp = Math.floor(Date.now() / 1000) + 3600;
		const token = await signSession({ email: 'mentor@example.com', exp }, SECRET);
		const [body, signature] = token.split('.');
		const tamperedBody = body.slice(0, -1) + (body.at(-1) === 'A' ? 'B' : 'A');
		const payload = await verifySession(`${tamperedBody}.${signature}`, SECRET);

		expect(payload).toBeNull();
	});

	it('rejects an expired session', async () => {
		const exp = Math.floor(Date.now() / 1000) - 10;
		const token = await signSession({ email: 'mentor@example.com', exp }, SECRET);
		const payload = await verifySession(token, SECRET);

		expect(payload).toBeNull();
	});
});
```

- [ ] **Step 2: Run the tests and verify they fail**

```bash
npm test
```

Expected: FAIL — `./session` has no exported members `signSession`/`verifySession`.

- [ ] **Step 3: Implement `src/lib/server/session.ts`**

```ts
export interface SessionPayload {
	email: string;
	exp: number;
}

async function hmacKey(secret: string): Promise<CryptoKey> {
	return crypto.subtle.importKey(
		'raw',
		new TextEncoder().encode(secret),
		{ name: 'HMAC', hash: 'SHA-256' },
		false,
		['sign', 'verify']
	);
}

function base64urlEncode(bytes: ArrayBuffer): string {
	const arr = new Uint8Array(bytes);
	let str = '';
	for (const byte of arr) str += String.fromCharCode(byte);
	return btoa(str).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

function base64urlDecode(value: string): Uint8Array {
	const padded = value.replace(/-/g, '+').replace(/_/g, '/') + '='.repeat((4 - (value.length % 4)) % 4);
	const bin = atob(padded);
	const bytes = new Uint8Array(bin.length);
	for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
	return bytes;
}

export async function signSession(payload: SessionPayload, secret: string): Promise<string> {
	const key = await hmacKey(secret);
	const body = base64urlEncode(new TextEncoder().encode(JSON.stringify(payload)).buffer as ArrayBuffer);
	const signatureBuf = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(body));
	return `${body}.${base64urlEncode(signatureBuf)}`;
}

export async function verifySession(token: string, secret: string): Promise<SessionPayload | null> {
	const parts = token.split('.');
	if (parts.length !== 2) return null;
	const [body, signature] = parts;

	const key = await hmacKey(secret);
	const valid = await crypto.subtle.verify(
		'HMAC',
		key,
		base64urlDecode(signature),
		new TextEncoder().encode(body)
	);
	if (!valid) return null;

	const payload = JSON.parse(new TextDecoder().decode(base64urlDecode(body))) as SessionPayload;
	if (payload.exp < Math.floor(Date.now() / 1000)) return null;
	return payload;
}
```

- [ ] **Step 4: Run the tests and verify they pass**

```bash
npm test
```

Expected: PASS (all 4 cases)

- [ ] **Step 5: Commit**

```bash
git add src/lib/server/session.ts src/lib/server/session.test.ts
git commit -m "Add HMAC-signed session token helpers"
```

---

### Task 5: Google OAuth Login Flow

**Files:**
- Create: `src/lib/server/googleOAuth.ts`
- Create: `src/lib/server/googleOAuth.test.ts`
- Modify: `src/lib/server/db.ts` (add `isAdmin`)
- Modify: `src/lib/server/db.test.ts` (add `isAdmin` tests)
- Create: `src/routes/auth/login/+server.ts`
- Create: `src/routes/auth/callback/+server.ts`
- Create: `.dev.vars.example`

**Interfaces:**
- Consumes: `signSession` from Task 4, `D1Database` binding from Task 2.
- Produces: `isAdmin(db: D1Database, email: string): Promise<boolean>` in `db.ts` — consumed by Task 6's admin route guard is NOT required (the guard only checks the session cookie; `isAdmin` is checked once, here, at login time). Routes `GET /auth/login` and `GET /auth/callback`.

**Prerequisite (manual, one-time): create a Google OAuth client**

1. In [Google Cloud Console](https://console.cloud.google.com/apis/credentials), create an OAuth 2.0 Client ID of type "Web application".
2. Add authorized redirect URIs: `http://localhost:8787/auth/callback` (local dev) and `https://<your-worker-subdomain>.workers.dev/auth/callback` (production — add once you know your deployed URL from Task 7).
3. Note the generated Client ID and Client Secret.

- [ ] **Step 1: Write the failing `db.ts` test for `isAdmin`**

Append to `src/lib/server/db.test.ts`:

```ts
import { isAdmin } from './db';

describe('isAdmin', () => {
	beforeEach(async () => {
		await env.DB.exec('DELETE FROM admins');
	});

	it('returns false for an email not in the admins table', async () => {
		expect(await isAdmin(env.DB, 'nobody@example.com')).toBe(false);
	});

	it('returns true for an email in the admins table', async () => {
		await env.DB.prepare('INSERT INTO admins (google_email) VALUES (?)').bind('mentor@example.com').run();
		expect(await isAdmin(env.DB, 'mentor@example.com')).toBe(true);
	});
});
```

- [ ] **Step 2: Run the test and verify it fails**

```bash
npm test
```

Expected: FAIL — `./db` has no exported member `isAdmin`.

- [ ] **Step 3: Add `isAdmin` to `src/lib/server/db.ts`**

Append to `src/lib/server/db.ts`:

```ts
export async function isAdmin(db: D1Database, email: string): Promise<boolean> {
	const row = await db.prepare('SELECT 1 FROM admins WHERE google_email = ?').bind(email).first();
	return row !== null;
}
```

- [ ] **Step 4: Run the test and verify it passes**

```bash
npm test
```

Expected: PASS

- [ ] **Step 5: Write the failing `googleOAuth.ts` tests**

Create `src/lib/server/googleOAuth.test.ts`:

```ts
import { describe, it, expect, vi, afterEach } from 'vitest';
import { buildGoogleAuthorizeUrl, exchangeGoogleCode, fetchGoogleUserInfo } from './googleOAuth';

describe('buildGoogleAuthorizeUrl', () => {
	it('builds a Google authorization URL with the given params', () => {
		const url = new URL(
			buildGoogleAuthorizeUrl({
				clientId: 'client-123',
				redirectUri: 'https://pobot.example.com/auth/callback',
				state: 'abc-state'
			})
		);

		expect(url.origin + url.pathname).toBe('https://accounts.google.com/o/oauth2/v2/auth');
		expect(url.searchParams.get('client_id')).toBe('client-123');
		expect(url.searchParams.get('redirect_uri')).toBe('https://pobot.example.com/auth/callback');
		expect(url.searchParams.get('state')).toBe('abc-state');
		expect(url.searchParams.get('scope')).toBe('openid email');
	});
});

describe('exchangeGoogleCode / fetchGoogleUserInfo', () => {
	afterEach(() => {
		vi.unstubAllGlobals();
	});

	it('exchanges a code for an access token', async () => {
		vi.stubGlobal(
			'fetch',
			vi.fn(async () => new Response(JSON.stringify({ access_token: 'fake-token' }), { status: 200 }))
		);

		const result = await exchangeGoogleCode({
			code: 'auth-code',
			clientId: 'client-123',
			clientSecret: 'secret-123',
			redirectUri: 'https://pobot.example.com/auth/callback'
		});

		expect(result.access_token).toBe('fake-token');
	});

	it('throws when the token exchange response is not OK', async () => {
		vi.stubGlobal('fetch', vi.fn(async () => new Response('nope', { status: 400 })));

		await expect(
			exchangeGoogleCode({
				code: 'bad-code',
				clientId: 'client-123',
				clientSecret: 'secret-123',
				redirectUri: 'https://pobot.example.com/auth/callback'
			})
		).rejects.toThrow('Google token exchange failed: 400');
	});

	it('fetches user info with the access token', async () => {
		vi.stubGlobal(
			'fetch',
			vi.fn(async () => new Response(JSON.stringify({ email: 'mentor@example.com', email_verified: true }), { status: 200 }))
		);

		const result = await fetchGoogleUserInfo('fake-token');

		expect(result).toEqual({ email: 'mentor@example.com', email_verified: true });
	});

	it('throws when the userinfo response is not OK', async () => {
		vi.stubGlobal('fetch', vi.fn(async () => new Response('nope', { status: 401 })));

		await expect(fetchGoogleUserInfo('bad-token')).rejects.toThrow('Google userinfo fetch failed: 401');
	});
});
```

- [ ] **Step 6: Run the tests and verify they fail**

```bash
npm test
```

Expected: FAIL — `./googleOAuth` module does not exist.

- [ ] **Step 7: Implement `src/lib/server/googleOAuth.ts`**

```ts
export interface GoogleUserInfo {
	email: string;
	email_verified: boolean;
}

export function buildGoogleAuthorizeUrl(params: {
	clientId: string;
	redirectUri: string;
	state: string;
}): string {
	const url = new URL('https://accounts.google.com/o/oauth2/v2/auth');
	url.searchParams.set('client_id', params.clientId);
	url.searchParams.set('redirect_uri', params.redirectUri);
	url.searchParams.set('response_type', 'code');
	url.searchParams.set('scope', 'openid email');
	url.searchParams.set('state', params.state);
	return url.toString();
}

export async function exchangeGoogleCode(params: {
	code: string;
	clientId: string;
	clientSecret: string;
	redirectUri: string;
}): Promise<{ access_token: string }> {
	const response = await fetch('https://oauth2.googleapis.com/token', {
		method: 'POST',
		headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
		body: new URLSearchParams({
			code: params.code,
			client_id: params.clientId,
			client_secret: params.clientSecret,
			redirect_uri: params.redirectUri,
			grant_type: 'authorization_code'
		})
	});
	if (!response.ok) {
		throw new Error(`Google token exchange failed: ${response.status}`);
	}
	return response.json();
}

export async function fetchGoogleUserInfo(accessToken: string): Promise<GoogleUserInfo> {
	const response = await fetch('https://www.googleapis.com/oauth2/v3/userinfo', {
		headers: { Authorization: `Bearer ${accessToken}` }
	});
	if (!response.ok) {
		throw new Error(`Google userinfo fetch failed: ${response.status}`);
	}
	return response.json();
}
```

- [ ] **Step 8: Run the tests and verify they pass**

```bash
npm test
```

Expected: PASS (all 5 cases)

- [ ] **Step 9: Create `src/routes/auth/login/+server.ts`**

```ts
import type { RequestHandler } from './$types';
import { buildGoogleAuthorizeUrl } from '$lib/server/googleOAuth';

export const GET: RequestHandler = async ({ platform, url, cookies }) => {
	const state = crypto.randomUUID();
	cookies.set('oauth_state', state, {
		path: '/',
		httpOnly: true,
		secure: true,
		sameSite: 'lax',
		maxAge: 600
	});

	const authorizeUrl = buildGoogleAuthorizeUrl({
		clientId: platform!.env.GOOGLE_CLIENT_ID,
		redirectUri: `${url.origin}/auth/callback`,
		state
	});

	return new Response(null, { status: 302, headers: { Location: authorizeUrl } });
};
```

- [ ] **Step 10: Create `src/routes/auth/callback/+server.ts`**

```ts
import type { RequestHandler } from './$types';
import { error, redirect } from '@sveltejs/kit';
import { exchangeGoogleCode, fetchGoogleUserInfo } from '$lib/server/googleOAuth';
import { signSession } from '$lib/server/session';
import { isAdmin } from '$lib/server/db';

export const GET: RequestHandler = async ({ platform, url, cookies }) => {
	const code = url.searchParams.get('code');
	const state = url.searchParams.get('state');
	const expectedState = cookies.get('oauth_state');
	cookies.delete('oauth_state', { path: '/' });

	if (!code || !state || !expectedState || state !== expectedState) {
		error(400, 'Invalid OAuth state');
	}

	const env = platform!.env;
	const { access_token } = await exchangeGoogleCode({
		code,
		clientId: env.GOOGLE_CLIENT_ID,
		clientSecret: env.GOOGLE_CLIENT_SECRET,
		redirectUri: `${url.origin}/auth/callback`
	});
	const userInfo = await fetchGoogleUserInfo(access_token);

	if (!userInfo.email_verified || !(await isAdmin(env.DB, userInfo.email))) {
		error(403, 'This Google account is not on the admin allowlist');
	}

	const token = await signSession(
		{ email: userInfo.email, exp: Math.floor(Date.now() / 1000) + 60 * 60 * 12 },
		env.SESSION_SECRET
	);
	cookies.set('session', token, {
		path: '/',
		httpOnly: true,
		secure: true,
		sameSite: 'lax',
		maxAge: 60 * 60 * 12
	});

	redirect(303, '/admin');
};
```

- [ ] **Step 11: Document required local secrets**

Create `.dev.vars.example`:

```
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
SESSION_SECRET=
```

Then, locally, copy it and fill in real values (this file is gitignored per Task 1):

```bash
cp .dev.vars.example .dev.vars
```

Fill in `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` from the prerequisite step, and set `SESSION_SECRET` to a random 32+ character string, e.g.:

```bash
openssl rand -base64 32
```

- [ ] **Step 12: Seed yourself as the first admin (local DB)**

```bash
npx wrangler d1 execute po-bot-db --local --command "INSERT INTO admins (google_email) VALUES ('YOUR_EMAIL@gmail.com')"
```

- [ ] **Step 13: Commit**

```bash
git add src/lib/server/googleOAuth.ts src/lib/server/googleOAuth.test.ts src/lib/server/db.ts src/lib/server/db.test.ts src/routes/auth/login src/routes/auth/callback .dev.vars.example
git commit -m "Add Google OAuth login flow with admin allowlist check"
```

---

### Task 6: Admin Route Guard & Logout

**Files:**
- Create: `src/routes/admin/+layout.server.ts`
- Create: `src/routes/admin/+layout.svelte`
- Create: `src/routes/admin/+page.svelte`
- Create: `src/routes/auth/logout/+server.ts`

**Interfaces:**
- Consumes: `verifySession` from Task 4.
- Produces: every route under `/admin/*` requires a valid session; `data.email` is available to all admin pages via the layout's returned data.

- [ ] **Step 1: Create `src/routes/admin/+layout.server.ts`**

```ts
import type { LayoutServerLoad } from './$types';
import { redirect } from '@sveltejs/kit';
import { verifySession } from '$lib/server/session';

export const load: LayoutServerLoad = async ({ platform, cookies }) => {
	const token = cookies.get('session');
	const payload = token ? await verifySession(token, platform!.env.SESSION_SECRET) : null;

	if (!payload) {
		redirect(303, '/auth/login');
	}

	return { email: payload.email };
};
```

- [ ] **Step 2: Create `src/routes/admin/+layout.svelte`**

```svelte
<script lang="ts">
	import type { LayoutData } from './$types';

	let { data, children }: { data: LayoutData; children: () => unknown } = $props();
</script>

<header>
	<span>Signed in as {data.email}</span>
	<a href="/auth/logout">Log out</a>
</header>

{@render children()}
```

- [ ] **Step 3: Create `src/routes/admin/+page.svelte`**

```svelte
<h1>PO Bot Admin</h1>
<p>Foundation phase — Needs Review, Settings, Webhook Logs, Stats, and Inventory pages arrive in later plans.</p>
```

- [ ] **Step 4: Create `src/routes/auth/logout/+server.ts`**

```ts
import type { RequestHandler } from './$types';
import { redirect } from '@sveltejs/kit';

export const GET: RequestHandler = async ({ cookies }) => {
	cookies.delete('session', { path: '/' });
	redirect(303, '/');
};
```

- [ ] **Step 5: Verify the guard manually with `wrangler dev`**

```bash
npm run build
npx wrangler dev &
sleep 3
curl -s -o /dev/null -w "%{http_code}" http://localhost:8787/admin
```

Expected: `303` (redirect to `/auth/login`, since there's no session cookie in this request)

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8787/auth/login
kill %1
```

Expected: `302` (redirect to Google's authorization URL)

- [ ] **Step 6: Commit**

```bash
git add src/routes/admin src/routes/auth/logout
git commit -m "Add admin route guard, logout, and admin landing page"
```

---

### Task 7: Deploy & Smoke Test

**Files:**
- None created — this task deploys and verifies the work from Tasks 1–6.

**Interfaces:**
- Consumes: everything from Tasks 1–6.
- Produces: a live URL at `https://po-bot.<your-subdomain>.workers.dev`.

- [ ] **Step 1: Apply the migration to the remote (production) D1 database**

```bash
npx wrangler d1 migrations apply po-bot-db --remote
```

Expected: output confirms `0001_initial_schema.sql` applied to the remote database.

- [ ] **Step 2: Set production secrets**

```bash
npx wrangler secret put GOOGLE_CLIENT_ID
npx wrangler secret put GOOGLE_CLIENT_SECRET
npx wrangler secret put SESSION_SECRET
```

Use the same values as your local `.dev.vars` (generate a fresh `SESSION_SECRET` if you prefer a separate one for production).

- [ ] **Step 3: Seed yourself as the first admin (remote DB)**

```bash
npx wrangler d1 execute po-bot-db --remote --command "INSERT INTO admins (google_email) VALUES ('YOUR_EMAIL@gmail.com')"
```

- [ ] **Step 4: Deploy**

```bash
npm run deploy
```

Expected: output includes the deployed Worker URL (`https://po-bot.<your-subdomain>.workers.dev`).

- [ ] **Step 5: Add the production redirect URI to the Google OAuth client**

Back in Google Cloud Console, add `https://po-bot.<your-subdomain>.workers.dev/auth/callback` to the OAuth client's authorized redirect URIs (per the Task 5 prerequisite note).

- [ ] **Step 6: Smoke test the public board**

```bash
curl -s -o /dev/null -w "%{http_code}" https://po-bot.<your-subdomain>.workers.dev/
```

Expected: `200`

- [ ] **Step 7: Smoke test the admin login redirect**

```bash
curl -s -o /dev/null -w "%{http_code}" https://po-bot.<your-subdomain>.workers.dev/admin
```

Expected: `303`

- [ ] **Step 8: Manually verify the full login flow**

In a browser, visit `https://po-bot.<your-subdomain>.workers.dev/admin`, sign in with the Google account seeded as an admin in Step 3, and confirm you land on the admin page showing "Signed in as \<your email\>".
