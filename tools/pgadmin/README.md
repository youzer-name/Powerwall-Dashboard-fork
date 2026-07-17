# pgAdmin for TimescaleDB

An optional [pgAdmin](https://www.pgadmin.org/) service you can add to the
Powerwall-Dashboard stack for browsing/querying the TimescaleDB datastore
directly -- inspecting tables, running ad hoc SQL, checking compression/chunk
status, etc. This is not wired into `setup.sh`; it's an opt-in add-on for
advanced users via the stack's existing `powerwall.extend.yml` mechanism (the
same one [`tools/tesla-history`](../tesla-history/) uses).

It only makes sense if you're running the TimescaleDB datastore -- see the
main [TimescaleDB README](../../timescaledb/README.md) if you haven't set
that up yet.

## Setup

1. Copy the sample extend file to the repo root:
   ```bash
   cp tools/pgadmin/powerwall.extend.yml.sample powerwall.extend.yml
   ```
   If you already have a `powerwall.extend.yml` (e.g. for `tesla-history`),
   merge the `pgadmin` service and `pgadmin-data` volume into it by hand
   instead of overwriting the file -- `docker compose` only reads one
   `powerwall.extend.yml`.
2. Edit the `PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD` values in
   `powerwall.extend.yml` -- this is pgAdmin's own login (any email-shaped
   string works, it doesn't need to be real), not the database password.
3. Start it:
   ```bash
   ./compose-dash.sh up -d
   ```
4. Browse to `http://<this host>:8433` and log in with the email/password
   from step 2.
5. A "TimescaleDB (Powerwall Dashboard)" server is pre-populated (see
   `servers.json`) under Servers -- expand it and you'll be prompted once
   for its password, which is `POSTGRES_PASSWORD` from `timescaledb.env` at
   the repo root. pgAdmin will offer to save it for future sessions.

`powerwall.extend.yml` is gitignored, so your copy (and the credentials in
it) survive `git pull` and `setup.sh` re-runs.

## If you customized TimescaleDB's user/database name

`servers.json` assumes the defaults from `timescaledb.env.sample`
(`POSTGRES_USER=telegraf_powerwall`, `POSTGRES_DB=powerwall`). If you changed
either in your own `timescaledb.env`, edit `tools/pgadmin/servers.json`'s
`Username`/`MaintenanceDB` to match before starting pgAdmin -- it's only read
on pgAdmin's *first* launch (when its internal config database is created),
so edit it before step 3, or delete the `pgadmin-data` volume to re-trigger
the import. `Host`/`Port` (`timescaledb`/`5432`) are the internal Docker
network address and don't need to change even if you've overridden
`TIMESCALEDB_PORTS` for external access (see below) -- that only affects the
host-side port mapping, not container-to-container traffic.

## Changing the port, or opening it up beyond localhost

The sample publishes pgAdmin on `127.0.0.1:8433` (localhost only). To use a
different port or expose it to your LAN, edit the `ports:` line in your
`powerwall.extend.yml` directly, e.g.:
```yaml
        ports:
            - "0.0.0.0:8433:80"
```
Anyone who can reach that port can attempt to log in to pgAdmin, so treat
opening it up the same way you'd treat opening any other admin UI -- keep
the `PGADMIN_DEFAULT_PASSWORD` non-default first.

Note this is a separate mechanism from `TIMESCALEDB_PORTS` in `compose.env`,
which controls whether TimescaleDB itself (not pgAdmin) is reachable
directly from other machines. `docker compose` merges the `ports:` list
across `powerwall.yml` and `powerwall.extend.yml` by concatenating entries,
not replacing them, so `powerwall.extend.yml` can't be used to override
TimescaleDB's existing port mapping -- use `TIMESCALEDB_PORTS` in
`compose.env` for that instead (see the "Local Settings" comments there).

## Removing pgAdmin

```bash
./compose-dash.sh down pgadmin
rm powerwall.extend.yml   # or remove just the pgadmin service/volume from it
docker volume ls | grep pgadmin-data   # find the project-prefixed volume name
docker volume rm <name from above>     # only if you want pgAdmin's saved
                                        # passwords/settings gone too
```
