# FlowSentra Installation Guide

FlowSentra runs as Docker containers from two pre-built images on Docker Hub: `flowsentra:backend` and `flowsentra:frontend`. We recommend Docker Compose for a single server and AWS ECS for production. Helm for Kubernetes is planned for a future release.

## Overview

The AWS ECS setup deploys the following:

- A VPC with public and private subnets.
- An ECS cluster of four `t4g.small` EC2 hosts in the private subnets.
- 12 ECS services: the frontend, the API and 10 background workers.
- An Application Load Balancer in the public subnets. It sends `/api/*` to the API and every other request to the frontend.
- An RDS PostgreSQL 15 database in the private subnets, with no public access.
- An ECR repository, `flowsentra`, for the container images.
- A CloudWatch Logs group, `/ecs/flowsentra`, for all container logs.

### FlowSentra AWS architecture diagram

This is how FlowSentra runs on AWS: the load balancer in front, the containers on the ECS cluster in the private subnet, the RDS database, and the logs in CloudWatch.

![FlowSentra on AWS](docs/aws-architecture.png)

FlowSentra connects to two applications outside AWS. They sit in the box at the bottom of the diagram, and the workers reach them through the NAT gateway.

- **Apache Airflow:** the DAG workers read DAG details, runs, task instances and lineage from it.
- **Tableau:** the Tableau workers read workbook metadata and runs from it.

GitHub and CodePipeline in the diagram build and push new images to ECR. They are part of the deployment pipeline, not of the application.

## System Requirements

FlowSentra is made of four parts. You need all four to run it:

- PostgreSQL 15: the database. Docker Compose starts it for you. On AWS, use RDS.
- The backend image: runs the API and 10 background workers.
- The frontend image: the web page users see in the browser.
- A reverse proxy or load balancer: the single public entry point. Docker Compose uses Caddy. AWS uses an Application Load Balancer (ALB).

### Server size

Frontend and API must be served from the same domain. The frontend image calls the relative path /api/v1.

FlowSentra runs 12 containers all the time: 1 API, 10 workers and 1 frontend. Together they need about 4 GB of RAM. This is a starting estimate, so check the real use with `docker stats` and adjust.

Recommended for production on AWS: four `t4g.small` instances. Each has 2 vCPUs and 2 GB of RAM, so together they give about 8 GB, which is enough for the 12 containers. One `t4g.small` on its own is not enough.

### CPU type

A Docker image is built for one type of CPU. The two common types are arm64 and amd64 (also called x86\_64).

| EC2 instance type | CPU type | Image needed |
| --- | --- | --- |
| `t4g`, `m6g`, `c6g` | arm64 | `linux/arm64` |
| `t3`, `m5`, `c5` | amd64 | `linux/amd64` |

`t4g.small` uses arm64, so the FlowSentra images must be built for arm64. If an image was built for the other CPU type, the container stops straight away and its log shows `exec format error`. That error means the image and the server do not match.

To check which CPU type an image supports, run:

```bash
docker manifest inspect DOCKERHUB_ORG/flowsentra:backend | grep architecture
```

The output must include `arm64` when you deploy on `t4g`.

### One domain for the whole app

Users open FlowSentra at one web address, for example `https://flowsentra.yourdomain.com`. Behind that address there are two programs: the frontend (the pages you see) and the API (the data behind the pages). The load balancer reads the path at the end of the address and sends the request to the right one:

| Request | Goes to |
| --- | --- |
| `https://flowsentra.yourdomain.com/` | Frontend |
| `https://flowsentra.yourdomain.com/api/v1/...` | API |

The load balancer needs only two rules for this. The AWS ECS section sets them up for you.

Do not give the API its own domain, such as `api.yourdomain.com`. The frontend calls the API as `/api/v1` with no domain in front, so the browser sends the call to the same address the page was opened from. With a second domain, those calls would never reach the API.

## Docker Compose

Docker Compose is a simple way to run FlowSentra on a single server. The file `docker-compose.prod.yml` is the only file you need. You do not need the source code. Docker downloads the two images by itself from the FlowSentra repository on Docker Hub.

**Step 1: Install Docker.** On a Mac or Windows PC, install Docker Desktop from docker.com and open it. Wait until it says Docker is running. On Debian or Ubuntu, run `sudo apt install docker.io`. Then check that Docker is ready:

```bash
docker info
```

The `Server:` part of the output must show details. If it says "Cannot connect to the Docker daemon", Docker is not running yet.

**Step 2: Make a folder.**

```bash
mkdir flowsentra
cd flowsentra
```

**Step 3: Save the compose file.** Save the whole file below in this folder with exactly this name: `docker-compose.prod.yml`. To save it from a terminal, copy the file, then run this command:

