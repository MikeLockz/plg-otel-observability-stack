# Deploying the Observability Stack with a Self-Hosted GitHub Runner

This guide details how to automate the deployment of the observability stack to a dedicated LXC container using a self-hosted GitHub Actions runner.

## 1. Preparing the LXC Container as a GitHub Runner (Automated Script)

This script automates the entire GitHub Runner setup inside your LXC container. Run this script directly on your **Proxmox host shell**.

### Step 1: Get a Runner Registration Token

1.  Navigate to your GitHub repository's settings: `Settings` > `Actions` > `Runners`.
2.  Click **New self-hosted runner**.
3.  You will see a section with a **registration token**. It will look something like `A4A...CQ`. Copy this token. **It is temporary and only valid for a short time.**

### Step 2: Run the Automation Script

Copy the script below, paste it into your Proxmox host's shell, and **update the variables at the top** before running it.

```bash
#!/bin/bash

# ==> PLEASE CONFIGURE THESE VARIABLES <==
LXID=900                                  # The ID of your LXC container.
GH_RUNNER_TOKEN="AAGCU2VBW2P7ZQV37BDHULTJEYFAA"         # Paste the token from GitHub here.
GH_OWNER="MikeLockz"                      # Your GitHub username or organization.
GH_REPO="plg-otel-observability-stack"    # Your GitHub repository name.
# ==> END OF CONFIGURATION <==

# --- Runner Details ---
RUNNER_USER="github-runner"
RUNNER_DIR="/home/$RUNNER_USER/actions-runner"
RUNNER_VERSION="2.311.0" # You can update this to the latest version found on the GitHub runners page.


echo "--- [1/7] Installing dependencies (sudo, curl) in LXC $LXID ---"
pct exec $LXID -- bash -c "apt-get update && apt-get install -y sudo curl"

echo "--- [2/7] Creating user '$RUNNER_USER' ---"
pct exec $LXID -- bash -c "adduser --disabled-password --gecos '' $RUNNER_USER && usermod -aG docker $RUNNER_USER"

echo "--- [3/7] Configuring passwordless sudo for '$RUNNER_USER' ---"
pct exec $LXID -- bash -c "echo '$RUNNER_USER ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/github-runner"

echo "--- [4/7] Creating runner directory ---"
pct exec $LXID -- bash -c "mkdir -p $RUNNER_DIR && chown $RUNNER_USER:$RUNNER_USER $RUNNER_DIR"

echo "--- [5/7] Downloading and extracting GitHub Runner ---"
pct exec $LXID -- sudo -u $RUNNER_USER -s /bin/bash -c " \
  cd $RUNNER_DIR && \
  curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz && \
  tar xzf ./actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
"

echo "--- [6/7] Configuring runner ---"
pct exec $LXID -- sudo -u $RUNNER_USER -s /bin/bash -c " \
  cd $RUNNER_DIR && \
  ./config.sh --unattended --url https://github.com/$GH_OWNER/$GH_REPO --token $GH_RUNNER_TOKEN --name lxc-runner-$LXID --work _work \
"

echo "--- [7/7] Installing and starting runner service ---"
# Note: The 'install' command requires the username as an argument if not run interactively.
pct exec $LXID -- bash -c "cd $RUNNER_DIR && ./svc.sh install $RUNNER_USER && ./svc.sh start"

echo "---"
echo "✅ Setup complete! You can verify the service status by running:"
echo "pct exec $LXID -- bash -c 'cd $RUNNER_DIR && ./svc.sh status'"
echo "Then, check your GitHub repository's 'Runners' settings to see the new runner with an 'Idle' status."
```

### Script Explanation
*   **New:** It now installs `sudo` and `curl` as the first step.
*   It creates a dedicated user (`github-runner`) inside the LXC.
*   **New:** It creates a file in `/etc/sudoers.d/` to grant the `github-runner` user passwordless `sudo` privileges, which is required for non-interactive scripting.
*   It downloads and extracts the official GitHub Runner software.
*   It uses the `--unattended` flag to automatically configure the runner.
*   It installs and starts the runner as a `systemd` service.


## 2. Configuring GitHub Repository Settings

No special repository secrets are needed for this basic deployment. The runner will have access to the repository code by default. Ensure that your repository's Actions settings (`Settings` > `Actions` > `General`) are configured to allow local and third-party actions, as needed.

## 3. GitHub Actions Workflow for Deployment

Create a workflow file in your repository at `.github/workflows/deploy.yml`. This workflow will trigger on a push to the `main` branch, check out the code, and deploy the stack using Docker Compose.

### Create the Workflow File

1.  Create the directory and file:
    ```sh
    mkdir -p .github/workflows
    touch .github/workflows/deploy.yml
    ```

2.  Add the following content to `deploy.yml`:

    ```yaml
    name: Deploy Observability Stack

    on:
      push:
        branches:
          - main

    jobs:
      deploy:
        name: Deploy to LXC
        runs-on: self-hosted
        steps:
          - name: Checkout Repository
            uses: actions/checkout@v4

          - name: Deploy Docker Compose Stack
            run: |
              echo "Starting deployment..."
              docker-compose up -d --build
              echo "Deployment complete."

          - name: Prune Old Docker Images
            if: always()
            run: |
              echo "Cleaning up old Docker images..."
              docker image prune -f
    ```

### Workflow Explanation

*   **`name`**: The name of the workflow.
*   **`on`**: This workflow triggers on any `push` to the `main` branch.
*   **`jobs`**: Defines the work to be done.
    *   **`deploy`**: The name of the job.
    *   **`runs-on: self-hosted`**: This is the crucial part. It tells GitHub to execute this job on a runner that is self-hosted, which will be the LXC container you configured.
    *   **`steps`**:
        *   **`Checkout Repository`**: This uses the standard `actions/checkout` action to pull your repository's code into the runner's workspace.
        *   **`Deploy Docker Compose Stack`**: This step runs `docker-compose up -d --build`. The `--build` flag ensures that any changes to your configuration files are reflected in the running containers.
        *   **`Prune Old Docker Images`**: This optional but recommended step cleans up old, unused Docker images to save disk space. It runs `if: always()` to ensure cleanup happens even if the deployment step fails.

With this setup, every time you push a change to the `main` branch, the GitHub runner in your LXC will automatically update and redeploy the observability stack.
