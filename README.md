# devopsy-template-whoami

The smallest devopsy project: [traefik/whoami](https://github.com/traefik/whoami)
behind a [devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik) server,
released with [devopsy](https://github.com/hanoii/devopsy-cli). Use it to try
targets, releases, domains and certificates, or as a starting point.

## Using this template

A starting point to own, not a dependency: start a project from it on
GitHub ("Use this template"), or clone it and keep this repository as a
remote (`upstream`) to pull its changes when you choose. Nothing updates
your copy, or what runs on your servers, but your own release.

## Try it

Point it at your server in `.devopsy/.env` (gitignored), or in the
environment:

```sh
echo DEVOPSY_SERVER=devopsy@203.0.113.10 >> .devopsy/.env
```

Each environment also gets `<project>-<environment>.<server's wildcard
domain>` (its compose project name, like `whoami-prod`), whose domain each
release imports from the server's proxy (devopsy-template-traefik), through the
`devopsy.import` label in `compose.yaml`: a release fails while no proxy
runs. The release says what it imported; `devopsy @prod --debug imports`
shows it later, and whether the proxy has changed it since. Override it,
or set it empty for none, per target: `devopsy @prod --vars set --show
DEVOPSY_WILDCARD_DOMAIN`.

```sh
devopsy @prod --release          # runs deploy, prints https://whoami-prod.<server's wildcard domain>
devopsy @<server>-traefik domains whoami-prod   # DNS, challenge and certificate per host, and what next
devopsy @prod logs -f web
devopsy @prod --releases
devopsy @staging --release
devopsy @pr-12 --release         # any pr-* target: one per pull request
devopsy @pr-12 --destroy         # and gone with it
```

`.devopsy/config.yaml` names the project (`whoami`) and defines `prod`,
`staging` and `pr-*` environments, each in its own directory on the server
(`whoami/prod`, under its release root), so its own containers and URL.
`DEVOPSY_SERVER_STAGING` (or `devopsy @<server>:staging`) puts staging on another server. Without a wildcard domain or `DEVOPSY_DOMAINS`, a
released environment has no host at all: `deploy` says so.

`pr-*` environments are children of `prod` (`parent: prod`): their first
release clones prod's data, and `devopsy @pr-12 --clone` clones it again.
whoami has no data, so a clone copies nothing and only shows the flow;
devopsy-template-drupal11 clones files and a database.

The hosts come from `.devopsy/capabilities/env/compute`, which devopsy runs
before every command: `SITE_HOST_RULE` (the router's rule) and `SITE_URL`,
from the compose project name, the imported wildcard domain and
`DEVOPSY_DOMAINS`. `devopsy --env` shows them. Print more there if your
project needs them.

## A custom domain

In `config.yaml`, give the target its domains and release again:

```yaml
environments:
  prod:
    env:
      DEVOPSY_DOMAINS: whoami.example.org
```

- **Default, HTTP-01:** point the domain at the server (an A record, or a
  CNAME to the wildcard URL), then release. `devopsy @<server>-traefik
  domains whoami-prod` (devopsy-template-traefik) shows when it is live; if the
  certificate came too early, `--retry`. `devopsy --probe whoami.example.org`
  checks it from your machine.
- **Before switching DNS, acme-dns:** add `CERTRESOLVER: acmedns`, release,
  and `devopsy @<server>-traefik domains whoami-prod` prints the
  `_acme-challenge` CNAME to create. Once it exists, the same with `--retry`
  gets the certificate while the domain still points elsewhere; then switch
  DNS.

## Locally

Locally, without a wildcard domain, the host is `whoami.localhost` (the
config's project name), served by a local devopsy-template-traefik:

```sh
devopsy up -d
```

## License

MIT, so you can start your own project from it. See [LICENSE](LICENSE).