```yaml
# ---------------------------------------------------------------------------
# FlowSentra — client-site production deployment
#
# Pre-built-image version of docker/docker-compose.yml, meant to run
# standalone on a target host using ONLY the two images already pushed to
# Docker Hub — no source checkout, no DDL files to find or copy anywhere.
# Schema bootstrap (db-init) and the first admin user (admin-seed) are both
# one-shot jobs run from the backend image itself, which already has the DDL
# files baked in (see pyinstaller.spec) — see DEPLOYMENT_GUIDE.md §3.2.
#
# Before using this file:
#   1. Export DOCKERHUB_ORG (e.g. flowsentra). BACKEND_TAG / FRONTEND_TAG
#      default to "backend" / "frontend" — override to pin a build.
#   2. Copy .env.example -> .env and fill in every REQUIRED value (this
#      includes ADMIN_EMAIL/ADMIN_PASSWORD — the admin-seed job below uses
#      them to create the first login automatically, no manual exec step).
#   3. Nothing else. The reverse-proxy config is embedded in this file (see
#      the `configs:` block at the bottom) — this YAML is the only file the
#      client needs. For HTTPS, export SITE_ADDRESS=their.domain.
#
# No ports are published directly from postgres/api/frontend to the host —
# the caddy service is the only public entry point (80/443). This is
# deliberate: see DEPLOYMENT_GUIDE.md §1 for why frontend+api must be
# same-origin.
# ---------------------------------------------------------------------------

# NOTE: backend and frontend live in ONE Docker Hub repo distinguished by TAG
# (flowsentra/flowsentra:backend and :frontend), so they need separate tag
# variables — a single shared TAG cannot be both at once.
x-backend-image: &backend-image ${DOCKERHUB_ORG:?set DOCKERHUB_ORG}/${BACKEND_IMAGE:-flowsentra}:${BACKEND_TAG:-backend}

x-backend-env: &backend-env
  L1_DB_URL: postgresql+psycopg2://l1:${L1_APP_PASSWORD:?set L1_APP_PASSWORD in .env}@postgres:5432/l1_postgres_db
  POSTGRES_DB: l1_postgres_db
  POSTGRES_HOST: postgres
  PGHOST: postgres
  PGPORT: "5432"
  PGUSER: l1
  PGPASSWORD: ${L1_APP_PASSWORD}

services:
  postgres:
    image: postgres:15
    container_name: flowsentra_postgres
    restart: unless-stopped
    env_file: [.env]
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
      POSTGRES_DB: l1_postgres_db
      TZ: UTC
      PGTZ: UTC
      L1_APP_PASSWORD: ${L1_APP_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks: [flowsentra]

  # One-shot bootstrap jobs — run once to completion, then exit 0 and stay
  # stopped. Both are safe to re-run (CREATE ... IF NOT EXISTS / ON CONFLICT
  # DO NOTHING throughout), so `docker compose up` re-running them on a
  # restart is harmless.
  db-init:
    image: *backend-image
    container_name: flowsentra_db_init
    command: ["/app/l1platform", "init-db"]
    restart: "no"
    environment:
      PGHOST: postgres
      PGPORT: "5432"
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
      POSTGRES_DB: l1_postgres_db
      L1_APP_PASSWORD: ${L1_APP_PASSWORD:?set L1_APP_PASSWORD in .env}
    depends_on:
      postgres: { condition: service_healthy }
    networks: [flowsentra]

  admin-seed:
    image: *backend-image
    container_name: flowsentra_admin_seed
    command: ["/app/l1platform", "seed-admin"]
    restart: "no"
    env_file: [.env]
    environment:
      <<: *backend-env
      ADMIN_EMAIL: ${ADMIN_EMAIL:?set ADMIN_EMAIL in .env}
      ADMIN_PASSWORD: ${ADMIN_PASSWORD:?set ADMIN_PASSWORD in .env}
    depends_on:
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  api:
    image: *backend-image
    container_name: flowsentra_api
    restart: unless-stopped
    command: ["/app/l1platform", "api"]
    env_file: [.env]
    environment:
      <<: *backend-env
      PGUSER: postgres
      PGPASSWORD: ${POSTGRES_PASSWORD}
      DRY_RUN: "false"
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  # 12 background workers. Every worker is a singleton: never scale
  # replicas: > 1 on any of these.
  dag_details_worker:
    image: *backend-image
    container_name: flowsentra_dag_details_worker
    restart: unless-stopped
    command: ["/app/l1platform", "dag_details_worker"]
    env_file: [.env]
    environment: *backend-env
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  airflow_dag_lineage_worker:
    image: *backend-image
    container_name: flowsentra_airflow_dag_lineage_worker
    restart: unless-stopped
    command: ["/app/l1platform", "airflow_dag_lineage_worker"]
    env_file: [.env]
    environment: *backend-env
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  dag_runs_worker:
    image: *backend-image
    container_name: flowsentra_dag_runs_worker
    restart: unless-stopped
    command: ["/app/l1platform", "dag_job_runs_worker"]
    env_file: [.env]
    environment:
      <<: *backend-env
      DRY_RUN: "false"
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  dag_task_instances_worker:
    image: *backend-image
    container_name: flowsentra_dag_task_instances_worker
    restart: unless-stopped
    command: ["/app/l1platform", "dag_task_instances_worker"]
    env_file: [.env]
    environment: *backend-env
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  tableau_metadata_worker:
    image: *backend-image
    container_name: flowsentra_tableau_metadata_worker
    restart: unless-stopped
    command: ["/app/l1platform", "tableau_metadata_worker"]
    env_file: [.env]
    environment:
      <<: *backend-env
      API_BASE_URL: http://api:8000
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
      api: { condition: service_started }
    networks: [flowsentra]

  tableau_runs_worker:
    image: *backend-image
    container_name: flowsentra_tableau_runs_worker
    restart: unless-stopped
    command: ["/app/l1platform", "tableau_runs_worker"]
    env_file: [.env]
    environment:
      <<: *backend-env
      API_BASE_URL: http://api:8000
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
      api: { condition: service_started }
    networks: [flowsentra]

  lineage_worker:
    image: *backend-image
    container_name: flowsentra_lineage_worker
    restart: unless-stopped
    command: ["/app/l1platform", "lineage_worker"]
    env_file: [.env]
    environment: *backend-env
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]
    # For git-based lineage extraction, set GIT_REPO_PATH in .env and mount
    # a read-only DAG repo checkout here, e.g.:
    # volumes:
    #   - /path/to/dags/checkout:/repos/dags:ro

  workbook_lineage_sla_worker:
    image: *backend-image
    container_name: flowsentra_workbook_lineage_sla_worker
    restart: unless-stopped
    command: ["/app/l1platform", "workbook_lineage_sla_worker"]
    env_file: [.env]
    environment: *backend-env
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  workbook_chain_sla_worker:
    image: *backend-image
    container_name: flowsentra_workbook_chain_sla_worker
    restart: unless-stopped
    command: ["/app/l1platform", "workbook_chain_sla_worker"]
    env_file: [.env]
    environment: *backend-env
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  incident_worker:
    image: *backend-image
    container_name: flowsentra_incident_worker
    restart: unless-stopped
    command: ["/app/l1platform", "incident_worker"]
    env_file: [.env]
    environment:
      <<: *backend-env
      DRY_RUN: "false"
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  maintenance_window_worker:
    image: *backend-image
    container_name: flowsentra_maintenance_window_worker
    restart: unless-stopped
    command: ["/app/l1platform", "maintenance_window_worker"]
    env_file: [.env]
    environment:
      <<: *backend-env
      # Sends the 30-minutes-before warning email. Without an explicit
      # DRY_RUN the app defaults to dry run and discards every email.
      DRY_RUN: "false"
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  # Deactivates users past their expiry date.
  user_expiry_worker:
    image: *backend-image
    container_name: flowsentra_user_expiry_worker
    restart: unless-stopped
    command: ["/app/l1platform", "user_expiry_worker"]
    env_file: [.env]
    environment: *backend-env
    depends_on:
      postgres: { condition: service_healthy }
      db-init: { condition: service_completed_successfully }
    networks: [flowsentra]

  frontend:
    image: ${DOCKERHUB_ORG:?set DOCKERHUB_ORG}/${FRONTEND_IMAGE:-flowsentra}:${FRONTEND_TAG:-frontend}
    container_name: flowsentra_frontend
    restart: unless-stopped
    # Only works if the image was built with VITE_API_BASE_URL=/api/v1
    # (a relative path).
    networks: [flowsentra]

  # Reverse proxy — the only container with published host ports. For HTTPS
  # on a real domain, export SITE_ADDRESS=your.domain and Caddy handles
  # Let's Encrypt itself.
  caddy:
    image: caddy:2-alpine
    container_name: flowsentra_caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    configs:
      - source: caddyfile
        target: /etc/caddy/Caddyfile
    volumes:
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - api
      - frontend
    networks: [flowsentra]

# Reverse-proxy config, embedded so this YAML is the only file to hand over.
# SITE_ADDRESS unset -> ":80", plain HTTP, right for a local trial.
configs:
  caddyfile:
    content: |
      ${SITE_ADDRESS:-:80} {
      	encode gzip

      	# Backend first — the catch-all below would otherwise swallow it.
      	handle /api/v1/* {
      		reverse_proxy api:8000
      	}

      	# Everything else is the React app.
      	handle {
      		reverse_proxy frontend:3000
      	}

      	log {
      		output stdout
      		format console
      	}
      }

networks:
  flowsentra:
    driver: bridge

volumes:
  pgdata:
  caddy_data:
  caddy_config:
```

