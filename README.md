# Setting Up a Custom Domain on GitHub Pages

A complete, ordered walkthrough — from a freshly registered domain to a working HTTPS site with apex/www redirects and takeover protection.

**Demo values used throughout.** Replace these with your own:

| Placeholder | Example value | What it is |
|---|---|---|
| Domain | `example-studio.com` | The apex (root/naked) domain you registered |
| GitHub account | `octo-dev` | Your GitHub username or organization name |
| Default Pages domain | `octo-dev.github.io` | The hostname GitHub gave your site |
| Repository | `octo-dev/portfolio` | The repo publishing the site |

---

## Before you start

You need three things in place:

1. A **registered domain** at any registrar (Namecheap, Cloudflare, Porkbun, GoDaddy…).
2. A **working GitHub Pages site** — confirm `https://octo-dev.github.io` loads before touching DNS. Debugging a broken site and broken DNS at the same time is miserable.
3. **Admin access to the repository**, and access to your registrar's DNS panel.

### Know which kind of site you have

This changes the `www` CNAME target, which is the single most common mistake.

| Site type | Repo name | Default domain |
|---|---|---|
| User / org site | `octo-dev.github.io` | `octo-dev.github.io` |
| Project site | anything, e.g. `portfolio` | `octo-dev.github.io/portfolio` |

**In both cases the CNAME target is `octo-dev.github.io` — never include the repository name.** DNS points at the account; the repo is selected by the `CNAME` file inside it.

### Know which kind of domain you're configuring

- **Apex domain** — `example-studio.com`. No subdomain. Needs **A records**.
- **Subdomain** — `www.example-studio.com`, `blog.example-studio.com`. Needs a **CNAME record**.

The recommended setup, covered below, uses **both**: apex as the primary, `www` redirecting to it.

---

## Step 1 — Verify the domain first

> Do this **before** Step 2. GitHub explicitly recommends verifying prior to adding the domain to a repository, and the docs warn that configuring DNS at your registrar without the domain registered on GitHub first could let someone else host a site on one of your subdomains.

### Why this exists

GitHub Pages routes requests by matching the incoming `Host` header against the `CNAME` file in repositories. There is **no ownership check** in that matching — whichever repo claims `example-studio.com` first gets served at that hostname.

That's a problem the day your claim lapses but your DNS doesn't. If you delete or rename the repo, downgrade your plan, or disable Pages while your A records still point at GitHub, the hostname becomes an unclaimed dangling record. Anyone can then create a repo, put `example-studio.com` in a `CNAME` file, and GitHub will serve their content at your domain — with a valid TLS certificate, because the ACME challenge resolves through *your* DNS. People scan certificate-transparency logs for exactly these dangling records.

Verification reserves the hostname to your account permanently, so the dangling window never opens.

### How

1. Click your profile picture → **Settings**. *(Profile settings, not repository settings — domain verification happens at the account level.)*
2. Sidebar → **Pages**.
3. Click **Add a domain**.
4. Enter `example-studio.com` → **Add domain**.
5. GitHub shows you a hostname and a token. Add this at your registrar:

   | Type | Host | Value | TTL |
   |---|---|---|---|
   | TXT | `_github-pages-challenge-octo-dev` | *(the token GitHub shows)* | Automatic |

   **Enter only the label in the Host field** — leave off `.example-studio.com`. Most DNS panels append your domain automatically. GitHub displays the fully-qualified name because that's what resolves; typing it in full produces `_github-pages-challenge-octo-dev.example-studio.com.example-studio.com` and the check fails.

   Don't add quotes around the token. The DNS server adds those when it serves the record.

6. Confirm it's live:

   ```bash
   dig _github-pages-challenge-octo-dev.example-studio.com TXT +short
   ```

   You should see your token echoed back in quotes.

7. Back in Settings → Pages, click **Verify** (or the `…` menu next to the domain → **Continue verifying** if you navigated away).

**Keep that TXT record forever.** GitHub re-checks it. Remove it and the verification lapses, putting you back in the exposed state.

> Verification covers the domain and its **immediate** subdomains — `www.example-studio.com`, `blog.example-studio.com`, and so on are protected by this one record.

---

## Step 2 — Add the custom domain to the repository

1. Go to the repo → **Settings** → sidebar **Pages**.
2. Under **Custom domain**, type `example-studio.com` → **Save**.

### What just happened

If you publish **from a branch**, GitHub commits a file called `CNAME` to the root of your source branch containing your domain. That file is what tells the Pages build which hostname this repo claims. Don't delete it.

