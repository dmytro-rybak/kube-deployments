# Kube-Deployments

A learning project focused on working with **Kubernetes** and **Argo CD**. The goal is to deploy a full-stack application using the [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template), which includes a **React** frontend, **FastAPI** backend, and **PostgreSQL** database.

The deployment will follow a **GitOps** approach using **Argo CD**, with two separate environments managed via **Kubernetes namespaces**. Additionally, the project integrates with **AWS Secrets Manager** for secure configuration management and **AWS ECR** for container image storage. 

**GitHub Actions** is used to automate the CI/CD pipeline, specifically for building container images, pushing them to AWS ECR, and updating image tags in Helm charts.

## Prerequisites

Before you begin, ensure you have the following tools installed on your machine:

- **Docker**: Required for building and running container images. Follow the installation instructions for your platform:
  - [Docker installation guide](https://docs.docker.com/get-docker/)

- **Kind** (Kubernetes in Docker): Used to create local Kubernetes clusters using Docker. Install Kind by following the official documentation:
  - [Kind installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/)

- **kubectl**: The command-line tool for interacting with your Kubernetes cluster. Install it using the official Kubernetes documentation:
  - [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

- **Helm**: A package manager for Kubernetes that simplifies the deployment of applications. Install Helm by following the instructions here:
  - [Helm installation guide](https://helm.sh/docs/intro/install/)

- Also you need an **AWS account** which is required to set up **AWS Secrets Manager** and **AWS ECR** for secure configuration management and container image storage.

> [!IMPORTANT]
> Make sure each of these tools is installed and properly configured before proceeding with the setup of **kube-deployments**.

## Links

For proper configuration of resources, refer to the official documentation linked below.

- [Kubernetes docs](https://kubernetes.io/docs/home/)
- [KinD docs](https://kind.sigs.k8s.io/)
- [Helm docs](https://helm.sh/docs/)
- [Argo CD docs](https://argo-cd.readthedocs.io/en/stable/)
- [External Secrets docs](https://external-secrets.io/latest/)
- [GitHub Actions docs](https://docs.github.com/en/actions)

<hr>

- [Argo CD Helm chart](https://artifacthub.io/packages/helm/argo/argo-cd)
- [Nginx Ingress Controller Helm chart](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx)
- [PostgreSQL Helm chart](https://artifacthub.io/packages/helm/bitnami/postgresql)
- [External Secrets Helm chart](https://artifacthub.io/packages/helm/external-secrets-operator/external-secrets)
- [Reloader Helm chart](https://artifacthub.io/packages/helm/stakater/reloader)

## Cluster Setup

To set up your Kubernetes cluster with **kind** and configure the necessary components, follow the steps below:

### Step 1: Create the Kind Cluster

The **kind** configuration is already prepared with 1 control-plane node and 2 worker nodes. To create the cluster, run:

```bash
kind create cluster --config kind-config.yaml
```

This command will create a local Kubernetes cluster using the `kind-config.yaml` configuration file.

### Step 2: Install the Argo CD via Helm chart

This command adds the official Argo Helm chart repository to your local Helm configuration:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

This command deploys Argo CD into the `argocd` namespace using the specified Helm chart version.

```bash
helm install argocd argo/argo-cd --version 7.8.2 -n argocd --create-namespace --set dex.enabled=false --set notifications.enabled=false
```

After installing Argo CD, you need to retrieve the initial admin password and access the Argo CD web UI.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

To access the Argo CD web interface locally, you need to forward a port from your local machine to the Argo CD server:

```bash
kubectl port-forward service/argocd-server -n argocd 8080:443
```

Now, you can open `localhost:8080` in your browser and log in using:

- Username: `admin`
- Password: (The one retrieved in the previous command)

### Step 3: Create a Private GitHub Repository for Helm charts

### Step 4: Create a GitHub App for Argo CD

Create a **GitHub App** to allow Argo CD to access this private repository.

1. Go to [GitHub Developer Settings](https://github.com/settings/apps) and click **New GitHub App**.

2. Configure the GitHub App for Argo CD:
   - **GitHub App Name**: Any valid name.
   - **Description**: "Allows Argo CD to access a private repository with Helm charts."
   - **Homepage URL**: In our case it can be any valid URL, for exmaple: https://argo-cd.readthedocs.io.
   - **Expire user authorization tokens**: **Disabled**.
   - **Webhook**: **Disabled**.

3. Set **Repository Permissions** for the GitHub App:
   - **Contents**: `Read-only`
   - **Metadata**: `Read-only`

4. Click **Create GitHub App**.

5. Also you need to generate a Private Key for the app. Scroll down and click **Generate a private key** (After that it will be downloaded to your machine).

### Step 5: Install the GitHub App

Now, install the GitHub App for your Helm charts repository.

1. In your newly created GitHub App settings, navigate to **Install App**.
2. Click **Install** and select only the helmcharts repository you created in the Step 3.
3. Confirm the installation.

### Step 6: Connect the GitHub App to Argo CD

1. Open the **Argo CD UI**.
2. Navigate to **Repositories** → **Connect Repo**.
3. Choose **GitHub App** as the connection method.
4. Fill in the required fields:
   - **Type**: `GitHub`
   - **Project**: `default`
   - **Repository URL**: *(Copy your Helm charts repo URL)*
   - **GitHub App ID**: *(Find it in your GitHub App settings)*
   - **GitHub App Installation ID**: *(Find it in the search bar on the Installed Apps page)*
   - **GitHub App Private Key**: *(Upload the private key downloaded when creating the app)*
5. Click **Connect**.
6. Verify that the repository is connected successfully

## AWS Configuring

> [!WARNING]
> **Important:** It is your responsibility to monitor the costs of your AWS services and delete any unused resources to avoid unexpected charges. We are using **AWS Secrets Manager** and **AWS ECR**, which are not free by default. However, in our case, we will take advantage of AWS's free tier benefits:
  >- **AWS Secrets Manager** provides a **free plan for the first 30 days** from the creation of your first secret.
  >- **AWS ECR** offers **50 GB per month of always-free storage** for public repositories.

### Step 1: Create AWS ECR Public Repositories

In real-world projects, we typically configure **private** Amazon Elastic Container Registry (ECR) repositories to ensure security and access control. However, for this setup, we will use **public** repositories to simplify the process of pulling images for our **local Kubernetes cluster**.

> [!NOTE]
> In production, always use **private repositories** with proper authentication and permissions.

> [!IMPORTANT]
These repositories in our case should be created **only in the `us-east-1` region** because it will be necessary for further GitHub Actions configuration.

Go to **AWS ECR** in the AWS Console and create two **public** repositories named `backend` and `frontend` in the **us-east-1** region. 

Once these repositories are created, they will be used for storing and pulling container images for deployment in our local Kubernetes cluster.

### Step 2: Connect your AWS account with the GitHub repository

To securely connect GitHub Actions with AWS, we will use **OpenID Connect (OIDC)** and assume an AWS IAM role.

1. Go to AWS IAM and create a new OpenID Connect (OIDC) identity provider
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
2. Assign role to the identity provider
   - Trusted entity type: `Web identity`
   - Identity provider: `token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
   - GitHub organization: `<Your GitHub username>`
   - GitHub repository: `<Your GitHub repository name>`
   - Permissions policies: `AmazonElasticContainerRegistryPublicPowerUser`
3. Create GitHub Actions secrets in the GitHub repository settings
   - AWS_REGION: `us-east-1`
   - AWS_ECR_ROLE_TO_ASSUME: `<Your IAM role ARN>`
   - AWS_ECR_REGISTRY_ALIAS: `<Your ECR public registry default alias>`

### Step 3: Create secrts in AWS Secrets Manager

To securely store sensitive information, we will create secrets in **AWS Secrets Manager** for both **dev** and **prod** environments.

> [!WARNING]
> The credentials provided below are not real and are intended solely for testing purposes.
> NEVER push actual credentials (API keys, passwords, tokens, etc.) to GitHub, even in private repositories.

Go to **AWS Secrets Manager** in the AWS Console and create 2 secrets: 
  - `myapp/prod/backend`
    - PROJECT_NAME: `kube-deployments`
    - SECRET_KEY: `verysecretprod`
    - FIRST_SUPERUSER: `prod-admin@example.com`
    - FIRST_SUPERUSER_PASSWORD: `prodpassword`
    - POSTGRES_SERVER: `postgres-prod-postgresql.prod.svc.cluster.local`
    - POSTGRES_PORT: `5432`
    - POSTGRES_USER: `postgres`
    - POSTGRES_PASSWORD: `0HOazoRK9s`

  - `myapp/dev/backend`
    - PROJECT_NAME: `kube-deployments`
    - SECRET_KEY: `verysecretdev`
    - FIRST_SUPERUSER: `dev-admin@example.com`
    - FIRST_SUPERUSER_PASSWORD: `devpassword`
    - POSTGRES_SERVER: `postgres-dev-postgresql.dev.svc.cluster.local`
    - POSTGRES_PORT: `5432`
    - POSTGRES_USER: `postgres`
    - POSTGRES_PASSWORD: `QclrKKcUpJ`

### Step 4: Create IAM user and Secutiry Credentials

This user will be used to securely manage and retrieve secrets in AWS Secrets Manager.

Go to **AWS IAM** in the AWS Console and create a new IAM user without Console Access and with `SecretsManagerReadWrite` permissions. Generate a new pair of security credentials and download or copy the **Access Key ID** and **Secret Access Key** as they will be needed later.