```bash
cat > docker-compose.prod.yml
```

Paste the yaml, press Return, then press Control+D. Run `ls` and check that the file is there.

**Step 4: Set your values.** Replace the words after the equals signs with your own values.

```bash
export DOCKERHUB_ORG=flowsentra
export POSTGRES_PASSWORD=choose-a-password
export L1_APP_PASSWORD=choose-another-password
export ADMIN_EMAIL=you@company.com
export ADMIN_PASSWORD='choose-a-strong-password'
```

- `DOCKERHUB_ORG`: leave it as `flowsentra`. It tells Docker where to download the images.
- `POSTGRES_PASSWORD`: a password you choose for the database admin. You do not type it again later.
- `L1_APP_PASSWORD`: a different password you choose for the database user the app uses.
- `ADMIN_EMAIL` and `ADMIN_PASSWORD`: the email and password you will use to sign in to FlowSentra.

Do not use spaces or quote characters inside the passwords. These values only last for the terminal window you typed them in, so run all the remaining steps in that same window.

**Step 5: Create the .env file.** This creates the two keys that sign the login tokens. Run it once.

```bash
printf 'JWT_SECRET_KEY=%s\nJWT_REFRESH_SECRET_KEY=%s\n' "$(openssl rand -hex 32)" "$(openssl rand -hex 32)" > .env
```

