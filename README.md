# Comper Private Beta

Hey there, welcome to the Comper private beta for developers.

You’ll get an access key from Jouke, which is a docker hub token.

You can deploy using Docker Compose or Kubernetes, we're currently working on a Helm chart, so ask us about it.

Make sure you have docker (with docker compose support) on your system.

Then run:

`docker login -u comperio`

And use the token provided to you.

Then use the following `docker-compose.yml` file (set `JWT_SECRET` to a long random string before you start):

## Docker Compose File

```yaml
volumes:
  postgres:
  osv_scanner_cache:

services:
  postgres:
    image: postgres:17
    ports:
      - 5432:5432
    volumes:
      - postgres:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=loco
      - POSTGRES_USER=loco
      - POSTGRES_DB=comper
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U loco -d comper"]
      interval: 5s
      timeout: 5s
      retries: 5

  osv-scanner:
    image: comperio/osv-scanner-server:latest
    platform: linux/amd64
    command:
      [
        "scan",
        "server",
        "--listen",
        "0.0.0.0:8002",
        "--offline",
        "--download-offline-databases",
      ]
    ports:
      - 8002:8002
    environment:
      - OSV_SCANNER_LOCAL_DB_CACHE_DIRECTORY=/osv-cache
    volumes:
      - $HOME/tmp/comper:/comper/storage
      - /tmp:/tmp
      - osv_scanner_cache:/osv-cache

  app:
    image: comperio/comper:latest
    ports:
      - 8001:8001
    depends_on:
      postgres:
        condition: service_healthy
      osv-scanner:
        condition: service_started
    environment:
      - DATABASE_URL=postgres://loco:loco@postgres:5432/comper
      - JWT_SECRET=change-me-to-a-long-random-secret
      - WORKERS=1
      - FRONTEND_URL=http://localhost:8001
      # Required for email/password sign-up in the private beta (see below)
      - PASSWORDS_ENABLED=true
      - ALLOW_UNINVITED_SIGNUP_VIA_EMAIL=true
      - SMTP_HOST=smtp.example.com
      - SMTP_PORT=587
      - SMTP_SECURE=true
      - SMTP_USER=your-smtp-user
      - SMTP_PASSWORD=your-smtp-password
    volumes:
      - $HOME/tmp/comper:/comper/storage
      - $HOME/your-local-repos:/comper/repos
```

Create the storage directory before starting: `mkdir -p $HOME/tmp/comper`. For local disk sources, clone repos into `$HOME/your-local-repos` (mounted at `/comper/repos` inside the container — must not overlap with `/comper/storage`).

The app listens on port **8001** (metrics on **9464** inside the container). OSV scanner defaults to `http://osv-scanner:8002` via `OSV_SCANNER_URL`.

## App environment variables

The image reads settings from `config/production.yaml`. Anything under `settings:` or used for auth/mail can be overridden with environment variables on the `app` service. Common ones for this beta:

| Variable | Default in image | Recommended for beta |
| --- | --- | --- |
| `JWT_SECRET` | (required) | Long random string |
| `FRONTEND_URL` | (required) | `http://localhost:8001` (or your public URL) |
| `WORKERS` | (required) | `1` (increase for larger teams) |
| `DATABASE_URL` | (required) | `postgres://loco:loco@postgres:5432/comper` |
| `PASSWORDS_ENABLED` | `false` | `true` |
| `ALLOW_UNINVITED_SIGNUP_VIA_EMAIL` | `false` | `true` |
| `SMTP_HOST` | (unset — mailer off) | Your SMTP server hostname |
| `SMTP_PORT` | `1025` | Usually `587` with `SMTP_SECURE=true` |
| `SMTP_SECURE` | `false` | `true` for STARTTLS (see below) |
| `SMTP_USER` / `SMTP_PASSWORD` | (unset) | Both required when your provider uses auth |
| `SMTP_FROM` | `Comper <mail@comper.io>` | e.g. `Comper <noreply@yourcompany.com>` |
| `OSV_SCANNER_URL` | `http://osv-scanner:8002` | leave default |
| `STORAGE_PATH` | `/comper/storage` | leave default (matches volume mount) |
| `DEPLOYMENT_TIER` | `enterprise` | leave default |
| `REMEMBER_ME_DEFAULT` | `true` | leave default |

