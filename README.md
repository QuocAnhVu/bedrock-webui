# Local Open WebUI with AWS Bedrock via Bedrock Access Gateway

This repository provides a streamlined setup to run [Open WebUI](https://github.com/open-webui/open-webui) locally using **Podman Compose** (default). It connects to [AWS Bedrock](https://aws.amazon.com/bedrock/) foundation models via the [Bedrock Access Gateway](https://github.com/aws-samples/bedrock-access-gateway), which translates OpenAI API calls to Bedrock.

This setup is ideal for local development, testing, and personal use of AWS Bedrock models with a user-friendly chat interface, while leveraging the security benefits of rootless Podman (if chosen).

## ✨ Features

* **One-Command Deployment:** Spin up both Open WebUI and Bedrock Access Gateway with a single `podman compose up` command.

* **Flexible Container Engine:** Compatible with both **Podman** (recommended for rootless operation) and **Docker**.

* **Modular Design:** Uses a Git submodule for the Bedrock Access Gateway, making it easy to track upstream updates.

* **Persistent Data:** Open WebUI chat history and settings are preserved across container restarts.

## 🚀 Prerequisites

Before you begin, ensure you have the following:

1.  **AWS Account:** An active AWS account.

2.  **Container Engine (Podman or Docker):** Installed on your system.

    * **Podman:** Refer to the [official Podman installation guide](https://podman.io/docs/installation) for your specific Linux distribution.

    * **Docker:** Refer to the [official Docker installation guide](https://docs.docker.com/engine/install/) for your specific Linux distribution.

3.  **Compose Plugin/Tool:** You will need the `compose` plugin/tool for your chosen container engine.

    * For **Podman**, install `podman-compose`. Refer to the [Podman Compose installation guide](https://github.com/containers/podman-compose#installation) or your distribution's packages (e.g., `sudo dnf install podman-compose` on Fedora).

    * For **Docker**, ensure you have the `Docker Compose plugin` (for Docker Desktop) or `docker-compose` (standalone v1 or plugin v2). Refer to the [Docker Compose documentation](https://docs.docker.com/compose/install/).

4.  **Git:** Installed on your system.

## 🔑 AWS Account & Permissions Setup (Crucial!)

Before deploying the applications, you need to configure your AWS environment to allow access to Bedrock.

1.  **Request Bedrock Model Access:**
    You **must** explicitly enable access to the specific Bedrock foundation models (e.g., Anthropic Claude, Amazon Titan Text, Meta Llama) you intend to use in the AWS region you will configure.

    * Log in to the [AWS Bedrock console](https://console.aws.amazon.com/bedrock/home#/model-access).

    * Navigate to **"Model access"** in the left sidebar.

    * Request and grant access to the desired models for your chosen AWS region. This process usually takes a few minutes.

2.  **Create an AWS IAM User and Policy:**
    You need an IAM user with programmatic access (Access Key ID and Secret Access Key) and the necessary permissions to invoke Bedrock models.

    * **IAM Policy:** Create an IAM policy with the following permissions. This policy grants the necessary actions for the Bedrock Access Gateway to interact with Bedrock, including streaming responses.

        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "BedrockAccess",
                    "Effect": "Allow",
                    "Action": [
                        "bedrock:InvokeModel",
                        "bedrock:InvokeModelWithResponseStream",
                        "bedrock:ListFoundationModels",
                        "bedrock:ListInferenceProfiles"
                    ],
                    "Resource": "*"
                }
            ]
        }
        ```

        **Description for IAM Policy:** Grants permissions to invoke and stream responses from AWS Bedrock foundation models, and to list available models.

    * **Create IAM User:** Create an IAM user and attach the policy you just created to this user.

    * **Generate Access Keys:** Generate an Access Key ID and Secret Access Key for this IAM user. **Save these keys securely immediately; they are only shown once.**

## 📦 Setup Steps

**Note:** All `podman compose` commands below can be replaced with `docker compose` if you are using Docker.

1.  **Clone This Repository:**

    ```bash
    git clone https://github.com/QuocAnhVu/bedrock-webui.git
    cd bedrock-webui
    ```

2.  **Initialize the Bedrock Access Gateway Submodule:**

    ```bash
    git submodule update --init --recursive
    ```

    This will clone the `aws-samples/bedrock-access-gateway` repository into the `bedrock-access-gateway/` subdirectory.

3.  **Configure AWS Credentials in `compose.yaml`:**
    Open `compose.yaml` in your text editor. You need to provide your AWS credentials and desired region to the `bedrock-access-gateway` service.

    ```yaml
    # compose.yaml (in bedrock-webui/)
    version: '3.8'

    services:
      bedrock-access-gateway:
        build:
          context: ./bedrock-access-gateway/src # Path to the submodule's source
          dockerfile: Dockerfile_ecs           # Use Dockerfile_ecs explicitly
        environment:
          # !! REPLACE WITH YOUR ACTUAL AWS CREDENTIALS AND REGION !!
          AWS_ACCESS_KEY_ID: "YOUR_AWS_ACCESS_KEY_ID"
          AWS_SECRET_ACCESS_KEY: "YOUR_AWS_SECRET_ACCESS_KEY"
          AWS_REGION: "your-aws-region"
          DEBUG: "true" # Optional: set to "false" for less verbose logs
        ports:
          - "8000:80"
        networks:
          - webui-bedrock-network
        restart: always

      open-webui:
        image: ghcr.io/open-webui/open-webui:main
        ports:
          - "3000:8080"
        volumes:
          - open-webui-data:/app/backend/data
        networks:
          - webui-bedrock-network
        depends_on:
          - bedrock-access-gateway # Ensures BAG starts before Open WebUI
        restart: always

    networks:
      webui-bedrock-network:
        name: webui-bedrock-network # Explicitly naming it for clarity

    volumes:
      open-webui-data:
    ```

    **Replace `YOUR_AWS_ACCESS_KEY_ID`, `YOUR_AWS_SECRET_ACCESS_KEY`, and `your-aws-region` with actual values.** For production or public repositories, consider setting these as environment variables in your shell (e.g., `export AWS_ACCESS_KEY_ID="xxx"`) before running `podman compose up`, and using `${VAR_NAME}` syntax in `compose.yaml`.

4.  **Build and Run the Containers:**
    Ensure you are in the root directory of this repository (`bedrock-webui/`).

    ```bash
    # Build the Bedrock Access Gateway image (uses Dockerfile_ecs as specified in compose.yaml)
    podman compose build bedrock-access-gateway

    # Start both Open WebUI and Bedrock Access Gateway in detached mode
    podman compose up -d
    ```

    *Allow a minute or two for the images to pull and containers to start.*

5.  **Configure Open WebUI:**

    1.  **Access Open WebUI:** Open your web browser and go to `http://localhost:3000`.

    2.  **Initial Setup:** If it's your first time, create an admin user.

    3.  **Connect to Bedrock Gateway:**

        * Click on your **profile icon** (top right corner).

        * Go to **Admin Panel** -> **Settings** -> **Connections**.

        * Under the **OpenAI API** section, click the **"+" (plus) button** to add a new connection.

        * For **API Base URL**, enter: `http://bedrock-access-gateway:80/api/v1`

            * **Important:** This URL allows the Open WebUI *backend* to communicate with the Bedrock Access Gateway *backend* within the shared container network. Do NOT use `localhost` or port `8000` here.

        * For **API Key**, enter: `bedrock` (the default key for Bedrock Access Gateway).

        * Click the **"Verify Connection"** button. You should see a "Server connection verified" alert.

        * Click **"Save"**.

6.  **Start Using Bedrock Models:**
    Return to the main chat interface in Open WebUI. You should now see the Bedrock models you enabled in your AWS account available in the model dropdown list. Select one and start chatting!

---

## Troubleshooting Common Issues

* **`net::ERR_NAME_NOT_RESOLVED` in browser/GUI:** This means you likely set up a "Direct Connection" in Open WebUI. Ensure you configured an "OpenAPI Connection" (or similar standard connection type) where the API Base URL is `http://bedrock-access-gateway:80/api/v1`.

* **`AccessDeniedException` when calling `ConverseStream` (in Bedrock Access Gateway logs):**

    * **IAM Policy:** Ensure your IAM policy includes `bedrock:InvokeModelWithResponseStream`. (See the [AWS Account & Permissions Setup](#aws-account--permissions-setup-crucial) section above).

    * **Model Access:** Double-check that you have explicitly enabled access to the specific Bedrock models you're trying to use in the [AWS Bedrock console under "Model access"](https://console.aws.amazon.com/bedrock/home#/model-access) for your configured `AWS_REGION`.

    * **Region Mismatch:** Verify `AWS_REGION` in `compose.yaml` matches where models are enabled.

* **`401 Unauthorized` / `403 Forbidden` (in Bedrock Access Gateway logs):**

    * This means Open WebUI reached the gateway, but the API key was rejected. Double-check that the "API Key" in Open WebUI's settings exactly matches the `DEFAULT_API_KEYS` set in Bedrock Access Gateway (which is `bedrock` by default). It is case-sensitive.

* **No Logs from Bedrock Access Gateway when Open WebUI connects:**

    * Ensure `DEBUG: "true"` is set in `compose.yaml` for `bedrock-access-gateway` service.

    * Confirm Open WebUI's "API Base URL" is `http://bedrock-access-gateway:80/api/v1`.

    * Verify internal connectivity by getting a shell into `open-webui` container (`podman exec -it open-webui /bin/bash` or `/bin/sh`) and running:

        ```bash
        python3 -c "import http.client, json; conn = http.client.HTTPConnection('bedrock-access-gateway', 80); conn.request('GET', '/health'); r = conn.getresponse(); print(f'Status: {r.status}, Reason: {r.reason}'); print(r.read().decode()); conn.close()"
        ```

        This *should* produce logs in the Bedrock Access Gateway container.

* **`podman compose` command not found or issues with rootless Podman/Docker:**

    * Ensure your chosen container engine (Podman or Docker) and its `compose` tool are correctly installed.

    * If using Podman: Verify your user's rootless Podman setup: `systemctl --user enable --now podman.socket` and `loginctl enable-linger $(whoami)`.

---

## 🔄 Updating Your Setup

* **Update Open WebUI:**

    ```bash
    podman compose pull open-webui
    podman compose up -d open-webui
    ```

* **Update Bedrock Access Gateway:**
    To update the Bedrock Access Gateway, you need to update the submodule and then rebuild its image:

    ```bash
    git submodule update --remote bedrock-access-gateway
    podman compose build bedrock-access-gateway
    podman compose up -d bedrock-access-gateway
    ```

---

## ⚠️ Security Notes and Customization

* **AWS API Keys:** Your AWS Access Key ID and Secret Access Key grant access to your AWS account. **Keep them extremely secure.** Avoid committing them directly to version control in public repositories.

* **Customizing Bedrock Access Gateway API Key:**
    By default, the Bedrock Access Gateway uses `"bedrock"` as its API key. **For any production use or when exposing your gateway to a network, you MUST change this default key to a strong, unique value.**

    To customize the API key:

    1.  Navigate into the submodule's source directory:
        ```bash
        cd bedrock-access-gateway/src/api
        ```
    2.  Open the `setting.py` file in a text editor:
        ```bash
        nano setting.py # or your preferred editor
        ```
    3.  Locate the line `DEFAULT_API_KEYS = "bedrock"` and change `"bedrock"` to your desired strong, secret key:
        ```python
        # setting.py
        DEFAULT_API_KEYS = 'your-new-strong-secret-key-here' # Replace with your desired key
        ```
    4.  Save the `setting.py` file.
    5.  Navigate back to the root of your main repository:
        ```bash
        cd ../../.. # From bedrock-access-gateway/src/api
        ```
    6.  **Rebuild the Bedrock Access Gateway image** to incorporate your changes:
        ```bash
        podman compose build bedrock-access-gateway
        ```
    7.  **Recreate the container** using the newly built image:
        ```bash
        podman compose up -d --force-recreate bedrock-access-gateway
        ```
    8.  Finally, update the "API Key" in Open WebUI's connection settings to your new key.

* **Rootless Containers:** Running with rootless Podman significantly enhances security by preventing containers from gaining root privileges on your host system.

## 🐳 Using Podman or Docker

This setup is compatible with both Podman and Docker. While the instructions assume `podman compose` as the default, you can generally replace `podman` with `docker` in any command (e.g., `docker compose up -d`, `docker ps`).

* **Command Interchangeability:** Most container commands are highly similar. You can generally replace `podman` with `docker` (e.g., `podman run` vs. `docker run`, `podman ps` vs. `docker ps`).

* **Compose Commands:** `podman compose` functions identically to `docker compose` when working with `compose.yaml` files.

* **Coexistence:** Podman and Docker can typically coexist on the same system without interference. You can choose whichever tool you prefer for your container workflows.
