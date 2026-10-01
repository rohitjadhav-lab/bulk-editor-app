# Bulk Editor — Learning Plan

A Shopify Admin app for bulk-editing products, built one milestone at a time.
Stack: React Router app template (React + Node.js) · Polaris web components · Prisma + SQLite.

**How to use this file**
- Do one milestone per sitting. Don't start the next one until every "Done when" box is ticked.
- Commit after each milestone: `git commit -m "M3: selection"`.
- Stuck for more than 20 minutes? Paste the error to Claude along with the milestone number.

---

## Everything here is free

| Need | Free tool | Notes |
|---|---|---|
| Shopify account | Shopify Partner / Dev Dashboard account | Free |
| Test store | Development store | Free and unlimited. No real payments |
| Dev server + public URL | Shopify CLI (`shopify app dev`) | Includes a free tunnel. No ngrok account |
| Runtime | Node.js 20 LTS | Free |
| Frontend | React + React Router + Polaris web components | Open source |
| Database | SQLite via Prisma | One local file, no DB server |
| Background jobs | Plain async functions in Node | No Redis, no paid queue |
| API explorer | GraphiQL (press `g` while `shopify app dev` runs) | Free. Use it to try every query before writing code |
| Hosting | None. Runs on your laptop | Decide on hosting after you finish learning |

---

## Progress

| # | Milestone | Hours | Done |
|---|---|---|---|
| 0 | Setup | 1 | [ ] |
| 1 | Understand the template (backend basics) | 2–3 | [ ] |
| 2 | Product table: search, filters, pagination | 3–4 | [ ] |
| 3 | Selection | 2 | [ ] |
| 4 | Action list + dynamic inputs | 3 | [ ] |
| 5 | Preview + validation | 3–4 | [ ] |
| 6 | Database + background executor | 4–5 | [ ] |
| 7 | Progress screen | 2–3 | [ ] |
| 8 | Results, retry, CSV export, history | 3 | [ ] |
| 9 | Rate-limit handling | 2–3 | [ ] |
| 10 | Bulk Operations API + webhook | 5–6 | [ ] |

MVP = milestones 0–8, about 23–28 hours.

---

## Final folder layout (what you're building towards)

```
bulk-editor/
  app/
    shopify.server.js          ← from template: auth, admin client
    db.server.js               ← from template: Prisma client
    bulk/
      actions.js               ← M4: one object per bulk action (the "modules")
      products.server.js       ← M2/M5: product GraphQL queries
      executor.server.js       ← M6: background job runner
      graphql.server.js        ← M9: rate-limit-aware GraphQL helper
    routes/
      app.jsx                  ← from template: layout + nav
      app.products.jsx         ← M2–M3: list, search, select
      app.bulk.jsx             ← M4–M5: choose action, edit, preview, confirm
      app.jobs._index.jsx      ← M8: job history
      app.jobs.$id.jsx         ← M7–M8: progress + results
      webhooks.bulk_operations.finish.jsx   ← M10
  prisma/schema.prisma         ← M6: BulkJob, BulkJobItem
  shopify.app.toml             ← scopes, webhooks
```

**Key idea: `.server.js` files.** Any file ending in `.server.js` runs only on the Node server and never reaches the browser. Anything that touches the database or access tokens goes in one of these files.

---

## M0 — Setup (1 h)

1. Switch to Node 20. Your current Node 16 is too old:
   ```bash
   nvm install 20
   nvm alias default 20
   node -v
   ```
2. Reinstall the Shopify CLI. Global packages are installed per Node version, so Node 20 needs its own copy:
   ```bash
   npm install -g @shopify/cli@latest
   ```
3. Create the app:
   ```bash
   cd "/Users/rohitjadhav/Documents/Shopify Apps"
   shopify app init
   ```
   - Name: `bulk-editor`
   - Template: **React Router**
   - Language, if asked: **JavaScript**
4. Run it:
   ```bash
   cd bulk-editor
   shopify app dev
   ```
   Press `p` to open the app, then install it on your dev store.
5. Add test data: you need **at least 60 products** to test pagination. Either create the dev store with generated test data, or import a products CSV (Admin → Products → Import).
6. Move this plan file into the repo and make your first commit:
   ```bash
   mv ../BULK_EDITOR_PLAN.md .
   git init && git add . && git commit -m "M0: template"
   ```

**Done when**
- [ ] The app opens inside Shopify Admin
- [ ] Pressing `g` in the terminal opens GraphiQL
- [ ] The store has 60+ products

---

## M1 — Understand the template (2–3 h)

**You learn:** how a request reaches the backend, how auth works, and how the backend talks to Shopify.

