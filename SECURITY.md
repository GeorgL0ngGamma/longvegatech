# Static site security

The site is served by the Cloudflare Worker named `longvegatech`, connected to this repository. GitHub Pages is disabled. The website has no application JavaScript, PHP, forms, API, database, or required runtime secrets.

## Deployment controls

`.assetsignore` excludes every file by default and includes only:

- `index.html`
- `styles.css`
- `disclaimer.html`
- `privacy-policy.html`
- `terms-of-service.html`
- `robots.txt`
- `llm.txt`
- `llms.txt`

Adding any other public asset requires deliberately editing this list. The same restriction applies to Git internals, environment files, backups, source maps, local review files, build output, and new directories.

`_headers` is read separately by Wrangler and remains excluded from public assets. It configures a restrictive Content Security Policy, clickjacking protection, MIME sniffing protection, a referrer policy, browser feature restrictions, and HTTPS transport policy.

`.gitignore` also excludes new files from ordinary Git staging until their names are explicitly allowed. It cannot prevent `git add --force`, protect already tracked files, or detect a credential written into an allowed HTML file.

`wrangler.jsonc` explicitly disables the workers.dev route and version URLs, retains the production custom domain, and disables application fallback for missing files. Missing files must return 404 rather than the home page.

The source PDF has been removed from the current repository tree and is not deployed. Historical public Git commits can still contain it. Removing a public file cannot revoke copies already downloaded.

## Deployment

Use the existing Cloudflare Git integration, production branch `main`, repository root, and the root Wrangler configuration. No site build step is needed. The deploy command can be pinned to:

```sh
npx wrangler@4.147.0 deploy
```

No new host or Worker is needed. Restrict the GitHub app installation to this repository. Keep the production deploy command pointed at this configuration, and do not override the asset directory or bypass the allowlist.

After deployment, check:

- The home page, stylesheet, legal pages, and crawler files work.
- `/.git/HEAD`, `/.git/config`, `/.git/index`, and `/.git/logs/HEAD` return 404 or 403.
- `/.env`, `/functions/.env`, `/review/`, `/_headers`, and `/wrangler.jsonc` return 404 or 403.
- The removed PDF URL returns 404 or 403.
- CSP, X-Content-Type-Options, and X-Frame-Options appear on successful static responses.

## Cloudflare account settings

These controls require an authenticated account session and should be reviewed in the dashboard:

1. **Purge old cached content once after the secure deployment.** The earlier deployment exposed Git metadata, and a Git config response was cached. Use Caching > Configuration > Purge Everything for this small site. Do not roll back to a deployment created before the asset allowlist.
2. **Add a WAF custom Block rule for unexpected paths or methods.** This provides a second check even if a later deployment accidentally broadens its assets. The proposed expression below allows this site's public routes and leaves Cloudflare-managed `/cdn-cgi/` endpoints alone.
3. **Disable non-production branch builds and public Preview environments** when unused. The Wrangler configuration disables production workers.dev and version URLs; Preview environments are a separate control. Review and remove/protect old public preview deployments.
4. **Use least privilege.** Keep the GitHub installation limited to this repository, limit production deploy access, use account-specific deployment tokens with only required permissions, and remove unused KV/R2/D1/service bindings and build/runtime secrets.
5. **Protect account access and releases.** Use security keys/passkeys or MFA for Cloudflare and GitHub, review collaborators and tokens, and protect `main` and the deployment configuration from unreviewed changes. Keep GitHub secret scanning and push protection enabled.
6. **Require HTTPS.** Enable Always Use HTTPS and minimum TLS 1.2. HSTS is already sent for this hostname. Avoid broadening HSTS to all subdomains until those subdomains have been checked.
7. **Monitor blocked probes and deployments.** Use Security Events and audit logs; consider rate limits for sustained scanning without blocking ordinary readers and search crawlers. Keep script-injecting features such as Rocket Loader, Email Address Obfuscation, and optional analytics beacons off for this script-free site.

### Proposed WAF rule

Action: **Block**. Apply to the `longvegatech.com` zone.

```text
(http.host eq "longvegatech.com" and
 not starts_with(http.request.uri.path, "/cdn-cgi/") and
 (
   not http.request.method in {"GET" "HEAD"} or
   not http.request.uri.path in {
     "/" "/index.html" "/styles.css"
     "/disclaimer" "/disclaimer/" "/disclaimer.html"
     "/privacy-policy" "/privacy-policy/" "/privacy-policy.html"
     "/terms-of-service" "/terms-of-service/" "/terms-of-service.html"
     "/robots.txt" "/llm.txt" "/llms.txt"
   }
 ))
```

Review existing rules and verify the public pages after enabling it. Add future public paths explicitly. This rule is an additional access control; robots.txt is only a voluntary crawler policy.

## Limits

The upload allowlist prevents files outside the listed names from becoming website assets under this configuration. It does not make all leaks impossible: an authorized file can contain a secret, an administrator can change the policy, an account or deployment tool can be compromised, and previously public copies cannot be recalled. Secret scans are useful checks, not proofs of absence.

## References

- [Cloudflare asset exclusions](https://developers.cloudflare.com/workers/static-assets/binding/#ignoring-assets)
- [Static response headers](https://developers.cloudflare.com/workers/static-assets/headers/)
- [Workers build configuration](https://developers.cloudflare.com/workers/ci-cd/builds/configuration/)
- [Version URLs](https://developers.cloudflare.com/workers/versions-and-deployments/version-urls/)
- [Branch builds](https://developers.cloudflare.com/workers/ci-cd/builds/build-branches/)
- [WAF custom rules](https://developers.cloudflare.com/waf/custom-rules/)
- [Cache purge](https://developers.cloudflare.com/cache/how-to/purge-cache/purge-everything/)