Running it again replaces the keys and signs everyone out. To connect Airflow, Tableau, email and the other services, add their settings to this file later. See the Configuration section below.

**Step 6: Start FlowSentra.**

```bash
docker compose -f docker-compose.prod.yml up -d
```

The first time, Docker downloads the two images from Docker Hub, which takes a few minutes. You do not need the source code.

**Step 7: Check that it started.**

```bash
docker compose -f docker-compose.prod.yml ps
curl http://localhost/api/v1/health
```

Every service should say `Up`, and Postgres should say `(healthy)`. The two setup jobs, `db-init` and `admin-seed`, run once and then exit, so they do not appear in the list. The health check should return `{"success": true, "data": {"status": "ok"}, ...}`.

**Step 8: Sign in.** Open `http://localhost` in your browser and sign in with `ADMIN_EMAIL` and `ADMIN_PASSWORD`. If the sign-in fails, type the password by hand instead of letting the browser fill it in. The dashboard stays mostly empty until you connect Airflow and Tableau.

`seed-admin` always creates the username `admin`, so you cannot create a second admin with it. Add more users from the Config page in the app.

On first start, the `db-init` job creates the `l1` database role and applies the schema, then `admin-seed` creates the first admin login from `ADMIN_EMAIL` and `ADMIN_PASSWORD`. Both are safe to re-run.

Caddy is the only service with published ports (80 and 443). For HTTPS, export `SITE_ADDRESS=your.domain` and Caddy gets a Let's Encrypt certificate itself. Without it, FlowSentra is served over plain HTTP on port 80, which is right for a local trial.

To stop and remove the containers:

```bash
docker compose -f docker-compose.prod.yml down
```

The frontend must be built with `VITE_API_BASE_URL=/api/v1`. The command above passes no build argument, so confirm the Dockerfile sets that value by default.

### Upgrading

1. Set `BACKEND_TAG` and `FRONTEND_TAG` to the new release.
2. Pull the latest images with `docker compose -f docker-compose.prod.yml pull`.
3. Restart with `docker compose -f docker-compose.prod.yml up -d`.

`db-init` runs again on start and is safe to re-run.

## AWS ECS

AWS ECS on EC2 with RDS and an Application Load Balancer is the recommended production setup. The ALB replaces Caddy and RDS replaces the `postgres` container. You create the database tables and the database user by hand in Step 5, so no database task is needed in ECS. Follow the steps in order, since each one depends on the one before it.

### What you will create

| Compose service | AWS equivalent |
| --- | --- |
| `postgres` | RDS PostgreSQL 15 |
| Docker Hub images | ECR repository `flowsentra` |
| `caddy` | Application Load Balancer, ACM certificate, Route53 record |
| `frontend` | ECS service, port 3000 |
| `api` | ECS service, port 8000 |
| 10 workers | One ECS service each, desired count 1 |

Traffic flow: the user opens your domain, Route53 sends it to the ALB, and the ALB forwards `/api/*` to the API (port 8000) and everything else to the frontend (port 3000). Workers have no inbound traffic.

### Before you start

Have these ready:

- AWS account, region, VPC, and at least two public and two private subnets.
- The domain you will use, for example `flowsentra.yourdomain.com`, in Route53.
- The CPU type of the EC2 instance. `t4g`, `m6g` and `c6g` are arm64; `t3`, `m5` and `c5` are x86\_64. The image platform must match.
- The production `.env` values from the Configuration section, including the API health check path.
- Access to the FlowSentra images on Docker Hub (`DOCKERHUB_ORG`).

Check which platforms the images were built for:

```bash
docker manifest inspect DOCKERHUB_ORG/flowsentra:backend | grep architecture
docker manifest inspect DOCKERHUB_ORG/flowsentra:frontend | grep architecture
```

If the platform does not match your instance type, choose a matching instance type or ask for a rebuild.

### Step 1: Create the RDS database

1. Go to RDS, then Create database.
2. Engine: PostgreSQL, version 15.
3. DB instance identifier: `flowsentra-db`.
4. Credentials: self managed. Username `postgres`. Set a password; it becomes `POSTGRES_PASSWORD`. Write it down, because AWS cannot show it again.
5. Instance class: `db.t4g.small` for an arm64 cluster.
6. VPC: your VPC. Public access: No.
7. Additional configuration, Initial database name: leave it empty. You create the database in Step 5.
8. Create the database and wait 5 to 10 minutes. Note the endpoint.

