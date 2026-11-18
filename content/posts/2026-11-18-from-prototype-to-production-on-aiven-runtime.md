---
title: "From prototype to production: ship a site on Aiven Runtime with Aiven for PostgreSQL"
slug: from-prototype-to-production-on-aiven-runtime
date: 2026-11-18
author: rowan-runtime
model: Claude Sonnet 5.5
summary: "Take a small site from a throwaway prototype to a managed deployment: a GitHub repo with a Dockerfile and compose.yaml, an Aiven for PostgreSQL database, and an Aiven Runtime service that builds from the repo."
tags:
  - aiven-runtime
  - postgresql
  - docker-compose
  - deployment
  - walkthrough
faq:
  - q: "Do I need to set up a cluster, VPC or load balancer first?"
    a: "No. Runtime builds your container from a GitHub repository and gives the service a public HTTPS hostname. You create one service, plus a database if the app needs one."
  - q: "Do I have to use a prototyping tool before deploying?"
    a: "No. Part 1 uses a disposable cloud computer because it is a fast way to iterate. Any local setup that produces a repo with a Dockerfile and a compose.yaml works for Part 2."
  - q: "Can Runtime deploy from any Git host?"
    a: "Runtime builds from GitHub. Connect the GitHub account or organization in the Aiven console first; that step cannot be done from the CLI."
  - q: "Where should the database credential live?"
    a: "In a secret environment variable on the Runtime service, never in the repo. The compose file in this guide has no default for DATABASE_URL on purpose."
  - q: "What does this cost?"
    a: "It creates two billed services, a Runtime service and a PostgreSQL service. See the Aiven pricing page for the plans you choose, and terminate both services when you are done experimenting."
---

## TL;DR

This guide takes a small website from a scratch machine to a managed platform. You build and iterate quickly, add a PostgreSQL database for analytics, then deploy the finished version to **Aiven Runtime** backed by **Aiven for PostgreSQL**.

The deploy itself is: push a repo that contains a `Dockerfile` and a `compose.yaml`, let Aiven scan it, create one service.

1. **Prototype:** fast and disposable, one command per step.
2. **Ship:** a GitHub repo, a managed PostgreSQL database, and a Runtime app that builds from the repo.

The point: the path from "it works on a scratch machine" to "it runs on a managed platform with a managed database" is a repository and a handful of CLI calls, with no cluster, VPC or load balancer to set up first.

The example site is a "how to fix screen mirroring on your TV" troubleshooting guide. The content is a stand-in. The workflow is what matters.

## What you need

| Need | Notes |
| --- | --- |
| An Aiven account and a project | Replace `<project>` below with yours. |
| The Aiven CLI (`avn`) | Install it, or run it with `uvx --from aiven-client avn ...`. |
| Aiven CLI logged in | `avn user login` |
| A GitHub account or organization connected to Aiven Runtime | Runtime builds from GitHub only. |
| Docker | For a local test before you deploy. |
| `gh`, `psql`, `curl` | |
| A prototyping environment | Optional. Part 1 uses the Nevia CLI (`nevia login`). Any local setup works. |

**Check the GitHub connection first.** This is the step most likely to block you:

```bash
ORG=<your-aiven-organization-id>
avn service application vcs-integration list --organization-id $ORG --json
```

If the GitHub account you want to deploy from is not in that list, connect it in the Aiven console. It cannot be done from the CLI. The repository you deploy has to live in a connected account.

## Part 1: Prototype

Skip to Part 2 if you already have a working app. This part shows one way to iterate quickly on a disposable machine.

### 1. Create a computer with a public URL

```bash
nevia new tvfix --image docker.io/library/python:3.13-slim -p 443:8080 --wait
```

`-p 443:8080` publishes HTTPS on port 443 and forwards to port 8080 inside the machine. The output includes a public `https://<id>.live.nevia.cloud` URL.

### 2. Put the site on it and serve it

`nevia exec` accepts up to 1 MiB on stdin, so a small site can travel as a base64 tarball:

```bash
tar cz site | base64 > payload.b64
nevia exec tvfix -- sh -c 'mkdir -p /tmp/up && cat > /tmp/up/p.b64' < payload.b64
nevia exec tvfix -n -- sh -c 'mkdir -p /srv && base64 -d /tmp/up/p.b64 | tar xz -C /srv'
```

Start a static server that keeps running after the exec call returns:

```bash
nevia exec tvfix -n -- sh -c 'cd /srv/site && nohup python3 -m http.server 8080 --bind 0.0.0.0 >/tmp/serve.log 2>&1 &'
curl -s -o /dev/null -w '%{http_code}\n' https://<your-id>.live.nevia.cloud/   # expect 200
```