If you publish **from a custom GitHub Actions workflow**, no `CNAME` file is created and any existing one is ignored — the domain lives purely in the repo settings.

> **Using a static site generator and building locally?** Pull that commit down before your next push, or your local build will overwrite the repo root and delete the `CNAME` file, breaking the domain.

You'll now see **DNS Check in Progress**. That's expected — nothing points at GitHub yet.

### Which name goes in the field?

Whichever you type becomes the **canonical** host; the other redirects to it.

- Enter `example-studio.com` → `www.example-studio.com` redirects to the bare domain.
- Enter `www.example-studio.com` → the bare domain redirects to `www`.

Either is fine. Pick one and be consistent — this guide uses the apex.

---

## Step 3 — Point DNS at GitHub

### 3a. Check your nameservers

At your registrar, confirm the domain is using the registrar's own DNS (e.g. Namecheap's **BasicDNS**) and not custom nameservers pointing elsewhere. Records you add in the panel are only authoritative if that panel's nameservers are the ones answering queries.

### 3b. Delete the default records

Registrars pre-fill parking records that will fight with yours. Remove anything on `@` or `www`, typically:

- `CNAME` · `@` or `www` → `parkingpage.<registrar>.com`
- `URL Redirect Record` · `www` → an unmasked redirect

GitHub's docs state this plainly: if your DNS provider automatically sets a default record, remove it before continuing.

### 3c. Add the records

**Apex — four A records:**

| Type | Host | Value | TTL |
|---|---|---|---|
| A | `@` | `185.199.108.153` | Automatic |
| A | `@` | `185.199.109.153` | Automatic |
| A | `@` | `185.199.110.153` | Automatic |
| A | `@` | `185.199.111.153` | Automatic |

**`www` — one CNAME record:**

| Type | Host | Value | TTL |
|---|---|---|---|
| CNAME | `www` | `octo-dev.github.io.` | Automatic |

**_Optional_ — (IPv6) four AAAA records:**

| Type | Host | Value | TTL |
|---|---|---|---|
| AAAA | `@` | `2606:50c0:8000::153` | Automatic |
| AAAA | `@` | `2606:50c0:8001::153` | Automatic |
| AAAA | `@` | `2606:50c0:8002::153` | Automatic |
| AAAA | `@` | `2606:50c0:8003::153` | Automatic |

Keep the *A* records if you add these. GitHub recommends *A* alongside *AAAA* because IPv6 adoption is still uneven globally.

### Why the records differ

An apex domain **cannot** hold a CNAME. DNS forbids it: the apex must also carry SOA and NS records, and a CNAME can't coexist with other records on the same name. So the apex needs literal IPs via A records. GitHub publishes four for redundancy across their edge — add all four so traffic survives one going down.

A subdomain has no such restriction, and a CNAME is *better* there because it points at a name. If GitHub ever renumbers its IPs, `www` follows automatically while hardcoded A records would silently break.

### Alternative: ALIAS / ANAME

If your provider supports them, one record replaces all four A records:

| Type | Host | Value |
|---|---|---|
| ALIAS or ANAME | `@` | `octo-dev.github.io` |

These are provider-specific pseudo-records that behave like a CNAME but are legal at the apex — the nameserver resolves the target itself and returns the resulting A records to the client, so no CNAME is actually stored at the apex. The upside is automatic IP updates. Use **either** this **or** the A records, not both.

### Two things that will break your setup

> **Never point `www` at your apex domain.** A CNAME from `www` → `example-studio.com` routes resolution through your own apex instead of GitHub's edge directly. GitHub's docs warn this breaks HTTPS enforcement and can stop the subdomain reaching your site at all. Point it at `octo-dev.github.io`.

> **Never use wildcard records** like `*.example-studio.com`. They put you at immediate risk of takeover **even with the domain verified** — verification covers immediate subdomains, so verifying `example-studio.com` blocks `a.example-studio.com` but someone could still take over `b.a.example-studio.com` through the wildcard. Add subdomains explicitly, one record each.

---

## Step 4 — Wait and verify

DNS changes can take up to 24 hours to propagate, though with a 30-minute TTL it's usually 10–60 minutes.

```bash
# Apex — expect the four GitHub IPs
dig example-studio.com +noall +answer -t A

# www — expect the CNAME, then the same four IPs
dig www.example-studio.com +short
```

Still seeing your registrar's parking IP? Your local resolver is caching. Query a public one to see the real state:

```bash
dig @8.8.8.8 example-studio.com +short
```

> **On Windows,** `dig` isn't included. Use PowerShell's `Resolve-DnsName example-studio.com -Type A`, or install BIND.

