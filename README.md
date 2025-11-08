# Comper Private Beta

Hey there, welcome to the Comper private beta for developers.

You’ll get an access key from Jouke, which is a docker hub token.

Make sure you have docker (with docker compose support) on your system.

Then run:

`docker login -u comperio`

And use the token provided to you.

Then use the following docker-compose.yml file:


## Docker Compose File


```
volumes:
  postgres:

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

  mailhog:
    image: mailhog/mailhog
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
    environment:
      - DATABASE_URL=postgres://loco:loco@postgres:5432/comper
      - JWT_SECRET=ZWVuZzh0ZWl0MnpvaDRXYTgK
      - CPU_PER_WORKER=15
      - WORKERS=1
      - FRONTEND_URL=http://localhost:8001
      - PASSWORDS_ENABLED=true
    volumes:
      - /home/<XXXXX>/tmp/comper:/comper/storage
      - /home/<XXXXX>/your-local-repos:/comper/repos
```

Replace the CPU and XXX placeholders based on your system. If you're paranoid and change the JWT secret, change it to something 


## Starting up

Run `docker compose up -d`

## Repo analysis

I suggest going with locally cloned repos for now. Just clone the repos that you're interested in to `/home/<XXXXX>/your-local-repos` or whatever you put in the docker compose file. Then comper will be able to scan them.

## Create an account

Just do a sign-up in the system. Go to `localhost:8001` to inspect it.

## Create a board

Create a board, then navigate to settings on the left, go to sources and use a directory `/comper/repos`. It will discover the repos and start analyzing them.

## Deduplicate contributors

When the analysis is underway, go to settings again, then to Contributors. The idea is to create "Contributors" from "Aliases". Each contributor (an actual person) might have multiple git aliases.

Under "Active Duty" you can mark users as still being with the company or already left. This is useful for the bus factor calculation.