RDS for PostgreSQL 15 requires SSL by default. If a service logs `no pg_hba.conf entry`, add `?sslmode=require` to `L1_DB_URL` and set `PGSSLMODE=require`.

### Step 2: Copy the images to ECR

Create a private ECR repository named `flowsentra`. Then, from a machine with Docker, pull both images for your host platform and push them. Use `linux/arm64` for t4g and `linux/amd64` for t3.

```bash
aws ecr get-login-password --region REGION | \
  docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com

docker pull --platform linux/arm64 DOCKERHUB_ORG/flowsentra:backend
docker pull --platform linux/arm64 DOCKERHUB_ORG/flowsentra:frontend

docker tag DOCKERHUB_ORG/flowsentra:backend  ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/flowsentra:backend
docker tag DOCKERHUB_ORG/flowsentra:frontend ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/flowsentra:frontend

docker push ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/flowsentra:backend
docker push ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/flowsentra:frontend
```

The frontend image must have been built with `VITE_API_BASE_URL=/api/v1`. This value is baked in at build time, so it cannot be changed in ECS.

### Step 3: Create the ECS cluster

1. Go to ECS, Clusters, Create cluster. Name: `flowsentra-cluster`.
2. Infrastructure: Amazon EC2 instances
3. Instance type: `t4g.small` (arm64, 2 GB RAM). The 12 containers need about 4 GB in total, so use four instances, which give about 8 GB.
4. Auto Scaling group: choose Create new ASG. This is the group that starts and replaces the EC2 instances for you. Set Minimum and Maximum capacity to 4 (and Desired to 4) so it always keeps four instances. VPC: the same VPC as RDS. Subnets: private.
5. Create the cluster.
6. Create the CloudWatch log group `/ecs/flowsentra`. Every task writes to it with a different stream prefix.

### Step 4: Create the load balancer

1. Request an ACM certificate for your domain with DNS validation. Add the CNAME records in Route53 and wait until the status is Issued.
2. Go to EC2, Load Balancers, Create Application Load Balancer. Name: `flowsentra-alb`. Scheme: Internet-facing.
3. VPC: the same one. Subnets: at least two public subnets in different availability zones.
4. Security group: allow 80 and 443 from `0.0.0.0/0`. Then allow this group to reach the instance security group on the ephemeral port range, since bridge mode uses dynamic host ports.
5. Create two target groups, both HTTP, type Instances, in the same VPC:

| Target group | Port | Health check path |
| --- | --- | --- |
| `flowsentra-ui-tg` | 3000 | `/` |
| `flowsentra-api-tg` | 8000 | The API health path from your team |

6. Listener HTTP:80: redirect to HTTPS:443.
7. Listener HTTPS:443: attach the ACM certificate and set the default action to forward to `flowsentra-ui-tg`. The default action handles every request that no other rule matches, so the website itself loads from the frontend.
8. In the HTTPS:443 listener, add a rule: Host header is `your.domain` and Path is `/api/*`, forward to `flowsentra-api-tg`, priority 1.
9. In Route53, create an A record (alias) for your domain that points to `flowsentra-alb`.

### Step 5: Initialize the database

AWS RDS creates an empty database. In this step you connect to it, create the schemas and tables, and create the application user.

**5.1 Why you cannot use CloudShell**

CloudShell runs outside your VPC, and RDS is inside your VPC with no public access. So you must connect from an EC2 instance, which is inside the VPC.

**5.2 Connect to EC2 Using SSM**

```bash
# In CloudShell or your terminal:
aws ssm start-session --target i-YOURINSTANCEID --region REGION

# Then switch to root:
sudo su
```

**5.3 Install the PostgreSQL client**

```bash
yum install -y postgresql15
```

**5.4 Create the Database**

RDS created only the database engine, so create the database yourself. Choose a name, for example `l1_postgres_db`, and use the same name wherever the commands below say `your_database_name`.

```bash
PGPASSWORD=POSTGRES_PASSWORD psql \
  -h RDS_ENDPOINT \
  -U postgres \
  -d postgres \
  -c 'CREATE DATABASE your_database_name;'
```

**5.5 Create the schemas**

Create any schema the FlowSentra SQL files use besides `public`. The L1 database used `config` and `target`. Ask the developers for the exact list.

```bash
PGPASSWORD=POSTGRES_PASSWORD psql \
  -h RDS_ENDPOINT \
  -U postgres \
  -d your_database_name \
  -c "CREATE SCHEMA IF NOT EXISTS public;
      CREATE SCHEMA IF NOT EXISTS config;
      CREATE SCHEMA IF NOT EXISTS target;"
```

**5.6 Run the DDL SQL files**

The `.sql` files contain the commands that create the tables. Get them from the developers. Copy the content of each file and paste it as a command:

```bash
PGPASSWORD=POSTGRES_PASSWORD psql \
  -h RDS_ENDPOINT \
  -U postgres \
  -d your_database_name \
  -c "CREATE TABLE IF NOT EXISTS your_table (...)"
```

**5.7 Create the application user**

The application connects as the `l1` user. `L1_APP_PASSWORD` is the password you choose now. Use the same value in the task definitions in Step 6.

