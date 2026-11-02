# Windule Deployment and Email Forwarding

## Deploy Site on Cloudflare Workers Builds

Pushes to `main` deploy with Workers Builds. `wrangler.toml` sets `assets.directory = "./public"`, so only that folder is uploaded.

Put files the live site should serve in `public/` (`index.html`, `styles.css`, and any images, icons, favicons, `robots.txt`, sitemap, or `.well-known` files the page or app links need). Leave repo and config files at the repository root — `.git`, markdown docs, and `wrangler.toml` are not part of the upload. A file committed outside `public/` is not served, including future notes.

Do not point the asset directory back at the repository root. That publishes the whole checkout, including `.git` and docs.

Custom domain: `windule.com` (and optionally `www.windule.com`) in the Worker domain settings.

## Set Up Email Forwarding for support@windule.com

Goal:
- Receive mail at `support@windule.com`
- Forward to `markduenas@gmail.com`

Steps in Cloudflare:

1. Ensure `windule.com` is using Cloudflare nameservers.
2. Go to `Email` > `Email Routing` and click `Get started`.
3. Add destination address: `markduenas@gmail.com`.
4. Open the verification email in Gmail and confirm destination ownership.
5. Create custom address rule:
   - Custom address: `support`
   - Action: `Send to`
   - Destination: `markduenas@gmail.com`
6. Save the rule.

Cloudflare will add required MX/TXT records automatically for Email Routing.

## Verify End-to-End

1. Send a test email from a non-Gmail account to `support@windule.com`.
2. Confirm it arrives in `markduenas@gmail.com`.
3. In Cloudflare Email Routing logs, confirm message status is forwarded/delivered.

## Important

Cloudflare Email Routing forwards inbound mail only.  
If you also want to send as `support@windule.com`, configure an SMTP sender separately in Gmail (`Settings` > `Accounts and Import` > `Send mail as`).
