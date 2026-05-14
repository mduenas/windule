# Windule Deployment and Email Forwarding

## Deploy Site on Cloudflare Pages (Free)

1. Create a GitHub repo for this folder and push:
   - `index.html`
   - `styles.css`
2. In Cloudflare Dashboard, go to `Workers & Pages` > `Create application` > `Pages` > `Connect to Git`.
3. Select the repo.
4. Build settings:
   - Framework preset: `None`
   - Build command: leave empty
   - Build output directory: `/`
5. Deploy.
6. Add your custom domain `windule.com` (and optionally `www.windule.com`) in the Pages project domain settings.

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
