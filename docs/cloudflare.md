# Cloudflare Workers deployment (personal fork)

This setup hosts the Smashrun MCP server on Cloudflare Workers with GitHub login and a pre-registered OAuth client for ChatGPT. It is specific to this fork; the generic Streamable HTTP transport does not require Cloudflare.

## One-time setup

1. Enable **Workers Free** and a `workers.dev` subdomain in Cloudflare.
2. Create a Workers KV namespace named `OAUTH_KV` and set its ID in `wrangler.jsonc`. Use the **remote** namespace for OAuth client registration; Wrangler's default local KV is separate.
3. Create a GitHub **OAuth App** with callback `https://<worker>.<subdomain>.workers.dev/callback` (optionally also `http://localhost:8788/callback` for local testing).
4. Obtain a Smashrun user access token. Create a git-ignored `.dev.vars` file containing:

   ```dotenv
   GITHUB_CLIENT_ID=...
   GITHUB_CLIENT_SECRET=...
   ALLOWED_GITHUB_USER_ID=...
   SMASHRUN_ACCESS_TOKEN=...
   ```

5. Register **one confidential ChatGPT OAuth client** in remote `OAUTH_KV` using `getOAuthApi(...).createClient()` from `@cloudflare/workers-oauth-provider`. Use `tokenEndpointAuthMethod: "client_secret_basic"` and `redirectUris: ["https://chatgpt.com/connector_platform_oauth_redirect"]`. Run registration via a **temporary localhost-only** Worker (`--ip 127.0.0.1`) with the KV binding set to `remote: true`; never deploy the registration endpoint. Save the returned `clientId` and `clientSecret` in a password manager, then delete local temporary files. The client secret is returned in plaintext only at creation.

## Deploy and connect

```bash
npm run build
npx wrangler deploy --secrets-file .dev.vars
```

In ChatGPT, add a custom MCP server at `https://<worker>.<subdomain>.workers.dev/mcp`. Select **User-Defined OAuth Client**, enter the saved `clientId` and `clientSecret`, choose `client_secret_basic`, and use scope `mcp:read`. The OAuth client registration and tokens reside in `OAUTH_KV`; the ChatGPT client secret does **not** belong in the Worker secrets.

Verify that an unauthenticated `POST /mcp` returns **HTTP 401** and that `/.well-known/oauth-authorization-server` advertises the expected HTTPS issuer and endpoints. Then complete the ChatGPT consent and GitHub login and test `get_user_info` and `get_activity`.

## Maintenance and security

- The README notes that Smashrun **user tokens expire after about 60 days**. Obtain a new token and rotate it with `npx wrangler secret put SMASHRUN_ACCESS_TOKEN`; also update the local `.dev.vars` before any subsequent `--secrets-file` deploy to avoid restoring an old token.
- Keep `.dev.vars`, OAuth client secrets, registration scripts, and exported client JSON **out of Git**. Never expose a public `/register` endpoint.
- Access is granted only after GitHub login matches `ALLOWED_GITHUB_USER_ID`; the `/mcp` endpoint still requires a valid OAuth bearer token. Keep the deployment on Workers Free if avoiding usage-based Workers charges is important.