```bash
PGPASSWORD=POSTGRES_PASSWORD psql \
  -h RDS_ENDPOINT \
  -U postgres -d your_database_name -c "
  CREATE USER l1 WITH PASSWORD 'L1_APP_PASSWORD';
  GRANT ALL PRIVILEGES ON DATABASE your_database_name TO l1;
  GRANT ALL ON ALL TABLES IN SCHEMA public TO l1;
  GRANT ALL ON ALL TABLES IN SCHEMA config TO l1;
  GRANT ALL ON ALL TABLES IN SCHEMA target TO l1;
"
```

These grants follow the L1 setup. Ask the developers whether FlowSentra needs more, such as access to sequences.

### Step 6: Create the task definitions

A task definition tells ECS how to run one container: the image, the command, the memory and the settings. Create 12 of them: the API, the frontend and the 10 workers. The 11 backend ones use the same image, `flowsentra:backend`, and differ only in their name, command and a few settings. Go to ECS, Task definitions, Create new task definition, Create new task definition with JSON.

**6.1 Settings for every task definition**

| Setting | Value |
| --- | --- |
| Launch type | EC2 |
| Network mode | `bridge`. ECS publishes the container on a free port of the host and the load balancer finds it. |
| CPU | 256 |
| Log driver | `awslogs`, log group `/ecs/flowsentra`, stream prefix = the service name |

**6.2 Environment variables (all 11 backend task definitions)**

Add these environment variables to every backend task definition. Type Value means you type the value in the task definition. Type Secret means the value is stored in AWS Secrets Manager. Create one secret for each (Secrets Manager, Store a new secret, Other type of secret, Plaintext), then in the task definition choose ValueFrom and paste the secret ARN. `RDS_ENDPOINT` is the database address from Step 1 and `L1_APP_PASSWORD` is the password you chose in Step 5.7.

| Name | Type | Value |
| --- | --- | --- |
| `L1_DB_URL` | Secret | `postgresql+psycopg2://l1:L1_APP_PASSWORD@RDS_ENDPOINT:5432/your_database_name?sslmode=require` |
| `POSTGRES_HOST` | Value | `RDS_ENDPOINT` |
| `POSTGRES_PORT` | Value | `5432` |
| `POSTGRES_DB` | Value | `your_database_name` |
| `POSTGRES_SCHEMA` | Value | `target` |
| `POSTGRES_USER` | Value | `postgres` |
| `POSTGRES_PASSWORD` | Secret | your RDS admin password |
| `API_BASE_URL` | Value | `https://your.domain` |
| `DRY_RUN` | Value | `false` |
| `AIRFLOW_BASE_URL` | Value | your Airflow address |
| `AIRFLOW_API_VERSION` | Value | `v1` |
| `AIRFLOW_USERNAME` | Value | your Airflow user |
| `AIRFLOW_PASSWORD` | Secret | password of that user |
| `AIRFLOW_TIMEOUT` | Value | `300` |
| `AIRFLOW_PAGE_SIZE` | Value | `100` |
| `AIRFLOW_HOURS_LOOKBACK` | Value | `2` |
| `AIRFLOW_TLS_VERIFY` | Value | `0` |
| `TABLEAU_BASE_URL` | Value | your Tableau address |
| `TABLEAU_API_VERSION` | Value | `3.28` |
| `TABLEAU_SITE_ID` | Value | your Tableau site ID |
| `CONTENT_URL` | Value | your Tableau site content URL |
| `PAT_NAME` | Value | name of your Tableau access token |
| `PAT_SECRET` | Secret | secret of that token |
| `SMTP_HOST` | Value | `smtp.gmail.com` |
| `SMTP_PORT` | Value | `587` |
| `SMTP_USER` | Value | your mail user |
| `SMTP_PASS` | Secret | password of that mail user |
| `SMTP_FROM` | Value | the sender email address |
| `ANTHROPIC_API_KEY` | Secret | your Anthropic API key |
| `ANTHROPIC_MODEL` | Value | `claude-sonnet-4-6` |

**6.3 The 12 task definitions**

Use the image `ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/flowsentra:backend`, or `:frontend` for the frontend. The command is `["/app/l1platform", "<value>"]`, where `<value>` is the Command column. The frontend has no command.

| Name | Command | Port | Memory | Extra settings |
| --- | --- | --- | --- | --- |
| `flowsentra-api` | `api` | 8000 | 512 MB | `JWT_SECRET_KEY` and `JWT_REFRESH_SECRET_KEY` (both Secret), `DRY_RUN=false` |
| `flowsentra-dag-details` | `dag_details_worker` | none | 256 MB | `DAG_DETAILS_HEARTBEAT_URL`, optional |
| `flowsentra-dag-runs` | `dag_job_runs_worker` | none | 256 MB | `DRY_RUN=false` |
| `flowsentra-dag-task-instances` | `dag_task_instances_worker` | none | 256 MB | none |
| `flowsentra-tableau-metadata` | `tableau_metadata_worker` | none | 256 MB | `API_BASE_URL=https://your.domain` |
| `flowsentra-tableau-runs` | `tableau_runs_worker` | none | 256 MB | `API_BASE_URL=https://your.domain` |
| `flowsentra-lineage` | `lineage_worker` | none | 256 MB | `GIT_REPO_PATH`, only if you use git lineage |
| `flowsentra-workbook-lineage-sla` | `workbook_lineage_sla_worker` | none | 256 MB | none |
| `flowsentra-workbook-chain-sla` | `workbook_chain_sla_worker` | none | 256 MB | none |
| `flowsentra-incident` | `incident_worker` | none | 256 MB | `DRY_RUN=false` |
| `flowsentra-maintenance-window` | `maintenance_window_worker` | none | 256 MB | `DRY_RUN=false` |
| `flowsentra-frontend` | none | 3000 | 128 MB | none. No database settings. |

