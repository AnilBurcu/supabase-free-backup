# supabase-free-backup

Your Supabase free-tier project has zero backups. This repo gives it a daily
one with a single GitHub Actions file: roles, schema and data, dumped with
the official Supabase CLI, compressed, and stored as an auto-expiring
workflow artifact.

No server, nothing to install locally, nothing to maintain. Set it up once
and download a backup whenever you need one.

> [!WARNING]
> Backups contain your entire database, and the connection string you will
> configure contains your database password. Three rules follow: run this
> only in a **private** repository (the workflow refuses to run in public
> ones), keep the connection string only in an Actions **secret**, and
> never paste it into the workflow file or anywhere else in your copy.

## Why artifacts and not commits

Most backup tutorials commit the dump into the repo. That works until you
look at what git does with it: deleting yesterday's dump from the working
tree does not delete it from history, so the repo grows with every single
backup, forever, and there is no simple way back.

Workflow artifacts avoid the whole problem. Each backup expires on its own
after `RETENTION_DAYS`, the repo stays at a few kilobytes, and old backups
disappear without you thinking about them. The tradeoff is honest: this
gives you a rolling window of recent backups, not an archive since the
beginning of time. For a free-tier safety net, the rolling window is the
right shape.

## Setup

About five minutes per project.

### 1. Create a private copy

Click "Use this template" at the top of this page, then "Create a new
repository". Set it to **Private**. This is not a style preference: the
workflow refuses to run in public repositories, because backups contain
your actual database contents.

### 2. Get the connection string

In your Supabase dashboard, click **Connect** at the top and copy the
**Session pooler** connection string. It looks like:

```
postgresql://postgres.abcdefghijklmnop:[YOUR-PASSWORD]@aws-1-eu-central-1.pooler.supabase.com:5432/postgres
```

Replace `[YOUR-PASSWORD]` with your database password (Settings > Database
if you need to reset it). If the password contains characters like `@`,
`#` or `/`, percent-encode them in the URL, or reset to a generated
password and save yourself the trouble.

Two details matter here:

- It must be the **Session** pooler (port 5432), not the Transaction pooler
  (port 6543). Dumps need a session-mode connection.
- The direct connection string (`db.<ref>.supabase.co`) will not work from
  GitHub Actions: direct connections are IPv6 and the runners only have
  IPv4. The pooler is reachable over IPv4, which is why this repo uses it.

### 3. Add it as a secret

In your repo: Settings > Secrets and variables > Actions > New repository
secret. Name it `SUPABASE_DB_URL_PROJECT_1` (or whatever you set in the
workflow matrix), paste the full connection string as the value.

This string contains your database password. That is exactly why the
public-repo guard exists and why it belongs in a secret, never in the
workflow file.

### 4. More than one project

Edit the matrix in `.github/workflows/backup.yml`, one block per project:

```yaml
- name: my-app
  url_secret: SUPABASE_DB_URL_MY_APP
```

Add one secret per project with the matching name.

### 5. Run it once by hand

Actions tab > "Supabase Backup" > Run workflow. When the run is green, open
it and you will find the backup under Artifacts. That download is your
proof the whole chain works; do it once before trusting the schedule.

From then on it runs daily on its own. If a run fails, GitHub emails you.

## Restoring

Download the artifact, extract it, and restore into a fresh Supabase
project using its own Session pooler string. This is the exact restore
sequence Supabase documents, as one command:

```bash
tar -xzf my-app-2026-07-28.tar.gz
psql \
  --single-transaction \
  --variable ON_ERROR_STOP=1 \
  --file roles.sql \
  --file schema.sql \
  --command 'SET session_replication_role = replica' \
  --file data.sql \
  --dbname "$NEW_DB_URL"
```

The `session_replication_role = replica` line between schema and data is
not decoration: it turns off triggers and foreign key enforcement while
the data loads, so rows can arrive in any order. Without it, restores die
on foreign key ordering.

Notes from doing this for real:

- Restore into a fresh project, not on top of a live one.
- Managed roles that already exist in every Supabase project can produce
  "already exists" notices. Notices are fine; errors that stop the
  transaction are not, and `ON_ERROR_STOP` makes the difference visible.
- Test a restore once before you need it. A backup you have never restored
  is a hope, not a backup.

## Storage math

Free-tier Supabase databases are capped at 500 MB, and dumps compress
well, so a typical project produces artifacts in the single-digit MB
range. GitHub's Actions storage for private repos on the free plan is
500 MB total: seven days of retention fits comfortably for most projects.
If your dumps are large, lower `RETENTION_DAYS` or reduce the matrix
before raising anything else.

## FAQ

**Does this also stop my free project from being paused?**
Yes, as a side effect. Supabase pauses free projects after about a week
without database activity, and a daily dump is database activity. If you
only want the pause protection without backups, the lighter tool is
[keep-supabase-alive](https://github.com/AnilBurcu/keep-supabase-alive).

**Why not upgrade to a paid plan instead?**
If the project earns money, do: paid plans come with real backups and
point-in-time recovery, and this repo is not a substitute for that. This
is for the free-tier projects that are not there yet but whose data you
would still hate to lose.

**Is the database password safe in GitHub secrets?**
Actions secrets are encrypted, hidden from logs, and unreadable after
saving. The realistic risk is not the secret store; it is accidentally
putting the connection string in a file. Keep it in the secret, keep the
repo private, and the workflow enforces the second part.

**What if my repository becomes public later?**
The guard runs on every backup, not only during setup. The moment the
repository is public, the next run refuses and fails loudly instead of
uploading your data where anyone could reach it.

**What exactly is in the backup?**
Three files produced by `supabase db dump`: `roles.sql` (custom roles),
`schema.sql` (structure), and `data.sql` (contents, as COPY statements).
Storage buckets' files and edge function source are not in the database
and therefore not in the dump; keep those in your project repo where they
belong.

## License

[MIT](LICENSE)
