# Upgrading Metabase

Metabase runs at `reports.grinta.eu` on the Heroku app **`grinta-metabase`**. The deploy is split
across **two repos** — an upgrade touches both:

| Repo | Role |
|---|---|
| **`metabase-buildpack`** (this repo) | Heroku buildpack. [`bin/version`](bin/version) pins the Metabase version; `bin/compile` downloads that Enterprise JAR at build time. |
| **`metabase-deploy`** | The actual Heroku app pushed to `grinta-metabase`. Holds the `Procfile`, `bin/start` (JVM options + Heroku→Metabase env translation), `system.properties` (Java version), and `app.json` (Postgres addon + buildpack list). |

The live app's buildpacks (`heroku buildpacks -a grinta-metabase`) are `heroku/jvm` +
`https://github.com/grinta/metabase-buildpack` — so the version in **this** repo's `bin/version` is
what actually gets deployed.

**The upgrade in one sentence:** bump `bin/version` here → push a rebuild commit from
`metabase-deploy` to Heroku. Everything else below is the safety around that: picking the right
version, not losing data, and not tripping Heroku's boot timeout. The steps are generic — they work
for any target version.

- **Heroku app:** `grinta-metabase` (both repos have a git remote `production` → `git.heroku.com/grinta-metabase.git`)
- **Edition:** Metabase **Enterprise/Pro** — the `1.x` version line (self-hosted)
- **JAR source:** `https://downloads.metabase.com/enterprise/v<version>/metabase.jar`

---

## Version numbering

| Edition | Format | Example |
|---|---|---|
| Open Source | `0.MAJOR.MINOR` | `0.62.3` |
| Enterprise/Pro (what we run) | `1.MAJOR.MINOR` | `1.62.3` |
| Enterprise hotfix build | `1.MAJOR.MINOR.HOTFIX` | `1.62.3.5` |

OSS `0.62.3` and EE `1.62.3` are the same release. The 4th digit (`.5`) is an Enterprise-only
hotfix build — always prefer the **highest** hotfix of the target release.

---

## Step 1 — Find the latest version

Two questions: what's the newest `MAJOR.MINOR`, and what's the newest EE hotfix build of it.

**a) Newest release (`MAJOR.MINOR`)** — GitHub releases are date-sorted and authoritative:

```sh
gh api "repos/metabase/metabase/releases?per_page=15" \
  --jq '.[] | "\(.tag_name)\t\(.published_at)"'
```

The top `v0.MAJOR.MINOR` is the newest release. (Or read <https://www.metabase.com/releases>.)
The EE equivalent is `1.MAJOR.MINOR`.

**b) Newest EE hotfix build** — probe the download server, incrementing the 4th digit until it
404s. The server 404s correctly on non-existent versions, so a `200` means a real JAR:

```sh
# Sanity-check the server 404s on a bogus version FIRST (guards against false 200s):
curl -s -o /dev/null -I -w "%{http_code}\n" -L \
  https://downloads.metabase.com/enterprise/v1.99.99.99/metabase.jar   # must print 404

# Then walk the hotfix builds of the target release until you hit a 404:
for n in "" .1 .2 .3 .4 .5 .6 .7 .8; do
  v="1.62.3$n"
  code=$(curl -s -o /dev/null -I -w "%{http_code}" -L \
    "https://downloads.metabase.com/enterprise/v$v/metabase.jar")
  echo "v$v -> $code"
done
```

The **highest version that returns `200`** is your target. (Cross-check: file sizes increase
build-over-build; a sudden `404` is the ceiling.)

---

## Step 2 — Can we jump straight there?

Metabase can upgrade **directly to the latest from any version ≥ 40**. Only instances older than
40 must step through releases. We're well past that, so any single jump (e.g. 56 → 62) is fine —
migrations run automatically on first boot.