Then reload the repo's Pages settings. **DNS Check in Progress** should become a green **DNS check successful**.

Confirm the site actually serves:

```bash
curl -sI http://example-studio.com | head -n 3
```

---

## Step 5 — Enable HTTPS

The **Enforce HTTPS** checkbox stays greyed out for a while after the DNS check passes. These are two separate stages.

### What's happening

Once DNS verifies, GitHub queues a certificate request to Let's Encrypt. Validation uses an HTTP-01 challenge: Let's Encrypt requests `http://example-studio.com/.well-known/acme-challenge/<token>` and expects GitHub's edge to return the matching token. That only works once your A records resolve to GitHub — which is why the order matters.

Until the issued certificate is installed across GitHub's edge nodes, the settings page keeps showing the old "not properly configured to support HTTPS" message, because it renders the last known cert status rather than re-evaluating live. **The page doesn't poll — reload it yourself.**

Typically 15–30 minutes; occasionally up to 24 hours.

### Then

Tick **Enforce HTTPS**. All traffic now redirects to `https://`.

```bash
curl -sI https://example-studio.com | head -n 3
curl -sI https://www.example-studio.com | head -n 5   # expect 301 → https://example-studio.com/
```

### Certificate timing gotcha

The certificate covers whichever hostnames resolved correctly **at the moment it was generated**. Set up apex and `www` together and you get one cert covering both.

Add `www` *after* the cert was issued for the apex alone, and `https://www.example-studio.com` throws a certificate-mismatch warning — the browser rejects it before GitHub can send its redirect.

**To force a reissue:** Settings → Pages → clear the Custom domain field → **Save** → re-enter the domain → **Save**. This re-queues the certificate request from scratch. It's harmless and also clears most stuck states.

---

## Step 6 — Final checks

- [ ] `http://example-studio.com` → redirects to HTTPS
- [ ] `https://example-studio.com` → loads, padlock, no warnings
- [ ] `https://www.example-studio.com` → 301s to the apex, no cert warning
- [ ] Domain shows as **Verified** in account Settings → Pages
- [ ] TXT verification record still present in DNS
- [ ] `CNAME` file present in the repo root (branch-published sites)
- [ ] No wildcard DNS record on the domain

Only now put the URL anywhere public — social profiles, email signature, Google Search Console. Anything crawling it earlier records the `http://` version.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Stuck on "DNS Check in Progress" | Records not propagated, or parking records still present | `dig @8.8.8.8` to see real state; delete leftover registrar defaults |
| DNS check passes, Enforce HTTPS greyed out | Certificate not issued yet | Wait, then hard-refresh the settings page |
| Still greyed after 24h | CAA record blocking Let's Encrypt | `dig example-studio.com CAA +short` — empty is fine; otherwise allow `letsencrypt.org` |
| Still greyed, CAA clean | Stale AAAA records pointing elsewhere | `dig example-studio.com AAAA +short` — should be empty or GitHub's IPs only |
| Cert warning on `www` only | Cert issued before `www` existed | Remove and re-save the custom domain to force reissue |
| "Domain is already taken" | Claimed by another repo | Remove it from that repo's Pages settings first |
| Domain breaks after a deploy | Local build overwrote the `CNAME` file | Pull the commit that added it; add it to your generator's static/public folder |
| 404 at the custom domain | `CNAME` file missing or has the wrong domain | Check the repo root on the published branch |

---

## Quick reference

**GitHub Pages IPs (apex A records)**
```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

**IPv6 (apex AAAA records)**
```
2606:50c0:8000::153
2606:50c0:8001::153
2606:50c0:8002::153
2606:50c0:8003::153
```

**Full record set for `example-studio.com` + `www`**

| Type | Host | Value |
|---|---|---|
| A | `@` | `185.199.108.153` |
| A | `@` | `185.199.109.153` |
| A | `@` | `185.199.110.153` |
| A | `@` | `185.199.111.153` |
| CNAME | `www` | `octo-dev.github.io.` |
| AAAA | `@` | `2606:50c0:8000::153` |
| AAAA | `@` | `2606:50c0:8001::153` |
| AAAA | `@` | `2606:50c0:8002::153` |
| AAAA | `@` | `2606:50c0:8003::153` |
| TXT | `_github-pages-challenge-octo-dev` | *(verification token)* |

---

## Sources

- [Managing a custom domain for your GitHub Pages site](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)
- [Verifying your custom domain for GitHub Pages](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages)
- [Troubleshooting custom domains and GitHub Pages](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/troubleshooting-custom-domains-and-github-pages)
- [Securing your GitHub Pages site with HTTPS](https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https)
