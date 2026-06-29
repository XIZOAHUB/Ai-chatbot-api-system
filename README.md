# private.coder — setup guide

A private, password-protected coding-only chat assistant powered by your
Claude API key. Frontend on Cloudflare Pages, backend on Cloudflare Workers,
chat history persisted in Cloudflare KV.

## 1. Push to GitHub

```
git init
git add .
git commit -m "private.coder initial setup"
git remote add origin https://github.com/<you>/private-coder.git
git push -u origin main
```

Keep `worker/` and `frontend/` in the same repo — they deploy separately.

## 2. Deploy the Worker (backend)

```
cd worker
npm install
npx wrangler login
npx wrangler kv namespace create CHAT_KV
```

Copy the `id` it prints into `wrangler.toml` under `[[kv_namespaces]]`.

Set your secrets (these never get committed to git):

```
npx wrangler secret put ANTHROPIC_API_KEY
npx wrangler secret put AUTH_PASSWORD
npx wrangler secret put JWT_SECRET
```

- `ANTHROPIC_API_KEY` → your Claude API key
- `AUTH_PASSWORD` → the password you'll use to log into the app
- `JWT_SECRET` → any long random string (e.g. generate with `openssl rand -hex 32`)

Deploy:

```
npx wrangler deploy
```

It'll print a URL like `https://private-coder-backend.<you>.workers.dev` —
save this, you need it in step 4.

## 3. Deploy the frontend (Cloudflare Pages)

In the Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages** →
**Connect to GitHub** → pick this repo.

- **Build command:** leave empty
- **Build output directory:** `frontend`

Deploy. Note the Pages URL (e.g. `https://private-coder.pages.dev`).

## 4. Wire frontend ↔ backend together

1. Open `frontend/app.js`, set:
   ```js
   const API_BASE = "https://private-coder-backend.<you>.workers.dev";
   ```
2. Open `worker/wrangler.toml`, set:
   ```toml
   ALLOWED_ORIGIN = "https://private-coder.pages.dev"
   ```
3. Commit + push both changes. Pages redeploys automatically. Re-run
   `npx wrangler deploy` from `worker/` for the Worker change.

## 5. Use it

Open your Pages URL → enter the `AUTH_PASSWORD` you set → start chatting.
Conversations persist in KV, so they're there next time on any device.

## Notes

- Model is set to `claude-sonnet-4-6` in `worker/src/index.js` (`MODEL` const)
  — swap to `claude-opus-4-7` for tougher tasks or
  `claude-haiku-4-5-20251001` for cheap/fast, anytime.
- The "only do coding" restriction lives in `SYSTEM_PROMPT` in the same file
  — edit the wording there to loosen/tighten it.
- No streaming yet (replies arrive all at once) — easy to add later if you
  want a typing effect.
- Token is a simple signed expiry token stored in `localStorage`, valid 7
  days. Fine for solo personal use; not bank-grade auth.