### 1. Read these files, in this order
1. `shopify.app.toml`: app config and scopes
2. `app/shopify.server.js`: creates the `shopify` object. Exports `authenticate` (used inside requests) and `unauthenticated` (used in background jobs, M6)
3. `app/routes/app.jsx`: layout for every `/app/*` page, including the nav menu
4. `app/routes/app._index.jsx`: example page with a `loader` and an `action`
5. `prisma/schema.prisma`: currently holds only the `Session` table, where access tokens live

### 2. The one diagram to remember

```
Browser                          Node server (your code)                Shopify
───────                          ───────────────────────                ───────
GET /app/products  ─────────▶    loader({ request })
                                   authenticate.admin(request)  ──▶ checks session token
                                   admin.graphql(query)         ──▶ Admin GraphQL API
                                   return { products }          ◀──
useLoaderData()    ◀─────────
POST (form/fetcher) ────────▶    action({ request })   → writes / mutations
```

- **loader** = runs on the server on every GET. It reads data.
- **action** = runs on the server on POST. It changes data.
- **component** = runs in the browser. It receives the loader's return value via `useLoaderData()`.
- `authenticate.admin(request)` does all the OAuth/session-token work. **Call it first in every loader and action.**

### 3. Scopes
In `shopify.app.toml` set:
```toml
[access_scopes]
scopes = "write_products"
```
`write_products` includes read access. Restart `shopify app dev` and accept the updated permissions.

### 4. Exercise: your first backend route
Create `app/routes/app.products.jsx`:

```jsx
import { useLoaderData } from "react-router";
import { authenticate } from "../shopify.server";

// Runs on the SERVER
export const loader = async ({ request }) => {
  const { admin } = await authenticate.admin(request);

  const response = await admin.graphql(`#graphql
    query {
      products(first: 10) {
        nodes { id title }
      }
    }`);
  const { data } = await response.json();

  return { products: data.products.nodes };
};

// Runs in the BROWSER
export default function ProductsPage() {
  const { products } = useLoaderData();
  return (
    <s-page heading="Products">
      <s-section>
        {products.map((p) => (
          <s-paragraph key={p.id}>{p.title}</s-paragraph>
        ))}
      </s-section>
    </s-page>
  );
}
```

Add a nav link in `app/routes/app.jsx`, inside `<s-app-nav>`:
```jsx
<s-link href="/app/products">Products</s-link>
```

### 5. Two quick experiments (10 min)
1. Add `console.log("loader ran")` inside the loader. Reload the page. **The log shows in your terminal, not in the browser console.** That proves the loader runs on the server.
2. Add a text field to the page and check that its events fire:
   ```jsx
   <s-text-field label="Test" onInput={(e) => console.log(e.currentTarget.value)} />
   <s-checkbox label="Test" onChange={(e) => console.log(e.currentTarget.checked)} />
   ```
   Both should log in the **browser** console. If either one doesn't, tell Claude your React version (in `package.json`). The event wiring differs between React 18 and 19.

**Done when**
- [ ] The Products page lists 10 titles
- [ ] You can explain in one sentence each: loader, action, `authenticate.admin`
- [ ] Both experiments behave as described

---

## M2 — Product table: search, filters, pagination (3–4 h)

**You learn:** GraphQL variables, Shopify search syntax, cursor pagination, and URL search params as state.

**Key idea:** store search, filters and the page cursor **in the URL** (`?q=shirt&status=ACTIVE&after=xyz`). The loader reads the URL, the back button works for free, and a refresh keeps your place.

### 1. `app/bulk/products.server.js`

```js
const PAGE_SIZE = 25;

// Turns form values into Shopify search syntax, e.g.  title:*shirt* AND status:ACTIVE
export function buildSearchQuery({ q, status, vendor, productType }) {
  const quote = (s) => `"${s.replace(/"/g, '\\"')}"`;
  const parts = [];
  if (q) parts.push(`title:*${q}*`);
  if (status) parts.push(`status:${status}`);
  if (vendor) parts.push(`vendor:${quote(vendor)}`);
  if (productType) parts.push(`product_type:${quote(productType)}`);
  return parts.join(" AND ");
}

export async function listProducts(admin, { query, after, before }) {
  // Going back a page uses last/before; forward uses first/after
  const paging = before
    ? { last: PAGE_SIZE, before }
    : { first: PAGE_SIZE, after: after || null };

  const response = await admin.graphql(
    `#graphql
    query ListProducts($first: Int, $last: Int, $after: String, $before: String, $query: String) {
      products(first: $first, last: $last, after: $after, before: $before, query: $query, sortKey: TITLE) {
        nodes { id title status vendor productType tags }
        pageInfo { hasNextPage hasPreviousPage startCursor endCursor }
      }
    }`,
    { variables: { ...paging, query: query || null } },
  );
  const { data } = await response.json();
  return data.products;
}
```

### 2. The loader in `app.products.jsx`

```jsx
import { authenticate } from "../shopify.server";
import { buildSearchQuery, listProducts } from "../bulk/products.server";

