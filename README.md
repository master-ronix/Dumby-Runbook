# Webhook Inbox

A tiny static site with a real serverless webhook endpoint. POST any text to
your webhook URL and it shows up on the page automatically.

## Does Vercel support this?

Yes. **Vercel Functions** (serverless, included on every plan — even the free
Hobby tier) run the backend, and **Vercel Blob** (Vercel's built-in object
storage) holds the received messages between requests. No third-party
database or separate account signup needed — just a Vercel account.

## What's inside

```
webhook-inbox/
├── vercel.json                 # tells Vercel where the static site lives
├── package.json                 # dependencies: @vercel/blob, @vercel/functions
├── public/
│   └── index.html               # the page — shows your live webhook URL + the log
└── api/
    ├── _lib/
    │   └── store.js              # shared Vercel Blob read/write helper (not a route)
    ├── webhook.js                # POST /api/webhook  — receives and stores content
    ├── messages.js                # GET  /api/messages — the page polls this
    └── clear.js                   # POST /api/clear    — wipes the log
```

Vercel turns every file directly inside `api/` into a route matching its
path (`api/webhook.js` → `/api/webhook`), and dispatches by the exported
`GET`/`POST`/`OPTIONS` functions — so there's no routing config file to
maintain. `api/_lib/` starts with an underscore, which Vercel's router
skips, so it's safe to keep shared code there.

## One-time setup: create a Blob store

The functions need somewhere to store messages. Do this once, before your
first deploy (or right after — either order works):

1. In the [Vercel dashboard](https://vercel.com/dashboard), open your
   project (or create it first by importing/deploying, then come back).
2. Go to the **Storage** tab → **Create Database** → **Blob**.
3. Set access to **Private** (the app reads/writes it from the server only —
   nothing here should be reachable by a guessed URL).
4. Connect it to this project, including the **Development** environment if
   you want to test locally with `vercel dev`.

That's it — no code changes needed. Vercel adds the credentials as
environment variables automatically, and `@vercel/blob` picks them up on its
own.

## Deploy it

**Option A — Vercel CLI (fastest, no Git required)**
```bash
npm install -g vercel
cd webhook-inbox
vercel login
vercel --prod
```
Accept the defaults it detects (framework: **Other**, output directory:
`public`) — `vercel.json` already pins these, so you shouldn't be prompted.

**Option B — Git (best if you'll keep iterating on this)**
1. Push this folder to a new GitHub (or GitLab/Bitbucket) repo.
2. In Vercel: **Add New → Project** → import the repo.
3. Vercel reads `vercel.json` and `package.json` automatically, so there's
   nothing to configure.
4. Deploy.

Either way, once it's live, open the site — the homepage displays your real
webhook URL automatically (it reads its own domain, so this works no matter
what your Vercel URL ends up being).

## Using the webhook

```bash
curl -X POST --data "hello world" https://YOUR-SITE.vercel.app/api/webhook
```

It also understands JSON and form bodies:
- `{"text": "..."}` (or `message` / `content`) — the value is extracted and shown
- any other JSON — shown pretty-printed
- anything else — stored and shown exactly as sent

The log keeps the most recent 200 messages; older ones roll off automatically.

## Optional: add a shared secret

By default, anyone with the URL can post to it. To require a secret:

1. In Vercel: **Project → Settings → Environment Variables** → add
   `WEBHOOK_SECRET` with any value you choose, scoped to whichever
   environments you want it enforced on → redeploy. (Or via CLI:
   `vercel env add WEBHOOK_SECRET`.)
2. Send requests with it, either as a header or a query parameter:
   ```bash
   curl -X POST --data "hello" \
     -H "X-Webhook-Secret: your-value" \
     https://YOUR-SITE.vercel.app/api/webhook
   ```
   or `https://YOUR-SITE.vercel.app/api/webhook?token=your-value`

The "Clear log" button on the page will prompt you for this secret too, if
it's set.

## Good to know

- The page checks for new messages every 4 seconds. That's not instant push,
  but it's simple and needs no extra infrastructure (real-time push over
  serverless functions would need a third-party service like Pusher/Ably).
- Reads pass `useCache: false` so a new message is visible on the very next
  poll instead of possibly waiting behind Vercel Blob's CDN cache (which can
  hold an overwritten file for up to 60 seconds otherwise). This costs a
  little more per read than a cached one, but at this app's scale (small
  JSON, occasional polling) it's well within the free Hobby tier — see
  [vercel.com/pricing](https://vercel.com/pricing) for current numbers.
- Writes use Blob's ETag-based conditional writes (`ifMatch`) with a few
  retries, so two webhook posts arriving at nearly the same instant don't
  silently overwrite each other.
- Multipart form uploads (file uploads) aren't parsed specially — this is
  built for text content, as requested.
- Want to test locally before deploying? Install the Vercel CLI, run
  `vercel link` once to connect this folder to your Vercel project, then
  `vercel dev` — functions, env vars, and Blob (if connected to the
  Development environment) all work the same way locally as they do once
  deployed.
