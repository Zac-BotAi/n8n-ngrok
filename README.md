# n8n with Docker for Local Development and Deployment

This repository provides a Docker Compose setup for running n8n, a powerful workflow automation tool. It's configured for flexible local development (with or without [ngrok](https://ngrok.com/) for public tunneling) and for deployment to [Render](https://render.com/).

## Prerequisites

- Docker: [Get Docker](https://docs.docker.com/get-docker/)
- Docker Compose: [Install Docker Compose](https://docs.docker.com/compose/install/)
- For ngrok tunneling (optional):
    - An ngrok account: [Sign up at ngrok.com](https://ngrok.com/)
    - Your ngrok authtoken.
- For Render deployment (optional):
    - A Render account.

## Setup

1.  **Clone the Repository**
    ```bash
    git clone <your-repository-url> # Replace with the actual URL
    ```

2.  **Environment Variables (`.env` file)**

    Create a `.env` file in the root of the project by copying `.env.example` (if one exists) or creating it manually. This file is used by `docker-compose` for local configuration. For Render, you will set these in the service's environment settings on the Render dashboard.

    **Example `.env` content:**
    ```sh
    # === General n8n Configuration ===
    # Timezone for n8n (e.g., America/New_York, Europe/Berlin)
    TIMEZONE=America/New_York

    # Add any other n8n specific environment variables you need.
    # Refer to the n8n documentation: https://docs.n8n.io/hosting/configuration/environment-variables/
    # Example for a custom database (if not using default SQLite):
    # DB_TYPE=postgresdb
    # DB_POSTGRESDB_HOST=your_db_host
    # ...

    # === Ngrok Configuration (only needed if using the ngrok profile) ===
    # Your ngrok authentication token
    NGROK_TOKEN= # Replace with your ngrok authtoken

    # Your ngrok domain (e.g., https://your-subdomain.ngrok-free.app)
    # This is the URL n8n will use for webhooks when ngrok is active.
    URL= # Replace with your ngrok public URL
    ```

    **Important:**
    - Only set `NGROK_TOKEN` and `URL` if you intend to use the ngrok tunneling feature locally.
    - For Render deployment or local development *without* ngrok, ensure `URL` is commented out or empty. n8n will then attempt to auto-detect its public URL or use the one provided by Render.

## Running the Application

This setup uses Docker Compose profiles to manage different configurations.

### A. Local Development (n8n only, no ngrok tunnel)

This is suitable for most local development and testing where you don't need a public URL for n8n.

1.  **Start n8n:**
    ```bash
    docker-compose up -d
    ```
    This command starts services in the default profile (which includes `n8n`). The `-d` flag runs containers in detached mode.

2.  **Accessing n8n Locally:**
    Once the `n8n` container is running, it will typically be accessible at `http://localhost:5678`.

3.  **Stopping Local Containers:**
    ```bash
    docker-compose down
    ```

### B. Local Development (n8n with ngrok Tunnel)

This setup runs n8n and exposes it to the internet using an ngrok tunnel. This is useful for testing webhooks from external services.

1.  **Ensure `.env` is Configured:**
    - Set your `NGROK_TOKEN` in the `.env` file.
    - Set your desired public `URL` (e.g., `https://your-name.ngrok-free.app`) in the `.env` file. This `URL` will be used by n8n to configure its webhooks.

2.  **Start n8n and ngrok:**
    ```bash
    docker-compose --profile ngrok up -d
    ```
    This command activates the `ngrok` profile, which starts both the `n8n` and `ngrok` services.

3.  **Accessing n8n via ngrok:**
    - n8n will be accessible via the `URL` you specified in your `.env` file.
    - You can also check the ngrok dashboard or its local API (usually `http://127.0.0.1:4040`) to see the tunnel status and public URL.

4.  **Stopping Local Containers:**
    ```bash
    docker-compose --profile ngrok down
    ```

### C. Deployment to Render

This project can be deployed to Render. The `render.yaml` file in this repository helps define the service for Render.

1.  **Using `render.yaml` (Recommended "Infrastructure as Code"):**
    - Commit the `render.yaml` file to your repository.
    - In the Render Dashboard, create a new "Blueprint Instance".
    - Connect your Git repository. Render should detect and use the `render.yaml` file.
    - Review the settings, especially:
        - **Environment Variables:** Set `TIMEZONE` and any other required n8n variables (like database credentials if not using SQLite, or `N8N_ENCRYPTION_KEY`) in a Render Environment Group or directly on the service. **Do not set `URL` or `NGROK_TOKEN` for Render deployment.**
        - **Persistent Disk:** The `render.yaml` defines a disk for `/home/node/.n8n`. Ensure this is correctly provisioned.

2.  **Manual Setup on Render (Alternative):**
    - Connect your Git repository to Render.
    - Create a new "Web Service".
    - Runtime: Docker.
    - Repository: Point to this repository.
    - **Environment Variables:** Configure as mentioned above. n8n is set up to use the `PORT` variable provided by Render.
    - **Persistent Disk:** Create and attach a Render Disk, mounting it at `/home/node/.n8n`.
    - **Health Check Path:** `/healthz`.

3.  **Accessing n8n on Render:**
    Render provides a public URL (e.g., `your-service-name.onrender.com`).

## `render.yaml` Configuration

The `render.yaml` file defines the n8n service for Render:
```yaml
services:
  - type: web
    name: n8n
    env: docker
    image:
      url: docker.n8n.io/n8nio/n8n:latest
    ports:
      - port: 5678 # n8n's internal default port; Render uses N8N_PORT=${PORT:-5678}
    envVars:
      - key: TIMEZONE
        value: America/New_York # TODO: Change as needed in Render dashboard
      - key: GENERIC_TIMEZONE
        value: America/New_York # TODO: Change as needed in Render dashboard
      # - key: N8N_ENCRYPTION_KEY # Recommended for security
      #   generateValue: true # Let Render generate a secure key
      # - key: WEBHOOK_URL
      #   fromService:
      #     type: web
      #     name: n8n # This service's name
      #     property: url
    disk:
      name: n8n-data
      mountPath: /home/node/.n8n
      sizeGB: 10 # Adjust as needed
    healthCheckPath: /healthz
    # autoDeploy: true # Optional
```

**Note on `WEBHOOK_URL`:**
-   **For Render or Local (no ngrok):** Leave the `URL` variable in your `.env` file empty or commented out. n8n's `WEBHOOK_URL=${URL}` will result in an empty value, prompting n8n to auto-detect its URL. Render can also supply its URL via `RENDER_EXTERNAL_URL` if needed (set this in Render's env vars for n8n).
-   **For Local with ngrok:** Set the `URL` in your `.env` file to your public ngrok domain. n8n will use this for webhooks.

## Stopping Applications
- **Local Docker:** `docker-compose down` or `docker-compose --profile ngrok down`.
- **Render:** Manage via the Render dashboard.
```