export const loader = async ({ request }) => {
  const { admin } = await authenticate.admin(request);
  const params = new URL(request.url).searchParams;

  const filters = {
    q: params.get("q") || "",
    status: params.get("status") || "",
    vendor: params.get("vendor") || "",
    productType: params.get("productType") || "",
  };

  const products = await listProducts(admin, {
    query: buildSearchQuery(filters),
    after: params.get("after"),
    before: params.get("before"),
  });

  return { products: products.nodes, pageInfo: products.pageInfo, filters };
};
```

### 3. The component (you write most of this)
- Keep form values in local state, initialised from `filters`
- "Search" button:
  ```js
  const [, setSearchParams] = useSearchParams();
  setSearchParams({ q, status, vendor, productType });
  ```
  Leaving out `after`/`before` resets to page 1.
- Status filter: `<s-select>` with `<s-option>` children: All, ACTIVE, DRAFT, ARCHIVED
- Table columns: checkbox (M3), Title, Status, Vendor, Type, Tags
- Pagination buttons:
  - Next: `setSearchParams({ ...currentFilters, after: pageInfo.endCursor })`, disabled when `!pageInfo.hasNextPage`
  - Previous: `before: pageInfo.startCursor`, disabled when `!pageInfo.hasPreviousPage`
- Loading state: `const navigation = useNavigation();` → `navigation.state === "loading"` → dim the table or show `<s-spinner>`

**Try first in GraphiQL:** `products(first: 5, query: "status:DRAFT")`, then `"vendor:\"Nike\""`.

**Done when**
- [ ] Searching "shirt" narrows the list
- [ ] Status + vendor filters work together
- [ ] Next/Previous pages work, and the browser back button returns to the previous page
- [ ] Refreshing the page keeps filters and page

---

## M3 — Selection (2 h)

**You learn:** selection state, and why state survives page changes.

```jsx
const [selected, setSelected] = useState(() => new Set()); // Set of product GIDs

const toggle = (id) =>
  setSelected((prev) => {
    const next = new Set(prev);          // never mutate state directly: copy, then change
    next.has(id) ? next.delete(id) : next.add(id);
    return next;
  });

const selectAllVisible = () =>
  setSelected((prev) => new Set([...prev, ...products.map((p) => p.id)]));

const clear = () => setSelected(new Set());
```

- Row checkbox: `<s-checkbox checked={selected.has(p.id)} onChange={() => toggle(p.id)} />`
- Toolbar: "Select all on this page", "Clear", "**{selected.size} selected**"
- **Why selection survives pagination:** changing search params reruns the loader, but it's the *same route*, so React keeps the component and its state.
- "Continue" button (disabled at 0) passes the IDs to the next page through **router state**. Router state is not the URL, so hundreds of IDs never hit URL length limits:
  ```js
  const navigate = useNavigate();
  navigate("/app/bulk", { state: { ids: [...selected] } });
  ```

**Done when**
- [ ] You can select 3 products on page 1 and 2 on page 2, and the count shows 5
- [ ] Clear resets to 0
- [ ] Continue opens `/app/bulk` (an empty page is fine for now)

---

## M4 — Action list + dynamic inputs (3 h)

**You learn:** config-driven UI. This is the "module" system that makes future bulk operations easy to add.

**Key idea:** each bulk action is **one object** in `app/bulk/actions.js`. The UI, preview, validation and executor all read from it. **Adding a new operation later = adding one object.**

### `app/bulk/actions.js`

```js
// {{title}} {{vendor}} {{type}} → replaced with each product's values
export function applyTemplate(template, p) {
  return template
    .replaceAll("{{title}}", p.title ?? "")
    .replaceAll("{{vendor}}", p.vendor ?? "")
    .replaceAll("{{type}}", p.productType ?? "");
}

const productUpdate = (product) => ({
  resultKey: "productUpdate",
  query: `#graphql
    mutation BulkProductUpdate($product: ProductUpdateInput!) {
      productUpdate(product: $product) {
        product { id }
        userErrors { field message }
      }
    }`,
  variables: { product },
});

