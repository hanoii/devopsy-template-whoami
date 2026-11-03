# devopsy-template-whoami

The smallest devopsy project, [traefik/whoami](https://github.com/traefik/whoami)
behind a [devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik)
server, released with [devopsy](https://github.com/hanoii/devopsy-cli).
Use it to try targets, releases, domains and certificates, or as a
starting point.

## Using this template

This template is meant to be copied and changed. Start a project from it
on GitHub ("Use this template"), or clone it and keep this repository as a
remote (`upstream`) to pull its changes when you choose. Nothing from this
repository reaches your copy or your servers until you pull its changes
and release them.

## Try it

Point the project at your server in `.devopsy/.env` (gitignored), or in
the environment:

```sh
echo DEVOPSY_SERVER=devopsy@203.0.113.10 >> .devopsy/.env
```

Each environment gets a URL, `<project>-<environment>.<server's base
domain>`, named after its compose project (`whoami-prod`). Each release
imports the base domain from the server's proxy (devopsy-template-traefik)
through the `devopsy.import` label in `compose.yaml`, and fails while no
proxy runs. The release prints what it imported, and `devopsy @prod
--debug imports` shows it later and whether the proxy has changed it
since. Override it per target, or set it empty for no wildcard URL, with
`devopsy @prod --vars set --show PROXY_BASE_DOMAIN`.

```sh
devopsy @prod --release          # runs deploy, prints https://whoami-prod.<server's base domain>
devopsy @<server>-traefik domains whoami-prod   # DNS, challenge and certificate per host, and what next
devopsy @prod logs -f web
devopsy @prod --releases
devopsy @staging --release
devopsy @pr-12 --release         # any pr-* target: one per pull request
devopsy @pr-12 --destroy         # removes it
```

`.devopsy/config.yaml` names the project (`whoami`) and defines the
`prod`, `staging`, `pr-*` and `demo-*` environments. Each has a directory
on the server (`whoami/prod`, under its release root), containers and a
URL. `DEVOPSY_SERVER_STAGING` (or `devopsy
@<server>:staging`) puts staging on another server. Without a base domain
or `PROXY_DOMAINS`, a released environment has no host, and `deploy`
prints that it has no URL.

`pr-*` environments are children of `prod` (`parent: prod`). Their first
release syncs prod's data, and `devopsy @pr-12 --sync` syncs it again.
whoami has no data, so a sync copies nothing and only shows the flow;
devopsy-template-drupal11 syncs files and a database.

The hosts come from `.devopsy/capabilities/env/compute`, which devopsy
runs before every command. It computes `SITE_HOST_RULE` (the router's
rule) and `SITE_URL` from the compose project name, the imported base
domain and `PROXY_DOMAINS`, and `devopsy --env` shows them. Print more
there if your project needs them.

## A custom domain

In `config.yaml`, give the target its domains and release again:

```yaml
environments:
  prod:
    env:
      PROXY_DOMAINS: whoami.example.org
```

The certificate comes from the server's default resolver,
devopsy-template-traefik's `PROXY_CERTRESOLVER`, which is HTTP-01 unless
it was set up otherwise.

With HTTP-01 (`le-http01`), point the domain at the server, with an A
record or a CNAME to the wildcard URL, then release. `devopsy
@<server>-traefik domains whoami-prod` (devopsy-template-traefik) shows
when it is live; if the certificate came too early, add `--retry`.
`devopsy --probe whoami.example.org` checks it from your machine.

To get the certificate before switching DNS, on a server whose default
is HTTP-01, set a DNS-01 resolver for this target and release. With the
server's acme-dns, that is `PROXY_CERTRESOLVER: le-dns01-selfhosted`.
`devopsy @<server>-traefik domains whoami-prod` then prints the
`_acme-challenge` CNAME to create. Once it exists, the same command with
`--retry` gets the certificate while the domain still points elsewhere,
and you can switch DNS. `le-dns01-cloudflare` works the same way,
through the server's Cloudflare token.

`PROXY_DOMAINS` takes host names only, no wildcards (`*.example.org`),
since Traefik's `Host()` matches names literally.

On a server whose sites are all tunneled (devopsy-template-traefik's
[Behind a Cloudflare
Tunnel](https://github.com/hanoii/devopsy-template-traefik/blob/main/docs/tunnel.md)),
nothing changes. On a server with direct sites too, a tunneled site names
the tunnel's entrypoint, `PROXY_ENTRYPOINTS: tunnel` in its target's
`env`, with `PROXY_CERTRESOLVER` unset. TLS ends at Cloudflare's edge, and
a router with a resolver never matches the tunnel's plain HTTP. The site
is then not served on the server's ports 80 and 443.

## Through a stop

The router names `hold@file`, the hold in the server's Traefik ([Stopped
sites](https://github.com/hanoii/devopsy-template-traefik/blob/main/docs/hold.md)).
While the container stops and starts again (a release, `devopsy @prod
restart`), visitors wait instead of getting a 502 or a 404, and are sent
on to the same URL once the site is back. Release the server's Traefik
first, since an older one has no hold. To opt out, drop `hold@file` from
the router's `middlewares` label.

## Basic auth

Set `SITE_BASIC_AUTH=user:password` in `.devopsy/.env`. On a server, use
`devopsy @prod --vars set SITE_BASIC_AUTH`, then `devopsy @prod --reload`
or a release. Traefik then asks for it on every request, through a
`basicauth` middleware on the router.

Traefik takes no plain passwords. `.devopsy/capabilities/env/compute`
hashes the password (apr1, with `openssl`) into `SITE_BASIC_AUTH_USERS`,
and the router names the middleware through `SITE_MIDDLEWARES`. Unset or
empty, the site is open. `pr-*` environments inherit prod's value.

The capability runs wherever devopsy does, so `openssl` must be there too.
macOS ships it, and devopsy-server installs it on servers.

Basic auth guards the router, not the container. Other containers on the
`traefik` network reach it directly (devopsy-template-traefik's [Routing
a
project](https://github.com/hanoii/devopsy-template-traefik#routing-a-project)).
devopsy-template-drupal11 checks it in the application instead, in Caddy.

## Locally

Without a base domain, the host is `whoami.localhost` (the config's
project name), served by a local devopsy-template-traefik:

```sh
devopsy --release
```

## License

MIT, so you can start your own project from it. See [LICENSE](LICENSE).
