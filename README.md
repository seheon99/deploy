# Docker Service CD

A reusable GitHub Action to manage **service-level Docker Continuous Delivery**.

This action builds and signs Docker images, then deploys running services
remotely via **Docker Compose over SSH**, using **image digests** for safe and
reproducible rollouts.


## ✨ What this Action Does

This action encapsulates a complete Docker-based CD workflow:

1. **Build Docker images** using Buildx with cache support
2. **Push images to a registry** (GHCR)
3. **Sign images with Cosign** for supply-chain security
4. **Update running services** on a remote server via SSH
5. **Deploy with image digests**, not mutable tags

It is designed to be reused across multiple services and repositories.


## 🔧 Key Features

- 🔁 **Digest-based deployment**  
  Ensures reproducible and rollback-friendly deployments.

- 🔐 **Image signing with Cosign**  
  Adds a security layer to published images.

- 🚀 **Remote deployment via SSH + Docker Compose**  
  No Kubernetes required. Simple, explicit, and transparent.

- 🧪 **Dry-run mode**  
  Run the entire pipeline without pushing or deploying artifacts.

- 📦 **Service-oriented design**  
  One action, many services.


## 🗂 Expected docker-compose.yml & .env Structure

This action assumes a **simple and explicit Docker Compose setup**.
To enable digest-based deployment, your project must follow the structure below.


### 1️⃣ docker-compose.yml

Each service image must reference a digest variable defined in `.env`.

```yaml
services:
  portfolio:
    image: ghcr.io/your-org/portfolio@${PORTFOLIO_IMAGE_DIGEST}
    restart: unless-stopped
  ports:
    - "3000:3000"
```

Key points:

* Image tags are **not used directly**
* Each service reads its image digest from `.env`
* Service name must match the `service-name` input


### 2️⃣ .env

For each service, define an image digest variable following this convention:

```env
PORTFOLIO_IMAGE_DIGEST=sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Naming rules:

* Based on `service-name`
* Converted to **UPPER_SNAKE_CASE**
* `-` and spaces are replaced with `_`
* Suffix `_IMAGE_DIGEST` is mandatory

Examples:

* service-name: `portfolio` → `PORTFOLIO_IMAGE_DIGEST`
* service-name: `user-api` → `USER_API_IMAGE_DIGEST`


### 3️⃣ How This Action Uses Them

During deployment, the action:

1. Builds and pushes a new Docker image
2. Resolves the **exact image digest**
3. Updates the corresponding variable in `.env`
4. Runs:

```bash
docker compose up -d --no-deps <service-name>
```

This ensures:

* Only the target service is restarted
* The deployment is deterministic
* Rollbacks are as simple as reverting the digest


## 📌 Usage

### Basic Example

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy service
        uses: your-org/docker-service-cd@v1
        with:
          image-name: ghcr.io/your-org/your-service
          service-name: portfolio
          ssh-host: ${{ secrets.SSH_HOST }}
          ssh-user: ${{ secrets.SSH_USER }}
          ssh-key: ${{ secrets.SSH_KEY }}
```


## ⚙️ Inputs

| Name           | Required | Default | Description                                            |
| -------------- | -------- | ------- | ------------------------------------------------------ |
| `image-name`   | ✅        | —       | Full Docker image reference (e.g. `ghcr.io/user/repo`) |
| `service-name` | ✅        | —       | Docker Compose service name                            |
| `dry-run`      | ❌        | `false` | Run pipeline without pushing or deploying              |
| `ssh-host`     | ✅        | —       | SSH host for deployment                                |
| `ssh-user`     | ✅        | —       | SSH username                                           |
| `ssh-key`      | ✅        | —       | Private SSH key                                        |
| `ssh-port`     | ❌        | `22`    | SSH port                                               |


## 🧪 Dry Run Mode

You can safely validate the pipeline logic without modifying any remote state:

```yaml
with:
  dry-run: "true"
```

In `dry-run` mode:

* Images are built but **not pushed**
* Images are **not signed**
* Deployment over SSH is **skipped**

This is useful for testing PRs or validating changes.


## 🧠 How Deployment Works

On the remote server, the action:

1. Converts `service-name` into an environment-safe variable
   (e.g. `portfolio-api` → `PORTFOLIO_API_IMAGE_DIGEST`)
2. Updates the `.env` file with the new image digest
3. Runs:

```bash
docker compose up -d --no-deps <service>
```

This allows:

* Minimal restarts
* Service-scoped deployments
* Easy rollback by reverting the digest