export const ACTIONS = {
  title: {
    label: "Update title",
    input: "text",
    help: "Use {{title}}, {{vendor}}, {{type}}. Example: {{title}} – Sale",
    getCurrent: (p) => p.title,
    getNew: (p, value) => applyTemplate(value, p).trim(),
    validate: (next) =>
      !next ? "Title can't be empty" : next.length > 255 ? "Title is over 255 characters" : null,
    display: (v) => v,
    mutation: (id, next) => productUpdate({ id, title: next }),
  },

  vendor: {
    label: "Update vendor",
    input: "text",
    getCurrent: (p) => p.vendor ?? "",
    getNew: (p, value) => value.trim(),
    validate: (next) => (!next ? "Vendor can't be empty" : null),
    display: (v) => v || "—",
    mutation: (id, next) => productUpdate({ id, vendor: next }),
  },

  productType: {
    label: "Update product type",
    input: "text",
    getCurrent: (p) => p.productType ?? "",
    getNew: (p, value) => value.trim(),
    validate: () => null, // empty is allowed: clears the type
    display: (v) => v || "—",
    mutation: (id, next) => productUpdate({ id, productType: next }),
  },

  status: {
    label: "Update status",
    input: "select",
    options: ["ACTIVE", "DRAFT", "ARCHIVED"],
    getCurrent: (p) => p.status,
    getNew: (p, value) => value,
    validate: (next) => (["ACTIVE", "DRAFT", "ARCHIVED"].includes(next) ? null : "Invalid status"),
    display: (v) => v,
    mutation: (id, next) => productUpdate({ id, status: next }),
  },

  tags: {
    label: "Update tags",
    input: "tags", // UI: mode select (add / remove / replace) + comma-separated text
    getCurrent: (p) => p.tags,
    getNew: (p, { mode, tags }) => {
      const list = tags.split(",").map((t) => t.trim()).filter(Boolean);
      if (mode === "add") return [...new Set([...p.tags, ...list])];
      if (mode === "remove") return p.tags.filter((t) => !list.includes(t));
      return list; // replace
    },
    validate: (next) => (next.some((t) => t.length > 255) ? "A tag is over 255 characters" : null),
    display: (v) => v.join(", ") || "—",
    // Replaces the full tag list with the previewed result.
    // Later exercise: switch "add"/"remove" to the tagsAdd / tagsRemove mutations.
    mutation: (id, next) => productUpdate({ id, tags: next }),
  },

  seo: {
    label: "Update SEO",
    input: "seo", // UI: two text fields. Leave a field blank to keep its current value
    getCurrent: (p) => ({ title: p.seo?.title ?? "", description: p.seo?.description ?? "" }),
    getNew: (p, value) => ({
      title: value.title ? applyTemplate(value.title, p) : p.seo?.title ?? "",
      description: value.description ? applyTemplate(value.description, p) : p.seo?.description ?? "",
    }),
    validate: (next) =>
      next.title.length > 70 ? "SEO title is over 70 characters"
      : next.description.length > 320 ? "SEO description is over 320 characters"
      : null,
    display: (v) => `${v.title || "—"} | ${v.description || "—"}`,
    // Sends both fields so the one you didn't edit is written back unchanged
    mutation: (id, next) => productUpdate({ id, seo: next }),
  },
};
```

Variant price comes in M5b. It needs variant data, so it's handled separately.

### The UI in `app/routes/app.bulk.jsx`
1. Read the IDs: `const ids = useLocation().state?.ids ?? [];`. If the list is empty, show a banner with a link back to Products.
2. `<s-select>` built from `Object.entries(ACTIONS)` → sets `actionKey`
3. Render the input based on `ACTIONS[actionKey].input`:
   - `"text"` → one `<s-text-field>`, plus the `help` text
   - `"select"` → `<s-select>` with `options`
   - `"tags"` → mode select + text field. Value is `{ mode, tags }`
   - `"seo"` → two text fields. Value is `{ title, description }`
4. Reset the value whenever `actionKey` changes

**Done when**
- [ ] Picking each action shows the right input
- [ ] The heading shows "Editing N products"

---

## M5 — Preview + validation (3–4 h)

**You learn:** server-side validation, calling actions with `useFetcher`, and "never trust the client".

### 1. Fetch the selected products by ID (add to `products.server.js`)

```js
// nodes(ids:) accepts at most 250 IDs, so split into chunks
export async function getProductsByIds(admin, ids) {
  const chunks = [];
  for (let i = 0; i < ids.length; i += 100) chunks.push(ids.slice(i, i + 100));

  const results = [];
  for (const chunk of chunks) {
    const response = await admin.graphql(
      `#graphql
      query ProductsByIds($ids: [ID!]!) {
        nodes(ids: $ids) {
          ... on Product {
            id title status vendor productType tags
            seo { title description }
          }
        }
      }`,
      { variables: { ids: chunk } },
    );
    const { data } = await response.json();
    results.push(...data.nodes.filter(Boolean)); // deleted products come back as null
  }
  return results;
}
```

### 2. Build preview rows. Shared by preview (M5) and run (M6)

Put this in `app/bulk/actions.js` so both steps use the exact same logic:

```js
export function buildRows(action, products, value) {
  return products.map((p) => {
    const current = action.getCurrent(p);
    const next = action.getNew(p, value);
    const unchanged = JSON.stringify(current) === JSON.stringify(next);
    return {
      id: p.id,
      title: p.title,
      current,
      next,
      unchanged,
      error: unchanged ? null : action.validate(next, p),
    };
  });
}
```

### 3. The action in `app.bulk.jsx` (server)

```jsx
import { authenticate } from "../shopify.server";
import { ACTIONS, buildRows } from "../bulk/actions";
import { getProductsByIds } from "../bulk/products.server";