Before a multi-major jump, skim the release notes for each major you're skipping past
(<https://www.metabase.com/releases>) for admin-facing changes.

---

## Step 3 — Pre-flight checks (no downtime)

```sh
# License is active Pro/Enterprise:
#   Admin → Settings → License and Billing  in the web UI

# Java version — Metabase 62 requires Java 21. Confirm what the app actually runs:
heroku run "java -version" -a grinta-metabase
```

**Java is pinned in `metabase-deploy/system.properties`** (`java.runtime.version=21`), provisioned
by the `heroku/jvm` buildpack. It's already on 21, which 62 needs. If a *future* Metabase version
requires a newer JDK, bump it there and deploy that repo (Step 6).

```sh
# Confirm the dyno size (see "Sizing / memory" below) and buildpacks:
heroku ps -a grinta-metabase
heroku buildpacks -a grinta-metabase
```

---

## Step 4 — Back up the application database (mandatory)

Downgrading after a major upgrade is **not supported** — the schema migrates forward and the old
JAR can't read it. This backup is the only rollback path.

```sh
heroku pg:backups:capture -a grinta-metabase
heroku pg:backups -a grinta-metabase        # confirm the new backup shows "Completed"
```

Do **not** rotate the embedding secret or `MB_ENCRYPTION_SECRET_KEY` — they live in the app DB and
are preserved by a same-DB upgrade. That's what keeps the partners-portal embeds valid (see Step 8).

---

## Step 5 — Bump the version (metabase-buildpack)

```sh
cd ~/code/metabase-buildpack
echo "1.62.3.5" > bin/version          # ← your Step 1 target
git commit -am "bump metabase to 1.62.3.5"
git push origin master                  # Heroku fetches the buildpack from GitHub at build time
```

Nothing is deployed yet — this only updates the pinned version. The rebuild happens in Step 6.

---

## Step 6 — Deploy + migrate (metabase-deploy)

The rebuild is triggered by pushing **`metabase-deploy`** to Heroku. Because the buildpack is
fetched fresh from GitHub on every build, an **empty commit** is enough — Heroku re-runs
`bin/compile`, which downloads the new JAR pinned in Step 5. (Our upgrade commits are empty by
convention: `Upgrade to <version>`.)

**Run migrations on a one-off dyno to avoid the R10 boot timeout.** A web dyno must bind `$PORT`
within **60 seconds** or Heroku kills it (error R10). Migrating across several majors can take
longer, causing a crash loop. Detach the migration from the web dyno:

```sh
cd ~/code/metabase-deploy
heroku maintenance:on -a grinta-metabase
heroku ps:scale web=0 -a grinta-metabase        # note the current count first: `heroku ps`

# Trigger the rebuild (downloads the new JAR) without starting the web dyno:
git commit --allow-empty -m "Upgrade to 1.62.3.5"
git push production master --force              # see "Heroku git repo has diverged" below
```

**The Heroku git repo has diverged from GitHub.** `production/master` still points at an abandoned
`0.53.x` lineage (there are also stale `maste` and `upgrade` branches). GitHub is the source of
truth — the Heroku repo is only a build input, so `--force` is expected here. Before forcing, sanity
-check that you're not changing the runtime config out from under the app:

```sh
git fetch production
git diff --stat production/master origin/master   # expect only README/docs — NOT Procfile,
                                                  # bin/start or system.properties
```

Then let the **web dyno** run the migrations at boot (Step 7) while maintenance is still on. A boot
under maintenance is contained: the router serves the maintenance page regardless of dyno state, so
even an `R10 Boot timeout` crash-loop is invisible to users, and you can watch Liquibase in the logs.

> **Don't use `heroku run … migrate up` blindly** — as written it fails *silently* two ways:
> `heroku run` doesn't source `bin/start`, so `DATABASE_URL` is never translated to
> `MB_DB_CONNECTION_URI` and Metabase falls back to a **throwaway H2 file** (look for `WARNING:
> Using Metabase with an H2 application database`); and one-off dynos default to **standard-1x
> (512 MB)** while `JAVA_TOOL_OPTIONS=-Xmx2g` asks for a 2 GB heap, so the dyno is killed by
> `R15 (Memory quota vastly exceeded)` — and the CLI still exits **0**. If you do need it detached
> from the web dyno (multi-major jump, long migration), run it like this:
>
> ```sh
> heroku run --size performance-m \
>   'MB_DB_CONNECTION_URI="$DATABASE_URL?ssl=true&sslmode=require&sslfactory=org.postgresql.ssl.NonValidatingFactory" \
>    java -jar target/uberjar/metabase.jar migrate up' -a grinta-metabase
> ```
>
> Never trust the exit code alone — verify against the DB (Step 7).

---

## Step 7 — Bring the web dyno back up

Boot **one** dyno while maintenance is still on, and watch it migrate:

```sh
heroku ps:scale web=1 -a grinta-metabase
heroku logs -n 400 -a grinta-metabase | grep -iE "migrat|Initialization COMPLETE|R1[045]"
```

Verify the migrations actually landed — the app DB is the only source of truth here (replace `v63`
with the major you're upgrading to; `dateexecuted` must be today):

```sh
heroku pg:psql -a grinta-metabase -c \
  "select substring(id from 1 for 4) maj, count(*), max(dateexecuted) last
     from databasechangelog group by 1 order by last desc limit 4;"
```

Then restore the **pre-upgrade dyno count** (it is `web=2`, not 1 — check Step 3's `heroku ps`
output) and reopen:

```sh
heroku ps:scale web=2 -a grinta-metabase
heroku maintenance:off -a grinta-metabase
curl -s -o /dev/null -w "%{http_code}\n" https://reports.grinta.eu/api/health   # expect 200
```

Then load `https://reports.grinta.eu`, log in, and open a saved dashboard + a native question to
confirm data sources survived. For reference, a 62 → 63 boot took **18 s** (JVM uptime 51 s) with
66 Liquibase changesets, no `R14`.

---

## Step 8 — Verify embedding (critical — this is how the Grinta app uses Metabase)

The Grinta partners portal embeds dashboards via signed JWTs (`Partners::MetabaseEmbeddable`,
signed with `METABASE_SECRET_KEY`). Static/signed embedding is stable across upgrades, but always
verify:

1. The embedding secret still matches Grinta's `METABASE_SECRET_KEY` — compare fingerprints without
   printing either secret:

   ```sh
   heroku pg:psql -a grinta-metabase -c \
     "select md5(value) from setting where key = 'embedding-secret-key';"
   grep '^METABASE_SECRET_KEY=' ~/code/grinta/.env | cut -d= -f2 | tr -d '\n' | md5
   ```

2. The dashboards the app actually embeds still render. Their ids live in
   `grinta/config/metabase_dashboards.yml`; each locked param in `report_dashboard.embedding_params`
   must be present in the signed token, then `GET /api/embed/dashboard/<jwt>` should return **200**
   with a non-empty `dashcards`. A 401 means the embedding secret changed — restore it in Metabase to
   match Grinta's config var, **don't change the Grinta side**. A 400 usually means a locked param is
   missing from the token, not a broken upgrade.

3. Finally, load Orders, Revenues, and Inventories in the **partners portal** for a real retailer.
   (`heroku logs -a grinta-metabase | grep /api/embed/dashboard` should show `202 [ASYNC:
   completed]` in a few hundred ms.)

---

## Step 9 — Rollback (only if Steps 7–8 fail)

```sh
# 1. Revert the version pin:
cd ~/code/metabase-buildpack
echo "<previous-version>" > bin/version      # e.g. 1.56.25
git commit -am "rollback metabase to <previous-version>"
git push origin master

# 2. Redeploy the old JAR + restore the DB (the schema was migrated forward — restore is REQUIRED):
cd ~/code/metabase-deploy
heroku ps:scale web=0 -a grinta-metabase
git commit --allow-empty -m "Rollback to <previous-version>"
git push production master --force
heroku pg:backups -a grinta-metabase                       # find the backup id (e.g. b001)
heroku pg:backups:restore <id> DATABASE_URL -a grinta-metabase
heroku ps:scale web=2 -a grinta-metabase                   # pre-upgrade count
heroku maintenance:off -a grinta-metabase
```

Then re-verify embeds (Step 8).

---

## Sizing / memory (R14)

Newer majors have a larger JVM footprint. Metabase **62 needs a ≥ 2.5 GB dyno**; the old 1 GB
Standard-2X dyno hit `Error R14 (Memory quota exceeded)`, swapped, and made embedded dashboards
crawl. Current sizing:

- Dyno: **Performance-M** (2.5 GB) — `heroku ps:resize web=performance-m -a grinta-metabase`
- Heap cap: **`JAVA_TOOL_OPTIONS=-Xmx2g`** config var, leaving ~500 MB for JVM off-heap / metaspace
  / threads. Without an explicit cap the JVM sizes its heap off the host and can overrun the quota.
  (Base JVM flags — G1GC, container support, etc. — live in `metabase-deploy/bin/start`.)

After an upgrade, watch for R14 and bump the dyno / heap if it appears:

```sh
heroku logs -n 1500 -a grinta-metabase | grep -iE "R14|Memory quota"
```

---

## Quick checklist

```
[ ] Step 1  Find latest version (gh releases + download-server probe)
[ ] Step 2  Confirm direct jump is allowed (≥ v40) + skim release notes
[ ] Step 3  Pre-flight: license active, `java -version` is 21+, dyno ≥ 2.5 GB, note dyno COUNT
[ ] Step 4  heroku pg:backups:capture   ← mandatory
[ ] Step 5  metabase-buildpack: echo <version> > bin/version; commit; push origin
[ ] Step 6  metabase-deploy: maintenance on; web=0; empty commit; push production --force
[ ] Step 7  web=1 (still in maintenance) → check databasechangelog → web=2; maintenance off
[ ] Step 8  verify embed secret md5 + /api/embed/dashboard 200 + partners portal
[ ] Step 9  (only on failure) revert version + redeploy + restore DB backup
```
