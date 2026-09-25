# mu-online-server

A ready-to-run [OpenMU](https://github.com/MUnique/OpenMU) private server setup,
using the project's official "all-in-one" Docker deployment: connect server,
game servers, login/chat server and the web admin panel all run in one
container, next to PostgreSQL and an nginx reverse proxy.

OpenMU is an open-source, MIT-licensed re-implementation of a MU Online server.
It does **not** include the game client — you need your own legally obtained
MU Online client to actually play.

## Requirements

- Docker + Docker Compose (Docker Desktop on Windows/Mac, or `docker` +
  `docker compose` on Linux)
- A MU Online game client (not provided here — see
  [Connecting a game client](#connecting-a-game-client))

## Quick start (local play/testing)

```bash
git clone https://github.com/sadikush/mu-online-server.git
cd mu-online-server
cp .env.example .env
# edit .env: set OPENMU_ADMIN_USER / OPENMU_ADMIN_PASSWORD to something
# only you know (this is your admin panel login)
docker compose up -d
```

That pulls the official `munique/openmu` image (no build needed) and starts:

- **Admin panel**: http://localhost/
- **Connect server**: `127.0.0.1:44405` (original client) / `:44406` (the
  [open source client](https://github.com/sven-n/MuMain))
- **PostgreSQL**: internal only, data persisted in the `dbdata` volume

First boot takes a minute while Postgres initializes and OpenMU applies
migrations.

## First-time setup

1. Open http://localhost/ and log in with the `OPENMU_ADMIN_USER` /
   `OPENMU_ADMIN_PASSWORD` you set in `.env` (this is a "bootstrap" account —
   see below).
2. If the panel shows no initialized game data, go to **Setup**, pick a game
   version (**Season 6 Episode 3** is the actively maintained one), choose how
   many game servers you want (1 is fine to start), and optionally enable
   **test accounts** for quick testing (see [Test accounts](#test-accounts)
   below — never enable this on a server real players can reach). Click
   **Install** and wait for it to finish.
3. Go to **Users** and create a real administrator account, then stop using
   the bootstrap one for daily use.

### About the bootstrap admin account

`OPENMU_ADMIN_USER`/`OPENMU_ADMIN_PASSWORD` in `.env` create a login that
works before any database/user exists. It's only kept in memory — changes to
it (password, 2FA) don't survive a restart. Use it once to create a real user
on the Users page, then work with that account instead.

If you don't set these two variables, the admin panel is reachable **without
any login** until the first real user is created — fine for a quick local
test, risky the moment the port is reachable by anyone else.

### Test accounts

If enabled during Setup, `test0`…`test9` (and more on Season 6 — `test300`,
`test400`, `testgm`, `testgm2`, `testunlock`, `quest1`-`quest3`, `ancient`,
`socket`; see
[OpenMU's test accounts docs](https://github.com/MUnique/OpenMU/blob/master/docs-website/docs/getting-started/test-accounts.md))
are created with the password equal to the username, some with game-master
rights. Delete or disable them (in **Accounts**) before exposing the server
publicly.

## Connecting a game client

OpenMU doesn't ship a client. Get a MU Online client yourself (original client
or the [open source client](https://github.com/sven-n/MuMain)), then point it
at your server:

- Easiest: use the official
  [MUnique.OpenMU.ClientLauncher](https://github.com/MUnique/OpenMU/releases)
  (requires the [.NET 10 runtime](https://dotnet.microsoft.com/download/dotnet/10.0)).
  Enter the server IP and port and select your client's `main.exe`.
- Server + client on the **same machine**: connect to `127.127.127.127`
  (not `127.0.0.1` — the client blocks that one), port `44405` or `44406`.
- Server + client on **different machines on your LAN**: use the host's LAN
  IP and make sure the ports below are reachable.
- If the client disconnects right after picking a server in the server-select
  screen, the server's **IP resolver** (admin panel → Configuration → System)
  doesn't match how you're connecting — pick `Loopback` for same-machine,
  `Local` for LAN, `Public`/`Custom` for internet-facing. See OpenMU's
  [game client docs](https://github.com/MUnique/OpenMU/blob/master/docs-website/docs/getting-started/game-client.md)
  for details.

### Troubleshooting: game server won't start listening

By default the server auto-detects its public IP by calling
[ipify.org](https://www.ipify.org/) on startup. If that host is unreachable
(offline machine, restrictive firewall/proxy, or a network that intercepts
HTTPS with an untrusted certificate), the game server listeners fail to start
with an SSL/HTTP error while the connect server still starts fine. Fix it by
setting the resolver explicitly, e.g. for local testing:

```bash
# in .env
RESOLVE_IP=loopback
```

or any other value from the table above, then `docker compose up -d` again.
(This was confirmed while validating this setup in a sandboxed environment
whose outbound HTTPS is restricted — it won't normally happen on a regular
machine, but the fix is the same either way.)

## Ports

| Port | Purpose |
|---|---|
| 80 | Admin panel / web (nginx) |
| 44405 | Connect server (original client) |
| 44406 | Connect server (open source client) |
| 55901–55906 | Game servers 1–6 (only as many as you configured in Setup) |
| 55980 | Chat server (in-game messenger) |

## Exposing the server to the internet

Don't skip HTTPS if you do this — the admin panel session/password travel in
the clear otherwise.

```bash
export DOMAIN_NAME=your-domain.example.org
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker compose -f docker-compose.yml -f docker-compose.prod.yml \
  run --rm certbot certonly --webroot --webroot-path /var/www/certbot/ -d "$DOMAIN_NAME"
```

Then set up certificate renewal (certs expire after 3 months) via a cron job
running:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml run --rm certbot renew
```

Before opening this up to real players:

- Create a real admin user and stop relying on the bootstrap credentials.
- Delete/disable test accounts.
- Set the correct IP resolver (`Public`, or `Custom` with your domain/IP).
- Open the game ports (44405/44406, 55901+, 55980) and 443 on your
  firewall/router; you generally don't need 80 open once HTTPS works, only
  443 and Let's Encrypt's renewal path.

## Operating the server

- Logs: `docker compose logs -f openmu-startup`
- Stop: `docker compose down` (data persists in the `dbdata` and
  `adminpanel-keys` volumes)
- Full reset (⚠️ deletes all accounts/characters): `docker compose down -v`
- Update to the latest OpenMU image: `docker compose pull && docker compose up -d`

## Credits

Server software: [MUnique/OpenMU](https://github.com/MUnique/OpenMU) (MIT
license). This repo just wraps its official `deploy/all-in-one` Docker Compose
setup with a couple of README notes. See the
[full OpenMU documentation](https://github.com/MUnique/OpenMU/tree/master/docs-website/docs)
for the admin panel, distributed deployment, and plugin system.