export const action = async ({ request }) => {
  const { admin } = await authenticate.admin(request);
  const form = await request.formData();

  const intent = form.get("intent");                 // "preview" | "run"
  const action = ACTIONS[form.get("actionKey")];
  const ids = JSON.parse(form.get("ids"));
  const value = JSON.parse(form.get("value"));

  if (!action) return { error: "Unknown action" };   // validate input: it came from the browser
  if (!ids.length) return { error: "No products selected" };

  const products = await getProductsByIds(admin, ids);
  const rows = buildRows(action, products, value);

  if (intent === "preview") return { rows };
  // intent === "run" is added in M6
};
```

### 4. The component (browser)

```jsx
const fetcher = useFetcher();

const preview = () =>
  fetcher.submit(
    { intent: "preview", actionKey, ids: JSON.stringify(ids), value: JSON.stringify(value) },
    { method: "post" },
  );

const rows = fetcher.data?.rows ?? [];
const willChange = rows.filter((r) => !r.unchanged && !r.error);
const willFail = rows.filter((r) => r.error);
const unchanged = rows.filter((r) => r.unchanged);
```

Show:
- Summary line: **"38 will change · 4 unchanged (skipped) · 2 will fail"**
- A table with Product | Current | New | Note. Failing rows show the error in a critical `<s-badge>`.
- Confirmation: a checkbox "I understand this will update **38** products", then a primary button **"Update 38 products"**. Disable the button until the box is ticked and while `willChange.length === 0`.

**Why validation runs on the server:** anyone can edit what the browser sends. The server recomputes everything from fresh Shopify data, and M6 does it again at run time.

### M5b — Variant price (optional here, or after M8)
- Add `variants(first: 50) { nodes { id price compareAtPrice } }` to the query **only when the price action is selected**. Variants make the query expensive, so also lower the chunk size to 10.
- Value: `{ field: "price" | "compareAtPrice", mode: "set" | "percent" | "amount", amount }`
- `getNew` returns `[{ id, price }]` for each variant. Round to 2 decimals: `Math.round(x * 100) / 100`, then `.toFixed(2)`.
- `validate`: flag anything that isn't a number or is below 0. For compare-at, also flag values lower than the price.
- Mutation:
  ```graphql
  mutation ($productId: ID!, $variants: [ProductVariantsBulkInput!]!) {
    productVariantsBulkUpdate(productId: $productId, variants: $variants) {
      productVariants { id }
      userErrors { field message }
    }
  }
  ```

**Done when**
- [ ] The preview shows correct current → new values for every action
- [ ] An empty title shows as "will fail" with a reason
- [ ] Rows that wouldn't change are marked "unchanged"
- [ ] You can't click the update button until the confirmation box is ticked

---

## M6 — Database + background executor (4–5 h) ⭐ biggest backend milestone

**You learn:** database modeling, migrations, background jobs, and audit trails.

### 1. Add two tables to `prisma/schema.prisma`

```prisma
model BulkJob {
  id         Int           @id @default(autoincrement())
  shop       String
  actionKey  String
  value      String        // JSON of what the user entered
  status     String        @default("pending") // pending | running | completed | interrupted
  total      Int
  processed  Int           @default(0)
  succeeded  Int           @default(0)
  failed     Int           @default(0)
  createdAt  DateTime      @default(now())
  finishedAt DateTime?
  items      BulkJobItem[]
}

model BulkJobItem {
  id        Int     @id @default(autoincrement())
  jobId     Int
  job       BulkJob @relation(fields: [jobId], references: [id], onDelete: Cascade)
  productId String
  title     String
  before    String  // JSON: value before the change (audit trail)
  after     String  // JSON: value we're writing
  status    String  @default("pending") // pending | success | failed
  error     String?

  @@index([jobId, status])
}
```

Then run:
```bash
npx prisma migrate dev --name bulk_jobs
```

**What happens:** Prisma writes a SQL migration file in `prisma/migrations/`, applies it to the SQLite file, and regenerates the client so `db.bulkJob` exists. Commit the migration file.

`before`/`after` are JSON strings because that's the simplest option that works everywhere: `JSON.stringify` on save, `JSON.parse` on read.

### 2. `app/bulk/executor.server.js`

```js
import db from "../db.server";
import { unauthenticated } from "../shopify.server";
import { ACTIONS } from "./actions";

