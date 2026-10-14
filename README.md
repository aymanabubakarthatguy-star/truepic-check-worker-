# TruePic Check Worker

Cloudflare Worker that gives an **educational visual-risk assessment** of an uploaded image using OpenRouter's free router (`openrouter/free`). It is never proof, never identifies people, and never states an image is definitely real, AI-generated, manipulated, or fake.

## Setup

1. Install: `npm install`
2. Create the rate-limit KV namespace and paste the returned `id` into `wrangler.toml`:
   ```
   npx wrangler kv namespace create RATE_LIMIT
   ```
3. Set allowed frontend origins in `wrangler.toml` (`ALLOWED_ORIGINS`, comma-separated, exact origins, no trailing slash).
4. Add your OpenRouter key as a **secret** (get one at https://openrouter.ai/keys):
   ```
   npx wrangler secret put OPENROUTER_API_KEY
   ```
5. Deploy: `npm run deploy`

### Local development
Copy `.env.example` to `.dev.vars`, fill in `OPENROUTER_API_KEY`, then run `npm run dev`. `.dev.vars` is git-ignored. Never commit real keys.

## API

`POST /api/check` as `multipart/form-data`:

| Field | Required | Notes |
|---|---|---|
| `image` | yes | JPEG, PNG or WebP, max 8 MB (verified by file signature) |
| `consent` | yes | `true` / `on` – user confirms they have the right to upload the image |

Success (200):
```json
{
  "riskLevel": "low | medium | high",
  "riskScore": 0,
  "observations": ["..."],
  "nextSteps": ["..."],
  "disclaimer": "...",
  "remainingToday": 2
}
```

Errors: `400` missing/invalid input or consent, `403` origin not allowed, `413` over 8 MB, `415` unsupported type, `429` daily limit (3 per IP), `503` analysis unavailable:
> The free analysis service is temporarily unavailable. Please try again later.

A 503 is returned if the free model can't analyse images, returns invalid JSON, times out (25 s), or is unavailable. A 503 does not use up the user's daily check.

## Test
```
curl -X POST https://YOUR-WORKER.workers.dev/api/check \
  -H "Origin: https://your-frontend-domain.example" \
  -F "consent=true" -F "image=@photo.jpg"
```

## Privacy & safety
- Images are processed in memory and sent to OpenRouter; they are **not stored** by this Worker.
- Rate limiting stores only a SHA-256 hash of IP + date with a ~26 h expiry, never the image or result.
- The API key is read only from `env.OPENROUTER_API_KEY`; it is never returned, logged, or included in errors. The Worker logs nothing about requests or upstream responses.
- The disclaimer is fixed server-side, and output lines containing definitive claims or identity/trait inferences are filtered out.
- No facial recognition or reverse-image search is performed.
- Note: free-tier routed models may have their own data/logging policies; review OpenRouter's terms.

## Cleaning up the old provider
Delete the old provider's Worker secret (`npx wrangler secret delete <OLD_SECRET_NAME>`), rotate that key, and remove its variables from any frontend, CI, or `.env` files.