Extra settings are added to the database settings in 6.2. What they mean:

- `JWT_SECRET_KEY` and `JWT_REFRESH_SECRET_KEY`: the API only. They are the keys that sign the login tokens, so generate two long random strings and keep them as secrets.
- `DRY_RUN`: set it to `false` on the API, dag-runs, incident and maintenance-window tasks. Without it the app only pretends, so no emails are sent.
- `API_BASE_URL`: set it to `https://your.domain` on the two Tableau workers only. It is the address they use to call the API.
- `DAG_DETAILS_HEARTBEAT_URL`: optional, dag-details worker only. It is the Uptime Kuma push address the worker pings to show it is alive. `GIT_REPO_PATH`: optional, lineage worker only. Skip it unless you use git-based lineage.

**6.4 Example: the API**

Paste this into Create new task definition with JSON and replace the capital-letter words (`ACCOUNT_ID`, `REGION`, `RDS_ENDPOINT`, `SECRET_ARN`). Add the other variables from 6.2 in the same way, as `environment` for Type Value and as `secrets` for Type Secret.

```json
{
  "family": "flowsentra-api",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/flowsentra:backend",
      "essential": true,
      "cpu": 256,
      "memory": 512,
      "command": ["/app/l1platform", "api"],
      "portMappings": [{"containerPort": 8000, "hostPort": 0, "protocol": "tcp"}],
      "environment": [
        {"name": "POSTGRES_HOST", "value": "RDS_ENDPOINT"},
        {"name": "POSTGRES_PORT", "value": "5432"},
        {"name": "POSTGRES_DB", "value": "your_database_name"},
        {"name": "POSTGRES_SCHEMA", "value": "target"},
        {"name": "POSTGRES_USER", "value": "postgres"},
        {"name": "API_BASE_URL", "value": "https://your.domain"},
        {"name": "DRY_RUN", "value": "false"}
      ],
      "secrets": [
        {"name": "L1_DB_URL", "valueFrom": "SECRET_ARN"},
        {"name": "POSTGRES_PASSWORD", "valueFrom": "SECRET_ARN"},
        {"name": "JWT_SECRET_KEY", "valueFrom": "SECRET_ARN"},
        {"name": "JWT_REFRESH_SECRET_KEY", "valueFrom": "SECRET_ARN"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/flowsentra",
          "awslogs-region": "REGION",
          "awslogs-stream-prefix": "api"
        }
      }
    }
  ]
}
```

Using the AWS CLI instead? Save the JSON as `api-task.json` and run:

```bash
aws ecs register-task-definition --cli-input-json file://api-task.json --region REGION
```

**Making the other 11.** Copy the file and change only these things:

1. `family`: the task definition name from the table.
2. `name` and `awslogs-stream-prefix`: any short, unique name, for example `incident`.
3. `command`: the one from the table.
4. Extra settings: add only what the table says, delete the whole `portMappings` line, and remove the JWT keys (the API is the only task that uses them).
5. Frontend only: use the `:frontend` image and port 3000, and delete `command` and `environment`.

The task execution role (`ecsTaskExecutionRole`) must be allowed to read the secrets. In IAM, add the permission `secretsmanager:GetSecretValue` for your secrets, or the tasks will fail to start.

### Step 7: Create the services

1. Go to ECS, Clusters, `flowsentra-cluster`, Services, Create.
2. Launch type: EC2. Choose the task definition and name the service `<family>-service`.
3. Desired tasks: 1. Leave Service auto scaling turned off. It adds or removes copies of a service, and FlowSentra services must stay at one copy.
4. Load balancing: attach `flowsentra-api-tg` to the API service and `flowsentra-ui-tg` to the frontend service. Workers get no load balancer.
5. Create the API first, then the frontend, then the 10 workers. The Tableau workers call the API, so start them after it is healthy.

Never set the desired count above 1 on a worker. Duplicate workers process the same jobs and send duplicate emails.

On a small instance, set Minimum healthy percent to 0 so ECS stops the old task before starting the new one. Otherwise a deployment tries to run both at once and fails for lack of memory. This causes a gap of about 30 seconds per service during a release.

### Step 8: Verify