// On server start, any job still marked "running" was killed by a restart.
// The globalThis flag stops dev hot-reload from running this twice.
if (!globalThis.__bulkRecovered) {
  globalThis.__bulkRecovered = true;
  db.bulkJob
    .updateMany({ where: { status: "running" }, data: { status: "interrupted" } })
    .catch(console.error);
}

export async function runJob(jobId) {
  const job = await db.bulkJob.update({ where: { id: jobId }, data: { status: "running" } });

  // No browser request exists in the background, so use the shop's stored offline token
  const { admin } = await unauthenticated.admin(job.shop);
  const action = ACTIONS[job.actionKey];

  const items = await db.bulkJobItem.findMany({ where: { jobId, status: "pending" } });

  for (const item of items) {
    try {
      const { query, variables, resultKey } = action.mutation(item.productId, JSON.parse(item.after));
      const response = await admin.graphql(query, { variables });
      const { data } = await response.json();

      // Shopify returns HTTP 200 even when a mutation fails. Errors live in userErrors.
      const userErrors = data[resultKey].userErrors;
      if (userErrors.length) throw new Error(userErrors.map((e) => e.message).join("; "));

      await finishItem(jobId, item.id, "success");
    } catch (err) {
      await finishItem(jobId, item.id, "failed", err.message ?? "Unknown error");
    }
  }

  await db.bulkJob.update({
    where: { id: jobId },
    data: { status: "completed", finishedAt: new Date() },
  });
}

async function finishItem(jobId, itemId, status, error = null) {
  // Transaction: the item and the job counters update together, or not at all
  await db.$transaction([
    db.bulkJobItem.update({ where: { id: itemId }, data: { status, error } }),
    db.bulkJob.update({
      where: { id: jobId },
      data: {
        processed: { increment: 1 },
        [status === "success" ? "succeeded" : "failed"]: { increment: 1 },
      },
    }),
  ]);
}
```

### 3. Add the `run` intent to the action in `app.bulk.jsx`

```jsx
import db from "../db.server";
import { runJob } from "../bulk/executor.server";

// inside action(), replace the end:
const { admin, session, redirect } = await authenticate.admin(request);
// ... same parsing + getProductsByIds + buildRows as before ...

if (intent === "preview") return { rows };

// "run": recompute from fresh data (never trust the old preview), keep only valid changes
const toApply = rows.filter((r) => !r.unchanged && !r.error);
if (!toApply.length) return { error: "Nothing to update" };

const job = await db.bulkJob.create({
  data: {
    shop: session.shop,
    actionKey: form.get("actionKey"),
    value: form.get("value"),
    total: toApply.length,
    items: {
      create: toApply.map((r) => ({
        productId: r.id,
        title: r.title,
        before: JSON.stringify(r.current),
        after: JSON.stringify(r.next),
      })),
    },
  },
});

runJob(job.id).catch(console.error); // no await: runs in the background
return redirect(`/app/jobs/${job.id}`);
```

**The core background-job idea:** the action saves the job and **returns immediately**. `runJob` keeps running in the same Node process after the response is sent. That's why the UI never freezes.

**Known limitation (accepted for the MVP):** if the server restarts mid-job, the job becomes `interrupted`. M8's retry can finish it.

**Tool:** run `npx prisma studio` to see your tables in a browser UI. It's free and very useful.

**Done when**
- [ ] Updating 5 product titles creates 1 `BulkJob` + 5 `BulkJobItem` rows (check in Prisma Studio)
- [ ] The titles change in Shopify Admin
- [ ] A product that fails (e.g. a title over 255 characters, if you bypass the preview) is stored with its error

---

## M7 — Progress screen (2–3 h)

**You learn:** polling, loading/error states, and confirmed vs optimistic updates.

`app/routes/app.jobs.$id.jsx`:

```jsx
import { useEffect } from "react";
import { useLoaderData, useRevalidator } from "react-router";
import { authenticate } from "../shopify.server";
import db from "../db.server";

export const loader = async ({ request, params }) => {
  const { session } = await authenticate.admin(request);
  const job = await db.bulkJob.findFirst({
    where: { id: Number(params.id), shop: session.shop }, // shop filter: never show another store's jobs
    include: { items: { where: { status: "failed" } } },
  });
  if (!job) throw new Response("Not found", { status: 404 });
  return { job };
};

