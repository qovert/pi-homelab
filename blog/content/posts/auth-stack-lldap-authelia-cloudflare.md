---
title: "Auth Stack: LLDAP, Authelia, and Cloudflare Access"
date: 2026-06-13
draft: false
tags: ["homelab", "security", "authelia", "cloudflare", "lldap", "oidc"]
---

Running services at home means eventually answering the question: how many different logins to I want to deal with? A single Tailscale ACL or an nginx basic auth block will get you through the weekend, but if you're exposing things to the internet — or you just want a real login screen instead of a password prompt from 1998 — you need a real auth stack. Alternatively, you enjoy over complicating your life for no real good reason. 

Here's how mine works.

## The Components

Three pieces, each with a distinct job:

- **LLDAP** — a lightweight LDAP server. Stores users and groups. Basically a minimal directory service with a usable web UI; works great for my use case.
- **Authelia** — the authentication frontend. Handles login flows, MFA (TOTP), and acts as an OIDC provider for downstream services.
- **Cloudflare Access** — sits at the edge, in front of every publicly-exposed service. Requires a valid session before it'll forward traffic to the homelab at all.

Cloudflare Access is the bouncer. Authelia is the ID check. LLDAP is the list.

## Where Things Live

Everything auth-related runs on a dedicated Pi (pi-04). The rest of the cluster doesn't need to know much about auth — it just handles requests that have already gotten through.

```
Cluster nodes
┌──────────────────────────────────────────────────────┐
│                                                      │
│  pi-01              pi-02             pi-03          │
│  (chat, bot)        (nextcloud,       (kavita,       │
│                      miniflux,         uptime-kuma,  │
│                      syncthing)        homepage)     │
│                                                      │
│  pi-04              pi-fw                            │
│  (identity)         (firewall/ingress)               │
│  ├─ LLDAP           └─ cloudflared                   │
│  └─ Authelia                                         │
└──────────────────────────────────────────────────────┘
```

LLDAP and Authelia run as Podman containers on pi-04, managed by systemd Quadlets. They share a Podman network so Authelia can reach LLDAP on its internal address without exposing LDAP to the rest of the LAN.

## The Tale of Two... uh... One Login

When you hit a public-facing service — say, a dashboard or a media app — here's what happens:

```
Browser                Cloudflare            pi-homelab
   │                      │                      │
   │  GET /app            │                      │
   ├─────────────────────>│                      │
   │                      │ No valid session      │
   │  Redirect to login   │                      │
   │<─────────────────────│                      │
   │                      │                      │
   │  GET /auth (OIDC authorize)                  │
   ├─────────────────────────────────────────────>│
   │                      │               Authelia│
   │  Login page          │                      │
   │<─────────────────────────────────────────────│
   │                      │                      │
   │  POST credentials + TOTP                    │
   ├─────────────────────────────────────────────>│
   │                      │           LLDAP lookup│
   │                      │                      │
   │  OIDC callback (code)│                      │
   │<─────────────────────────────────────────────│
   │                      │                      │
   │  Exchange code for token                    │
   │  Cloudflare validates ID token              │
   │<─────────────────────│                      │
   │                      │                      │
   │  Session cookie set  │                      │
   │  Forwarded to app ──>│ ────────────────────>│
   │<─────────────────────────────────────────────│
```

Cloudflare Access is an OIDC *relying party* delegating authentication entirely to Authelia. After you log in, Cloudflare validates the token it gets back, sets a session cookie in your browser, and forwards subsequent requests to the origin (the homelab) without you ever interacting with Authelia again until the session expires.

## The OIDC Gotchas

Setting this up had a few sharp edges (hooray, learning?) worth documenting.

**Cloudflare Access doesn't call the userinfo endpoint.** Standard OIDC flow has the relying party call `/userinfo` to get claims like email and group membership after the token exchange. Cloudflare doesn't do this — it only reads what's in the ID token itself.

By default, Authelia puts email claims in userinfo, not the ID token. So Cloudflare was getting back a token with no email, rejecting it, and returning a generic "user email was not returned" error. Not a super obvious one.

The fix is a `claims_policy` in Authelia's config that explicitly pushes the claims you need into the ID token:

```yaml
identity_providers:
  oidc:
    claims_policies:
      cloudflare:
        id_token:
          - email
          - email_verified
          - name
          - preferred_username
          - groups
    clients:
      - client_id: cloudflare-access
        claims_policy: cloudflare
        # ...
```

**Authelia 4.39 requires an RSA key.** If you generate a JSON Web Key using an elliptic curve algorithm (ES256/EC P-256), Authelia will reject it at startup. The JWKS configuration needs at least one RS256 key.

**The client secret goes in YAML, not environment variables.** Authelia's env var naming scheme for OIDC client secrets doesn't follow the pattern I expected. The secret lives in the config file as `"$plaintext$<secret>"` or as a hashed value.

## The Tunnel

None of this requires opening a port on the router. Cloudflare Tunnel (cloudflared) runs on pi-fw and maintains an outbound connection to Cloudflare's edge. Traffic comes in from the internet, hits Cloudflare, gets validated by Access, and is forwarded inward through the tunnel to the appropriate service.

```
Internet → Cloudflare Edge → Access policy check
                                      │
                           ┌──────────▼──────────┐
                           │   Authelia (OIDC)    │
                           └──────────┬──────────┘
                                      │ validated
                           ┌──────────▼──────────┐
                           │  cloudflared tunnel  │
                           │  (outbound, pi-fw)   │
                           └──────────┬──────────┘
                                      │
                                 internal service
```

The tunnel config maps public hostnames to internal addresses. No inbound firewall rules, no dynamic DNS, no port forwarding.

## Internal vs. External Access

Not everything goes through Cloudflare. Internal services use a `lab.bitrot.me` subdomain that only resolves on the LAN. Split-horizon DNS handles the separation: internal hostnames point to Pi IP addresses directly, while public hostnames route through Cloudflare Tunnel.

The practical effect is that accessing a service from inside the network is direct (no Cloudflare hop, no Access check), while the same service from outside goes through the full auth flow. This is intentional — if you're on the LAN, you're already past the perimeter.

Whether that's the right call depends on your threat model. For a homelab, it's a reasonable tradeoff between convenience and security.

## The Ansible Side

All of this is configured via Ansible. LLDAP and Authelia each have container templates (Podman Quadlets) and env files that get rendered from `group_vars` and vault variables. Secrets — LDAP passwords, OIDC client secrets, the RSA private key — live in an Ansible Vault file and never touch the repo in plaintext.

Adding a new OIDC client means editing the Authelia config template and adding a new `clients` entry. Adding a new user or group is done through LLDAP's web UI and takes about 30 seconds.

It's more moving parts than managing each service's authentication options. But it's also a real SSO setup with MFA enrollment. Definitely a fun learning opportunity as well, rather than just relying on 'Login with GitHub'.