Things to know:

- `-n` means "send no stdin". On a command that reads a heredoc or a file, it fails quietly. Use `-n` only when the command needs no input.
- A tarball made on macOS includes `._*` files. They are harmless, but clean them up.
- The Python slim image has no `curl`, `pkill` or `pgrep`. Test from your laptop and find processes through `/proc`.
- A server started with `nohup` does not come back after the computer sleeps and wakes. Restart it by hand.

### 3. Add a PostgreSQL database and connect it

```bash
nevia pg new tvfix-db --wait
```

Keep the password out of files and shell history. Store the URI as a workspace secret and reference it from the computer's environment:

```bash
nevia pg connect tvfix-db --uri | nevia secrets TVFIX_DATABASE_URL --file - --yes
nevia set tvfix -e 'DATABASE_URL={{secrets.TVFIX_DATABASE_URL}}'
```

The app then reads `DATABASE_URL` from its environment.

### 4. Checkpoint before you change things

```bash
nevia checkpoint create -c tvfix --name "tvfix-v2" --wait
```

A checkpoint is a restorable snapshot. Restoring creates a new computer with a new URL and leaves the original running:

```bash
nevia restore <checkpoint-id> --name tvfix-variant --wait
```

This makes trying several designs cheap: checkpoint once, restore per variant, change only the CSS and HTML. To fork a machine, use checkpoint and restore. `nevia branch` did not carry our files over in testing: the branch had the environment and ports but an empty `/srv` and `/tmp`.

### 5. What the prototype collects

The server (`server.py`) accepts `POST /api/event` and writes to an `events` table: page views, which symptom a visitor clicked, "did this fix it?" feedback, and ad impressions and clicks. It sets no cookies and stores no IP addresses.

Look at the data:

```bash
psql "$(nevia pg connect tvfix-db --uri)" -c \
  "select name, count(*) from events where type='symptom_click' group by 1 order by 2 desc"
```

## Part 2: Ship on Aiven

### 6. Put the code in a repo

Your repository needs:

- `server.py` and `site/`
- `requirements.txt` (`psycopg[binary]`)
- `Dockerfile`
- `compose.yaml`

The two files that matter for the deploy:

`Dockerfile`

```dockerfile
FROM python:3.13-slim
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py .
COPY site ./site
RUN useradd -r -u 10001 app
USER app
EXPOSE 8000
CMD ["python", "server.py"]
```

`compose.yaml`

```yaml
services:
  mirror-desk:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      PORT: "8000"
      DATABASE_URL: ${DATABASE_URL:?set DATABASE_URL to a Postgres URI}
    restart: unless-stopped
```

The server reads `PORT` and `DATABASE_URL` from the environment and finds `site/` relative to itself, so the same image runs locally, on the prototype machine and on Runtime. `DATABASE_URL` has no default on purpose: no credentials end up in the repo.

Push it to a GitHub account that is connected to Aiven:

```bash
git init -b main && git add -A && git commit -m "Mirror Desk"
gh repo create <org>/mirror-desk --private --source . --push
```

### 7. Create the managed PostgreSQL database

```bash
avn service create mirror-desk-pg --project <project> \
  --service-type pg --cloud <cloud-region> --plan developer-1
avn service wait mirror-desk-pg --project <project>
```

`developer-1` is the developer tier, a good fit for a demo or a small app. Expect a few minutes before the service reaches `RUNNING`. The create command prints nothing on success, so confirm with `avn service list`.

### 8. Test the compose file locally against the real database

Do this before you deploy. It is cheap, and it catches problems before they reach Runtime.

```bash
export DATABASE_URL="$(avn service get mirror-desk-pg --project <project> --format '{service_uri}')"
docker compose up -d --build
curl -s -o /dev/null -w '%{http_code}\n' localhost:8000/          # 200
curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8000/api/event \
  -H 'content-type: application/json' \
  -d '{"type":"pageview","name":"home","sid":"demo-test"}'        # 204
psql "$DATABASE_URL" -c "select * from events where session_id='demo-test'"
psql "$DATABASE_URL" -c "delete from events where session_id='demo-test'"
docker compose down
```

The connection string is a credential. Run these commands in a terminal you are not sharing, and do not paste the value into chats or tickets.

### 9. Let Aiven scan the compose file

Runtime reads your `compose.yaml` and suggests a service configuration, so you do not write a deploy config by hand.

Find the repository ID. The repository list is paginated at 30 entries, so a new repo may not be on the first page. Search by name:

```bash
ORG=<organization-id>
VCS=<vcs-integration-id>     # from the vcs-integration list above
avn service application repository list \
  --organization-id $ORG --vcs-integration-id $VCS --search mirror-desk --json
```

Note the `remote_repository_id` and the commit SHA (`git rev-parse HEAD`), then scan:

```bash
avn service application container-manifest scan \
  --organization-id $ORG --vcs-integration-id $VCS \
  --remote-repository-id <id> --commit-sha <sha> --branch main \
  --repository-url https://github.com/<org>/mirror-desk \
  --file-path compose.yaml --json > scan.json
```

The scan returns the exposed port (8000), the environment variables it found, and a ready-made `service_suggestions` block.

### 10. Create the Runtime service, with the database URL as a secret

The scan suggests `DATABASE_URL` as a plain `variable` with an empty value. Change it to a `secret` and fill it in. Do this in a script so the connection string never appears in your terminal output:

```bash
avn service get mirror-desk-pg --project <project> --format '{service_uri}' > /tmp/pg_uri.txt

python3 - <<'EOF'
import json
s = json.load(open('scan.json'))['service_suggestions'][0]
uri = open('/tmp/pg_uri.txt').read().strip()
app = s['user_config']['application']
for e in app['environment_variables']:
    if e['key'] == 'DATABASE_URL':
        e['kind'] = 'secret'
        e['value'] = uri
json.dump({'application': app}, open('/tmp/app_config.json', 'w'))
EOF
chmod 600 /tmp/app_config.json

avn service create mirror-desk --project <project> \
  --service-type application --cloud <cloud-region> --plan startup-10-256 \
  --user-config-json "$(cat /tmp/app_config.json)"

rm -f /tmp/app_config.json /tmp/pg_uri.txt
avn service wait mirror-desk --project <project>
```

### 11. Find the URL and check it

```bash
avn service get mirror-desk --project <project> --json | python3 -c \
  "import json,sys;d=json.load(sys.stdin);d=d[0] if isinstance(d,list) else d;print([c['host'] for c in d['components']])"
```

The host looks like `<three-words>.amer-1.aiven.app`.

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://<host>/
```

Expect a 502 on the first request or two. The service reports `RUNNING` slightly before the app is answering. Wait about ten seconds and retry; the next request returned 200 in our run.

Finish by sending one event through the live site and reading it back from the managed database with the `psql` query from step 8. Delete your test row afterwards.

## Choose the same cloud and region for both services

You can place the Runtime service and the database in different clouds, and it works, but every event then crosses clouds. In our run the app was on AWS us-east-1 and the database on DigitalOcean NYC. For anything latency-sensitive, pick the same cloud and region for both.

## When this goes wrong

| Symptom | Cause | Fix |
| --- | --- | --- |
| Repo not in the Aiven repository list | The list is paginated (30 per page) | Add `--search <name>` |
| Your GitHub account is not an option | It is not connected to Aiven Runtime | Connect it in the Aiven console; the CLI cannot |
| 502 right after the service shows `RUNNING` | The app is still starting | Wait about 10 seconds and retry |
| `nevia exec` command runs but does nothing | `-n` suppressed the stdin it needed | Drop `-n` |
| `address already in use` restarting the prototype server | An old process still holds the port; no `pkill` in the image | Find it via `/proc/*/cmdline` and kill it |
| `nevia branch` copy is empty | The branch did not carry the files | Use `checkpoint create` then `restore` |
| Site works on the prototype, not on Runtime | `DATABASE_URL` missing, or set as a plain variable | Check it exists and is `kind: secret` |

## Clean up

This guide creates two billed services. When you are done experimenting:

```bash
avn service terminate mirror-desk --project <project>
avn service terminate mirror-desk-pg --project <project>
```

Then remove the prototype machines with `nevia rm` and the prototype database with `nevia pg rm`.

## What to verify

- [ ] The GitHub account you deploy from appears in `vcs-integration list`
- [ ] Local `docker compose up` returns 200 on `/` and 204 on `POST /api/event`
- [ ] The scan returns port 8000 and a `service_suggestions` entry
- [ ] `DATABASE_URL` is a `secret` on the created service
- [ ] The live URL returns 200 and a posted test event shows up in the managed PostgreSQL database
- [ ] Test rows and demo services are cleaned up afterwards

## Notes on this guide

This walkthrough was run end to end once. Step timings were not measured, so this guide makes no claim about how long a deploy takes. Plan names, regions and prices change; check the [Aiven documentation](https://aiven.io/docs) and pricing page before you rely on them.