export default function JobPage() {
  const { job } = useLoaderData();
  const revalidator = useRevalidator();
  const running = job.status === "pending" || job.status === "running";

  // Re-run the loader every 2s while the job runs
  useEffect(() => {
    if (!running) return;
    const timer = setInterval(() => revalidator.revalidate(), 2000);
    return () => clearInterval(timer);
  }, [running]);

  const percent = job.total ? Math.round((job.processed / job.total) * 100) : 0;

  return (
    <s-page heading={`Bulk job #${job.id}`}>
      <s-section>
        <s-paragraph>{running ? "Updating products…" : `Status: ${job.status}`}</s-paragraph>
        <progress value={job.processed} max={job.total} style={{ width: "100%" }} />
        <s-paragraph>
          {percent}% · Processed {job.processed} / {job.total} · Successful {job.succeeded} · Failed {job.failed}
        </s-paragraph>
      </s-section>
    </s-page>
  );
}
```

**Optimistic vs confirmed:** in `product-info-app` you flipped the status in the UI *before* Shopify answered (optimistic). For bulk jobs you show only what the database confirms. The numbers come from real results, so you never show a success that didn't happen.

**Done when**
- [ ] Running a 50-product job shows the bar moving
- [ ] Polling stops when the job completes (check the Network tab)
- [ ] Refreshing mid-job continues showing progress

---

## M8 — Results, retry, CSV export, history (3 h)

**You learn:** retry logic, exporting data, and a job history (audit log).

### 1. Failed list (on the job page, once the job is finished)
A table with Product | Error, built from `job.items`. Add a "Completed: X · Failed: Y" summary.

### 2. Retry failed: add an `action` to `app.jobs.$id.jsx`

```jsx
import { runJob } from "../bulk/executor.server";

export const action = async ({ request, params }) => {
  const { session, redirect } = await authenticate.admin(request);
  const old = await db.bulkJob.findFirst({
    where: { id: Number(params.id), shop: session.shop },
    include: { items: { where: { status: { in: ["failed", "pending"] } } } }, // pending = never ran (interrupted)
  });
  if (!old?.items.length) return { error: "Nothing to retry" };

  const job = await db.bulkJob.create({
    data: {
      shop: old.shop,
      actionKey: old.actionKey,
      value: old.value,
      total: old.items.length,
      items: {
        create: old.items.map(({ productId, title, before, after }) => ({ productId, title, before, after })),
      },
    },
  });
  runJob(job.id).catch(console.error);
  return redirect(`/app/jobs/${job.id}`);
};
```

Button: a `<fetcher.Form method="post">` containing `<s-button type="submit">Retry failed</s-button>`.

Retry creates a **new** job instead of editing the old one, so history stays honest.

### 3. Export failures as CSV (in the browser, no extra route)

```js
function downloadCsv(job) {
  const esc = (s) => `"${String(s ?? "").replace(/"/g, '""')}"`;
  const lines = [
    ["Product ID", "Title", "Error"].map(esc).join(","),
    ...job.items.map((i) => [i.productId, i.title, i.error].map(esc).join(",")),
  ];
  const url = URL.createObjectURL(new Blob([lines.join("\n")], { type: "text/csv" }));
  const a = document.createElement("a");
  a.href = url;
  a.download = `job-${job.id}-failures.csv`;
  a.click();
  URL.revokeObjectURL(url);
}
```

### 4. History page: `app/routes/app.jobs._index.jsx`
- Loader: `db.bulkJob.findMany({ where: { shop: session.shop }, orderBy: { createdAt: "desc" }, take: 50 })`
- Table: # | Action | Status | Succeeded / Total | Date. Each row links to `/app/jobs/{id}`.
- Add a "Bulk jobs" link to the nav in `app.jsx`

### 5. View details (optional)
Show all items on the job page with before → after values. That's your audit trail.

**Done when**
- [ ] Failures are listed with reasons
- [ ] "Retry failed" creates a new job and runs only those items
- [ ] The CSV downloads and opens in a spreadsheet
- [ ] The history page lists every job

🎉 **MVP complete.** The full flow works from selecting products to reviewing results.

---

## M9 — Rate-limit handling (2–3 h)

**You learn:** Shopify's cost-based leaky bucket, and backoff.

**How Shopify limits you:**
- Every query has a *cost* in points.
- Your app has a bucket of points per store that refills at a fixed rate per second.
- If you spend faster than it refills, you get a `THROTTLED` error.
- Every response tells you where you stand, in `extensions.cost.throttleStatus`: `maximumAvailable`, `currentlyAvailable` and `restoreRate`.

**Step 1: look first (10 min).** In `runJob`, log one response's `extensions`:
```js
const json = await response.json();
console.log(JSON.stringify(json.extensions, null, 2));
```
Note your store's `maximumAvailable` and `restoreRate`.

**Step 2: `app/bulk/graphql.server.js`**

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const MIN_POINTS = 100; // keep a safety margin in the bucket

function isThrottled(err) {
  // Log `err` the first time you hit a throttle, then adjust this check to match what you see
  return /throttled/i.test(String(err?.message)) || JSON.stringify(err?.body ?? "").includes("THROTTLED");
}

export async function gql(admin, query, variables) {
  for (let attempt = 0; ; attempt++) {
    try {
      const response = await admin.graphql(query, { variables });
      const json = await response.json();

      // Running low? Wait just long enough for the bucket to refill
      const t = json.extensions?.cost?.throttleStatus;
      if (t && t.currentlyAvailable < MIN_POINTS) {
        await sleep(((MIN_POINTS - t.currentlyAvailable) / t.restoreRate) * 1000);
      }
      return json;
    } catch (err) {
      if (!isThrottled(err) || attempt >= 4) throw err;
      await sleep(2 ** attempt * 1000); // exponential backoff: 1s, 2s, 4s, 8s
    }
  }
}
```

