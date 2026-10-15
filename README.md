# ReelPilot (Cloudflare Worker + Cloudinary media storage)

ReelPilot is a private Cloudflare Workers dashboard prototype for preparing batches of 1–300 Instagram Reels and submitting them through Meta's official Instagram API. The batch's selected account IDs apply to every video. One cover URL is reused for the batch.

## Important limitations

- This is a starter implementation, not a production-certified publisher. Test with one Creator account and one short video first.
- Instagram permissions, app review, token expiry, media processing, cover behavior, rate limits and publishing eligibility depend on Meta and must be verified with your app/account.
- Publishing limits apply per account. A 300-video batch may take many days and may exceed free storage/processing quotas.
- Cloudinary is used for media storage instead of Cloudflare R2, so the Worker no longer needs an R2 binding or R2 subscription. Create a Cloudinary account and unsigned upload preset. Cloudinary's free credits/limits can change; check your dashboard and do not assume 300 videos will fit within the free allowance.
- Cloudinary unsigned upload presets can be abused if publicly exposed. Configure allowed formats and limits in Cloudinary, use a dedicated preset, and monitor usage. A production deployment should use signed uploads.
- Uploaded media URLs are public URLs so Meta can fetch them. Do not upload private or sensitive media.
- This project has not been live-tested against Meta's API.

## What you need

- Cloudflare account for Workers and D1.
- GitHub account (optional, useful for browser-based deployment workflows).
- Cloudinary account for public media URLs.
- Meta developer app configured for Instagram Login and content publishing.

## 1. Configure Cloudinary (no R2 needed)

1. Sign in to [Cloudinary Console](https://console.cloudinary.com/).
2. From the dashboard, copy your **Cloud name**.
3. Open **Settings → Upload** and create an **unsigned upload preset**. Restrict allowed formats to `mp4,mov,jpg,jpeg,png,webp` where the console permits, and set sensible file-size limits.
4. Copy the preset name. Never put your Cloudinary API secret in the browser or Worker; this project only needs the cloud name and unsigned preset name.
5. Check the free-plan credits and storage/bandwidth limits before attempting a large batch.

## 2. Create Cloudflare D1 database

Create a D1 database named `reelpilot-db` in Cloudflare Dashboard → Storage & databases → D1. Update `database_id` in `wrangler.toml` with the actual ID. Keep the existing migration path `migrations`.

## 3. Configure Worker variables and secrets

Set these in Worker Settings → Variables and Secrets (or using Wrangler if you have it):

- `ADMIN_PASSWORD` — long unique dashboard password (secret)
- `SESSION_SECRET` — random secret of at least 32 bytes (secret)
- `META_APP_ID` — Meta app ID (variable/secret)
- `META_APP_SECRET` — Meta app secret (secret)
- `META_REDIRECT_URI` — exact URL `https://YOUR-WORKER.workers.dev/auth/callback`
- `CLOUDINARY_CLOUD_NAME` — Cloudinary cloud name (variable)
- `CLOUDINARY_UPLOAD_PRESET` — unsigned preset name (variable)

Keep secrets out of GitHub and frontend files. Update `APP_BASE_URL` in `wrangler.toml` after the Worker URL is known.

## 4. Deploy

The repository includes `package.json` and `wrangler.toml`. You can use GitHub integration in the Cloudflare dashboard for a browser-based deploy, or install Node.js and use Wrangler locally. For local Wrangler deployment:

```bash
npm install
npx wrangler login
npx wrangler d1 create reelpilot-db
# Put the returned database ID in wrangler.toml
npx wrangler d1 migrations apply reelpilot-db --remote
npx wrangler secret put ADMIN_PASSWORD
npx wrangler secret put SESSION_SECRET
npx wrangler secret put META_APP_ID
npx wrangler secret put META_APP_SECRET
npx wrangler secret put META_REDIRECT_URI
npx wrangler secret put CLOUDINARY_CLOUD_NAME
npx wrangler secret put CLOUDINARY_UPLOAD_PRESET
npx wrangler deploy
```

If using Cloudflare's browser-based Git integration, create the D1 database and bind it as `DB`, apply the SQL in `migrations/0001_init.sql`, add the variables/secrets listed above, then deploy the Worker. Configure the Queue named `reelpilot-publish` and its producer/consumer as defined in `wrangler.toml` if your Cloudflare account UI supports it. The queue is required for background publication.

## 5. Meta setup

1. In [Meta for Developers](https://developers.facebook.com/apps/), configure Instagram Login for professional accounts.
2. Add the exact deployed callback URL as an allowed OAuth redirect URI.
3. Request the Instagram business basic and content-publishing permissions supported by the selected login flow.
4. Use Meta's test users/app roles while in development. App review/advanced access may be needed for other accounts.
5. Connect one Creator account from ReelPilot. Never enter Instagram passwords into ReelPilot.

Official documentation: https://developers.facebook.com/docs/instagram-platform/instagram-api-with-instagram-login/content-publishing/

## 6. First test

1. Sign into the private dashboard.
2. Connect one eligible Creator account.
3. Select one short MP4 video and one JPG cover.
4. Select only that account, enter a short caption, and queue the batch.
5. Confirm Cloudinary returns a secure URL and that Instagram confirms publication before testing more videos.
6. Expand gradually. Do not start with 300 videos.

## Known work still required before production

- Robust OAuth state binding to the authenticated admin session and stronger CSRF protections.
- Token refresh/re-authorisation lifecycle and disconnect-account UI.
- Durable long-horizon scheduling, rate-limit handling and reconciliation of uncertain publish responses to minimize duplicate posts.
- Multipart/resumable uploads for very large files and better progress persistence.
- Live tests of Meta's current API, shared `cover_url` support, token permissions, and all job recovery paths.

## Free-tier and cost warning

Cloudflare Workers and D1 have free quotas, and Cloudinary has a limited free allowance. These are not unlimited services. Cloudinary usage can be consumed by video storage, transformations and delivery; check current provider pricing before using large batches. This setup removes the mandatory Cloudflare R2 subscription dependency, but cannot guarantee every 300-video batch stays within free quotas.


## v67 — Import Reels from an authorized profile

The dashboard now includes **Import Reels from a profile**. Choose a connected Instagram professional account, enter that same account's profile URL, and select Reels returned by Meta's official `/{user-id}/media` endpoint. Captions are prefilled and editable; selected items can be added to the current batch.

**Important limits:** this does not scrape arbitrary public profiles or bypass permissions. The profile URL must match the username of an account authorized in this ReelPilot instance. Meta may not return every Reel or a usable `media_url`; the URL can expire, and the publishing API may fail if Meta cannot fetch it. Always test with one Reel before a larger batch. For another account, connect and authorize that account first. The importer currently requests the first page (up to 50 items) and reports if Meta indicates more pages; pagination is not yet implemented.
