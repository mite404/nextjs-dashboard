# React Routing Framework Concepts — Cheatsheet

> Add lessons-learned below each section as you go.

---

## 1. Server vs Client Components

**Rule of thumb:** needs `useState` / `onClick` / browser APIs → client. Just fetching data and
rendering HTML → server.

| | Server Component | Client Component |
|---|---|---|
| Runs on | Server | Browser |
| Can `await` DB calls | ✓ | ✗ |
| Can use `useState` | ✗ | ✓ |
| JS sent to browser | None | Yes |
| Directive needed | None (default in Next.js) | `"use client"` at top |

```tsx
// Server component — no directive needed
async function BlogPost() {
  const post = await db.getPost(); // runs on server, never exposed to browser
  return <article>{post.content}</article>;
}

// Client component — must opt in
"use client";
function LikeButton() {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(true)}>Like</button>;
}
```

> [!TIP]
> **Analogy:** Server component = kitchen cooks and sends a finished plate. Client
> component = ingredients sent to the table, assembled there.

---

## 2. File-based vs Code-based Routing

### File-based (Next.js, Astro, SvelteKit)

The folder structure **is** the URL map. Move a file → change the
URL.

```text
app/
  page.tsx           →  /
  about/
    page.tsx         →  /about
  blog/
    [slug]/
      page.tsx       →  /blog/:slug
```

### Code-based (React Router library mode)

A `routes.ts` config file maps URLs to components. Move a file → update one line,
URL unchanged.

```ts
// routes.ts
export default [
  index("routes/home.tsx"),           // /
  route("chat", "routes/chat.tsx"),   // /chat
  route("api/chat", "routes/ai.ts"),  // /api/chat
  route("api/auth/*", "routes/api.auth.$.ts"), // /api/auth/* (wildcard)
  route("protected", "routes/protected.tsx")
] satisfies RouteConfig;
```

> [!TIP]
> **Analogy:** File-based = bookshelf (location IS the address). Code-based = library
> catalog (entry points to wherever the book lives).

**When code-based wins:** renaming routes without moving files, complex auth guards, feature-flag
routing.
**When file-based wins:** legibility for new devs, most apps where route = component anyway.

---

## 3. Pages Router vs App Router (Next.js)

| | Pages Router (old) | App Router (current ✦) |
|---|---|---|
| Directory | `pages/` | `app/` |
| Data fetching | `getServerSideProps()` | `async function Page() { await... }` |
| Server components | ✗ | ✓ default |
| Shared layouts | `_app.tsx` workaround | `layout.tsx` first-class |
| What you'll write | Older codebases | All new projects (2023+) |

```ts
// Pages Router — data fetching lives outside the component
export async function getServerSideProps() {
  const data = await db.getStuff();
  return { props: { data } };
}

// App Router — component IS async, fetches its own data
async function Page() {
  const data = await db.getStuff(); // just works
  return <div>{data}</div>;
}
```

> [!NOTE]
> **Learn:** App Router. Skim Pages Router so you can read old codebases.

---

## 4. loader vs action (React Router)

**loader = READ. action = WRITE.**

| | `loader` | `action` |
|---|---|---|
| HTTP verb | GET | POST · PUT · DELETE · PATCH |
| Triggered by | Navigating to the route | Form submit / fetcher call |
| Runs where | Server (before render) | Server (before re-render) |
| Component reads via | `useLoaderData()` | redirect or returned data |

```ts
// loader — runs when user navigates to /chat
export async function loader() {
  const messages = await db.getMessages();
  return { messages }; // component reads this via useLoaderData()
}

// action — runs when user submits a form on /chat
export async function action({ request }: ActionFunctionArgs) {
  const form = await request.formData();
  const message = form.get("message");
  await db.saveMessage(message);
  return { ok: true };
}
```

> [!IMPORTANT]
> Component never touches the DB — it only calls `useLoaderData()` and submits
> `<Form>`. Data logic stays out of the component entirely.

---

## 5. How They All Relate (server/client boundary)

Three different layers expressing the same idea:

```text
SERVER                          │  CLIENT
────────────────────────────────┼──────────────────────────────
React Router:                   │
  loader / action               │  useLoaderData() / <Form>
  route-level, nav-triggered    │  component reads the result
                                │
Next.js / React 19:             │
  "use server" (Server Action)  │  "use client"
  callable like a function      │  useState, onClick, DOM APIs
                                │
Both:                           │
  DB queries, auth checks       │  UI state, event handlers
  secrets never leave here      │  needs the DOM to exist
```

**Key distinction:** `loader`/`action` are triggered by routing events. `"use server"` can be called
like a regular function from anywhere.

> [!NOTE]
> In React Router projects: loader/action **are** your server functions — no
> `"use server"` directive needed.
> In Next.js projects: async Server Components replace loaders, Server Actions replace
> action functions.

---

## 6. Wildcard Routes + Third-party Handlers

Pattern you'll see repeatedly with auth libraries (Better Auth, NextAuth, Auth.js):

```ts
// One wildcard catches ALL auth endpoints
route("api/auth/*", "routes/api.auth.$.ts")
// Covers: /api/auth/sign-in, /api/auth/sign-out, /api/auth/session, etc.
```

```ts
// api.auth.$.ts — just a pass-through to the auth library
export async function loader({ request }) {
  return auth.handler(request);
}
export async function action({ request }) {
  return auth.handler(request);
}
```

> [!NOTE]
> The `*` wildcard = "catch anything after this prefix." The `$` in the filename is
> React Router's way of expressing that wildcard in file-based mode.

---

## 7. clsx — Conditional className Utility

Tiny (239 B) utility for building `className` strings conditionally. Drop-in replacement
for the older `classnames` package but smaller and faster.

**Install:** `npm i clsx`

### Why not just template strings?

Template strings get ugly when you have more than one conditional class, especially with
Tailwind's long utility names.

```tsx
// Hard to read
<span
  className={`inline-flex rounded-full px-2 py-1 text-sm ${status === 'paid' ? 'bg-green-500 text-white' : 'bg-gray-100 text-gray-500'}`}
>

// Clean with clsx
<span
  className={clsx(
    'inline-flex rounded-full px-2 py-1 text-sm',
    {
      'bg-gray-100 text-gray-500': status === 'pending',
      'bg-green-500 text-white': status === 'paid',
    },
  )}
>
```

### Usage patterns

```tsx
import clsx from 'clsx';

// Strings (always included)
clsx('foo', 'bar'); // => 'foo bar'

// Conditional strings
clsx('foo', isActive && 'bar'); // => 'foo bar' (or just 'foo')

// Object syntax (keys included if value is truthy)
clsx({ 'bg-green-500': isPaid, 'bg-gray-100': isPending });

// Mixed arguments
clsx('base-class', [1 && 'conditional'], { 'extra': true });
```

### clsx/lite — string-only variant

If you only use the string / conditional-string pattern (no objects or arrays), import
`clsx/lite` to shave the bundle down to 140 B. Non-string arguments are ignored in this
mode.

```tsx
import clsx from 'clsx/lite';

clsx('hello', true && 'foo', false && 'bar'); // => 'hello foo'
clsx({ foo: true }); // => '' (objects ignored!)
```

> [!TIP]
> **When to use:** Any time you assemble `className` from more than one source —
> props, state, or mapped data. Essential with Tailwind CSS.

---

## Lessons Learned

_Add your own notes here as you go._

-