1. All 12 services show running 1 of 1 in ECS.
2. Both target groups show the target as Healthy.
3. `https://your.domain/api/v1/health` (use your API health path) returns a success response.
4. `https://your.domain` loads and you can log in.
5. In the browser Network tab, calls to `/api/v1/...` return 200.

If a worker keeps restarting, read its log stream in `/ecs/flowsentra`.

### Upgrading

1. Take an RDS snapshot.
2. Push the new images to ECR.
3. Register new task definition revisions that point to the new tag.
4. Update the API, frontend and worker services to the new revisions.

To apply schema changes after an upgrade, connect to EC2 as in Step 5 and run the new SQL file with `psql`.

## How do I...?

### Update a service to a new image version

ECS replaces the running container with a new one when you point the service at a new task definition revision.

1. Push the new image to ECR with a new tag.
2. Register a new revision of the task definition with the new image tag.
3. Update the service:

```bash
aws ecs update-service --cluster flowsentra-cluster \
  --service flowsentra-api-service \
  --task-definition flowsentra-api:REVISION --region REGION
```

Repeat for each service that uses the backend image, or for the frontend when its image changes.

### Add a new worker

1. Copy the task definition of an existing worker.
2. Change the family, the container name, the stream prefix, and the `command` to the new worker name.
3. Create a service from it with desired count 1 and no load balancer.

The worker name must be one the backend image supports. Ask the developer for the exact `command`.

### Change the ECS host instance type

Change the instance type in the launch template of the cluster's Auto Scaling group, then replace the instances one at a time. The images must match the new CPU type: `t4g` needs arm64 images and `t3` needs amd64 images.

### Change the number of ECS hosts

The cluster runs four hosts. To change that, edit the minimum, maximum and desired capacity of the cluster's Auto Scaling group. Adding hosts does not add workers: every worker stays at one copy.

### See the container logs

Every container writes to the CloudWatch Logs group `/ecs/flowsentra`, with one stream per service. Open it in the CloudWatch console, or run:

```bash
aws logs tail /ecs/flowsentra --follow --region REGION
aws logs tail /ecs/flowsentra --log-stream-name-prefix incident --since 1h --region REGION
```

To change how long logs are kept, edit the retention setting of the log group.

### Connect to an ECS host

Use AWS Systems Manager, so you do not need SSH keys:

```bash
aws ssm start-session --target i-INSTANCEID --region REGION
sudo su
docker ps
docker logs CONTAINER_ID --tail 50
```

### Deploy another environment (for example test)

Repeat the AWS ECS steps with new names, such as `flowsentra-test-cluster`, a separate RDS database and a separate domain. Do not share one RDS database between two environments.

## Helm

Helm for Kubernetes is not available yet. A Helm chart is planned for a future release. This section will be updated when it is published.

## Background Workers

FlowSentra starts 10 workers from the backend image: `dag_details_worker`, `dag_job_runs_worker`, `dag_task_instances_worker`, `tableau_metadata_worker`, `tableau_runs_worker`, `lineage_worker`, `workbook_lineage_sla_worker`, `workbook_chain_sla_worker`, `incident_worker` and `maintenance_window_worker`.

Every worker is a singleton. Never run more than one copy of any worker, since duplicates process the same jobs and send duplicate emails.

## Configuration

Required environment variables:

- `DOCKERHUB_ORG` set to your Docker Hub organization. Used by Docker Compose to build the image names.
- `POSTGRES_PASSWORD` password of the `postgres` database user.
- `L1_APP_PASSWORD` password for the `l1` application database role, created by `db-init`.
- `ADMIN_EMAIL` email of the first admin user.
- `ADMIN_PASSWORD` password of the first admin user.
- `JWT_SECRET_KEY` secret used to sign access tokens. Generate it with `openssl rand -hex 32`.
- `JWT_REFRESH_SECRET_KEY` secret used to sign refresh tokens. Generate it with `openssl rand -hex 32`.

Optional environment variables:

| Variable | Default | Description |
| --- | --- | --- |
| `BACKEND_TAG` | `backend` | Backend image tag. Set a specific tag to pin a build. |
| `FRONTEND_TAG` | `frontend` | Frontend image tag. Set a specific tag to pin a build. |
| `BACKEND_IMAGE` | `flowsentra` | Backend image repository name. |
| `FRONTEND_IMAGE` | `flowsentra` | Frontend image repository name. |
| `SITE_ADDRESS` | `:80` | Domain for Caddy. Set to `your.domain` to enable HTTPS. |
| `DRY_RUN` | `false` for the API and the dag runs, incident and maintenance window workers | Without `false`, the app defaults to dry run and discards actions such as the 30-minute maintenance warning email. |
| `API_BASE_URL` | `http://api:8000` | Address the two Tableau workers use to reach the API. On ECS, set it to the public URL. |
| `GIT_REPO_PATH` | unset | Path to a DAG repository checkout for git-based lineage in `lineage_worker`. Mount the repository read-only. |

The database connection variables (`L1_DB_URL`, `POSTGRES_DB`, `POSTGRES_HOST`, `PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`) are set by `docker-compose.prod.yml`. On ECS, set them in each task definition with the RDS endpoint instead of `postgres`.
