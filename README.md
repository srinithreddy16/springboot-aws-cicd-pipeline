# Spring Boot AWS CI/CD Pipeline

A minimal Spring Boot application that demonstrates end-to-end CI/CD: build with Maven, package with Docker, publish to Docker Hub, and **deploy automatically to an AWS EC2 instance** on every push to `main`.

## What it does

| Layer | Technology |
|-------|------------|
| App | Spring Boot 3, Java 21 |
| Build | Maven (via `mvnw` wrapper) |
| Container | Docker (`eclipse-temurin:21`) |
| CI/CD | GitHub Actions |
| Registry | Docker Hub |
| Runtime | AWS EC2 (Docker) |

**API:** `GET /` returns:

```text
Hello from CI/CD pipeline
```

## Architecture

```mermaid
flowchart LR
  Dev[Developer push to main] --> GHA[GitHub Actions]
  GHA --> Maven[Maven build JAR]
  Maven --> Docker[Build and push image]
  Docker --> Hub[Docker Hub]
  GHA --> SSH[SSH to EC2]
  SSH --> Pull[docker pull latest]
  Pull --> Run[docker run on port 8080]
  User[Browser] --> Run
```

1. **GitHub Actions** checks out code, installs Java 21, and runs `./mvnw clean package`.
2. The workflow builds a Docker image and **pushes** it to Docker Hub as `/<your-username>/springboot-aws-cicd-pipeline:latest`.
3. The workflow **SSHs into EC2**, pulls the new image, stops/removes the old container, and starts a fresh one on **port 8080**.

## Project structure

```text
.
├── .github/workflows/ci-cd.yml   # CI/CD + EC2 deploy
├── src/main/java/...             # Spring Boot app
├── Dockerfile                    # Copies target/*.jar into image
├── pom.xml
├── mvnw / mvnw.cmd               # Maven wrapper (used in CI)
└── README.md
```

## Run locally

**Prerequisites:** Java 21, or only Docker after a build.

```bash
# Build
./mvnw clean package -DskipTests

# Run with Maven
./mvnw spring-boot:run

# Or run with Docker
docker build -t springboot-aws-cicd-pipeline .
docker run -p 8080:8080 springboot-aws-cicd-pipeline
```

Open [http://localhost:8080](http://localhost:8080).

## AWS EC2 setup (one-time)

Prepare an EC2 instance before the pipeline can deploy.

### 1. Launch instance

- **AMI:** Ubuntu 22.04 or Amazon Linux 2023 (examples below use `ubuntu`).
- **Instance type:** `t2.micro` or similar is enough for this demo.
- **Key pair:** Create/download a `.pem` file (e.g. `demo-keypair.pem`). **Never commit `.pem` files** — they are listed in `.gitignore`.

### 2. Security group

| Type | Port | Source | Purpose |
|------|------|--------|---------|
| SSH | 22 | Your IP (or GitHub Actions IPs if restricting) | SSH deploy |
| Custom TCP | 8080 | `0.0.0.0/0` (demo) or your IP | HTTP to the app |

For production, restrict **8080** and **22** to known IPs.

### 3. Install Docker on EC2

SSH into the instance (replace key and host):

```bash
ssh -i demo-keypair.pem ubuntu@<EC2_PUBLIC_IP>
```

**Ubuntu:**

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker ubuntu
# Log out and back in so group membership applies
```

**Amazon Linux 2023:**

```bash
sudo yum update -y
sudo yum install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

Verify:

```bash
docker --version
```

### 4. Manual smoke test (optional)

On EC2, after your first successful pipeline run:

```bash
docker pull <dockerhub-username>/springboot-aws-cicd-pipeline:latest
docker run -d --name spring-demo -p 8080:8080 <dockerhub-username>/springboot-aws-cicd-pipeline:latest
curl http://localhost:8080
```

From your laptop: `http://<EC2_PUBLIC_IP>:8080`.

## GitHub Actions & EC2 deployment

Workflow file: [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml).

**Triggers:**

- Push to `main`
- Manual run: **Actions → CI/CD Pipeline → Run workflow**

**Deploy step (EC2):** uses [`appleboy/ssh-action`](https://github.com/appleboy/ssh-action) to run on the server:

```bash
docker stop spring-demo || true
docker rm spring-demo || true
docker pull <username>/springboot-aws-cicd-pipeline:latest
docker run -d --name spring-demo --restart unless-stopped -p 8080:8080 <username>/springboot-aws-cicd-pipeline:latest
```

### Required GitHub secrets

Add under **Settings → Secrets and variables → Actions** (repository **secrets**, not variables):

| Secret | Description |
|--------|-------------|
| `DOCKER_USERNAME` | Docker Hub username (also accepts `DOCKERHUB_USERNAME` or `DOCKER_HUB_USERNAME`) |
| `DOCKERHUB_TOKEN` | Docker Hub [access token](https://hub.docker.com/settings/security) (not your account password) |
| `EC2_HOST` | EC2 public IPv4 or DNS (e.g. `ec2-xx-xx-xx-xx.compute.amazonaws.com`) |
| `EC2_USER` | SSH user: `ubuntu` (Ubuntu AMI) or `ec2-user` (Amazon Linux) |
| `EC2_SSH_KEY` | Full contents of your `.pem` private key |

**`EC2_SSH_KEY`:** open the `.pem` file in a text editor and paste everything, including:

```text
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

### Enable Actions

**Settings → Actions → General** → allow GitHub Actions for this repository.

### Verify a deployment

1. Push to `main` or run the workflow manually.
2. **Actions** tab → open the latest run → confirm **SSH and Deploy to EC2** is green.
3. Visit `http://<EC2_PUBLIC_IP>:8080`.

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| `Username required` (Docker login) | Missing or misnamed `DOCKER_USERNAME` secret; use **Secrets**, not **Variables** |
| SSH step fails | Wrong `EC2_HOST`, `EC2_USER`, or `EC2_SSH_KEY`; security group blocks port 22 |
| Pull fails on EC2 | Image name mismatch; private repo without `docker login` on EC2 |
| App unreachable | Security group missing inbound **8080**; container not running (`docker ps` on EC2) |
| Maven works locally but not in CI | Use `./mvnw` in the repo; do not commit `target/` |

On EC2, inspect the running app:

```bash
docker ps
docker logs spring-demo
```

## Security notes

- Do **not** commit `.pem` files, passwords, or tokens (see `.gitignore`).
- Rotate keys if a private key was ever committed or shared.
- Prefer Docker Hub **access tokens** with minimal scope.
- Restrict security group rules in non-demo environments.

## License

Demo / educational project — use and adapt as needed.