### SMTP (sign-up and password reset)

Set `SMTP_HOST` to turn the mailer on. Without it, email/password sign-up will not work.

`SMTP_SECURE` controls how Comper connects:

| `SMTP_SECURE` | Connection | Typical port | Use when |
| --- | --- | --- | --- |
| `true` | STARTTLS (upgrade plain connection to TLS) | `587` | Most providers (SendGrid, Mailgun, Postmark, Amazon SES, Microsoft 365, Google Workspace, etc.) |
| `false` | Plain SMTP, no TLS | `25` or `1025` | Local/dev only (e.g. Mailhog in your own compose stack) |

**STARTTLS on 587 (recommended):**

```yaml
      - SMTP_HOST=smtp.example.com
      - SMTP_PORT=587
      - SMTP_SECURE=true
      - SMTP_USER=your-smtp-user
      - SMTP_PASSWORD=your-smtp-password
      - SMTP_FROM=Comper <noreply@yourcompany.com>
```

**Not supported:** implicit TLS on port **465** (SMTPS). If your provider only offers 465, use their STARTTLS endpoint on 587 instead, or put an SMTP relay in front that accepts STARTTLS.

**Auth:** `SMTP_USER` and `SMTP_PASSWORD` must both be set; if either is missing, Comper connects without SMTP authentication.

**Sender address:** `SMTP_FROM` must be an address your provider allows you to send as (verified domain, mailbox, or “send as” permission).

OAuth (Google, GitHub, Microsoft, GitLab) and SAML are configured via additional env vars when you set the corresponding `*_OAUTH_CLIENT_ID` / `SAML_*` values; see `production.yaml` in the main Comper repo for the full list.

## System requirements

### Database

The app uses up to **50** database connections by default (`DB_MAX_CONNECTIONS`). Size Postgres to 1GB RAM + 5GB disk and expand for larger teams.

### Disk

Comper will clone all your repos and maintain a lot of metadata, so for the non-database part, expect to use your git repo size * 2. We suggest starting with 10GB and expanding when needed.

### CPU

During the initial sync, we use a LOT of CPU resources to analyse all the git repos. The more CPU you throw at it, the faster it gets. After we're done analyzing the git repos, we use next to nothing.


## Starting up

Run `docker compose up -d`

## Repo analysis

You can sync with your Github, Gitlab, Azure DevOps or Bitbucket account. For Bitbucket, we suggest going with personal access tokens, as the other solutions require premium plans. Alternatively, you can use local disk if you just want to look at locally checked out repos.

If you use locally cloned repos, clone them into `$HOME/your-local-repos` (or whatever host path you mount at `/comper/repos`).

## Create an account

Open http://localhost:8001 and sign up with email and password. Set `PASSWORDS_ENABLED`, `ALLOW_UNINVITED_SIGNUP_VIA_EMAIL`, and your SMTP settings as above so verification emails can be delivered.

## Create a board

Create a board, then navigate to settings on the left, go to sources and configure your source. For local dirs: use a path under `/comper/repos` (e.g. `/comper/repos/my-project`). Comper will now start analyzing the repos in four steps for each:

1. fetch
2. shallow inspection, where we just look at what is in the HEAD commit
3. history parsing, where we look at contribution stats over time
4. deep inspection, where we get a lot of authorship stats (think massive `git blame` stats)

After step 2, we already show something on the board, and you can use "Auto Layout" and save to put things in a nice place. However, statistics won't work yet and authorship stats also won't work.

In these steps you can expect a LOT of CPU, disk and network traffic.

In the bottom left you can see that progress and queue of tasks. If the number says "0", we are completely done.

Our feedback in the UI when there are errors isn't great yet, so keep an eye on the logs of the application to see if there are any errors.

## Deduplicate contributors

When the analysis is underway, go to settings again, then to Contributors. The idea is to create "Contributors" from "Aliases". Each contributor (an actual person) might have multiple git aliases.

Under "Active Duty" you can mark users as still being with the company or already left.

Once you've done this, the bus factor calculation starts making sense.
