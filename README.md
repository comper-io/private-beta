# Comper Private Beta

Hey there, welcome to the Comper private beta for developers.

You’ll get an access key from Jouke, which is a docker hub token.

You can deploy using Docker Compose or Kubernetes, we're currently working on a Helm chart, so ask us about it.

Make sure you have docker (with docker compose support) on your system.

Then run:

`docker login -u comperio`

And use the token provided to you.

Then use the following docker-compose.yml file:


## Docker Compose File


```
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
      - /home/<XXXXX>/tmp/comper:/comper/storage
      - /tmp:/tmp
      - osv_scanner_cache:/osv-cache

  mailhog:
    image: mailhog/mailhog
    platform: linux/amd64
    ports:
      - 1025:1025
      - 8025:8025

  app:
    image: comperio/comper:latest
    ports:
      - 8001:8001
    depends_on:
      postgres:
        condition: service_healthy
      mailhog:
        condition: service_started
      osv-scanner:
        condition: service_started
    environment:
      - DATABASE_URL=postgres://loco:loco@postgres:5432/comper
      - JWT_SECRET=ZWVuZzh0ZWl0MnpvaDRXYTgK
      - WORKERS=2
      - FRONTEND_URL=http://localhost:8001
      - PASSWORDS_ENABLED=true
      - ALLOW_UNINVITED_SIGNUP_VIA_EMAIL=true
    volumes:
      - /home/<XXXXX>/tmp/comper:/comper/storage
      - /home/<XXXXX>/your-local-repos:/comper/repos
```

Replace the XXX placeholders based on your system. If you're paranoid and change the JWT secret, change it to something 

## System requirements

### Database

We use Postgres and use max 50 connections by default, make sure your DB can handle that. Size it to 1GB RAM + 5GB disk, and expand for larger teams.

### Disk

Comper will clone all your repos and maintain a lot of metadata, so for the non-database part, expect to use your git repo size * 2. We suggest starting with 10GB and expanding when needed.

### CPU

During the initial sync, we use a LOT of CPU resources to analyse all the git repos. The more CPU you throw at it, the faster it gets. After we're done analyzing the git repos, we use next to nothing.


## Starting up

Run `docker compose up -d`

## Repo analysis

You can sync with your Github, Gitlab, Azure DevOps or Bitbucket account. For Bitbucket, we suggest going with personal access tokens, as the other solutions require premium plans. Alternatively, you can use local disk if you just want to look at locally checked out repos.

If you go with locally cloned repos. Just clone the repos that you're interested in to `/home/<XXXXX>/your-local-repos` or whatever you put in the docker compose file. Then comper will be able to scan them.

## Create an account

Just do a sign-up in the system. Go to `localhost:8001` to inspect it.

## Create a board

Create a board, then navigate to settings on the left, go to sources and configure your source. For local dirs: use a directory `/comper/repos`. Comper will now start ananlyzing the repos in four steps for each:

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
