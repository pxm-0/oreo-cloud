# References

Implementation assumptions should be checked against current official docs during implementation.

## Tailscale

- Tailscale Serve docs: https://tailscale.com/docs/features/tailscale-serve
- Tailscale Funnel docs: https://tailscale.com/kb/1223/funnel

Important P0 assumption:

- Serve is private to the tailnet.
- Funnel is public.
- P0 must not enable Funnel.

## Caddy

- Caddy `bind` directive: https://caddyserver.com/docs/caddyfile/directives/bind

Important P0 assumption:

- `bind` controls which interface the listener uses.
- The site address controls the port.

## Docker Compose

- Compose project names: https://docs.docker.com/compose/how-tos/project-name/
- Docker Compose CLI reference: https://docs.docker.com/reference/cli/docker/compose/

Important P0 assumption:

- Moving Compose files can change default project naming.
- Preserve project names with `-p` or `COMPOSE_PROJECT_NAME`.

## Cloudflare Tunnel

- Cloudflare Tunnel setup: https://developers.cloudflare.com/tunnel/setup/
- Cloudflare Tunnel local configuration: https://developers.cloudflare.com/tunnel/advanced/local-management/configuration-file/
- Quick Tunnels: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/

Important P0 assumption:

- Quick Tunnels are for testing/development.
- Named tunnels are the better future shape for stable workload hostnames.
- P0 generates Cloudflare plans only and does not start a tunnel.

## Git

- `git init`: https://git-scm.com/docs/git-init
- `.gitignore`: https://git-scm.com/docs/gitignore

Important P0 assumption:

- Git tracks the platform, not secrets or workload source by default.