**Step 3:** replace `admin.graphql(...)` + `response.json()` in `runJob` with `await gql(admin, query, variables)`.

**Stretch goal:** process 3 items at a time (`Promise.all` on small batches) to finish jobs faster. That makes throttling actually happen, so you can watch your backoff work.

**Done when**
- [ ] A 200-product job finishes with zero `THROTTLED` failures
- [ ] You can explain the bucket in two sentences

---

## M10 — Bulk Operations API + webhook (5–6 h)

**You learn:** Shopify's async bulk mutations, staged uploads, JSONL and webhooks.

**When to use it:** jobs over about 250 products. Instead of sending one request per product, you upload **one file** and Shopify runs it on its side.

**The flow:**
```
1. Build a JSONL file: one line of variables per product
     {"product":{"id":"gid://shopify/Product/1","title":"New A"}}
     {"product":{"id":"gid://shopify/Product/2","title":"New B"}}
2. stagedUploadsCreate(resource: BULK_MUTATION_VARIABLES)  → upload URL + params
3. POST the file to that URL (Node 20 has fetch/FormData built in)
4. bulkOperationRunMutation(mutation, stagedUploadPath)    → Shopify starts the job
5. Wait: poll node(id) { ... on BulkOperation { status objectCount } }
       or receive the bulk_operations/finish webhook
6. Download the result JSONL from BulkOperation.url
     each line = { data: { productUpdate: { userErrors } }, __lineNumber: N }
7. Match __lineNumber → BulkJobItem (same order as step 1), save success/failed
```

**What you'll add:**
- `bulkOperationId String?` and `mode String @default("direct")` on `BulkJob`, then run `npx prisma migrate dev --name bulk_mode`
- A webhook subscription in `shopify.app.toml`:
  ```toml
  [[webhooks.subscriptions]]
  topics = ["bulk_operations/finish"]
  uri = "/webhooks/bulk_operations/finish"
  ```
- `app/routes/webhooks.bulk_operations.finish.jsx`:
  ```jsx
  export const action = async ({ request }) => {
    const { shop, payload } = await authenticate.webhook(request); // verifies Shopify's signature
    // payload.admin_graphql_api_id → find the BulkJob → fetch results → update items
    return new Response();
  };
  ```
- In `runJob`: `if (items.length > 250) → runBulkOperation(job)`, otherwise the existing loop

We'll write the full M10 code together when you get here. It builds on everything from M6–M9.

**Done when**
- [ ] A 300-product job runs as a bulk operation
- [ ] The webhook marks it complete
- [ ] Per-product failures from the result file show on the results page

---

## After the MVP: future modules (don't build yet)

Each of these is a new object in `ACTIONS`, sometimes with a new resource query:

| Module | New scope needed |
|---|---|
| Bulk metafields | none (`metafieldsSet`) |
| Bulk collection assignment | none (`collectionAddProducts`) |
| Bulk inventory | `read_inventory`, `write_inventory`, `read_locations` |
| Bulk publishing / channels | `read_publications`, `write_publications` |
| Bulk customer tags | `read_customers`, `write_customers` (+ a customers list page) |
| Bulk order tags | `read_orders`, `write_orders` |
| Bulk redirects | `read_online_store_navigation`, `write_online_store_navigation` |

Rule: add a scope only when you ship the feature that needs it.

---

## Safety checklist (applies to every milestone)

- [ ] Nothing changes without a preview
- [ ] The confirmation shows the exact number of affected records
- [ ] The server re-validates at run time
- [ ] Every change stores `before` + `after` (audit trail)
- [ ] Failures are stored, never dropped
- [ ] Every DB query filters by `shop`
- [ ] Test only on your dev store
