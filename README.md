# Kube-Deployments

A learning project focused on working with **Kubernetes** and **Argo CD**. The goal is to deploy a full-stack application using the [fastapi/full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template), which includes a **React** frontend, **FastAPI** backend, and **PostgreSQL** database.

The deployment will follow a **GitOps** approach using **Argo CD**, with two separate environments managed via **Kubernetes namespaces**. Additionally, the project integrates with **AWS Secrets Manager** for secure configuration management and **AWS ECR** for container image storage. 

**GitHub Actions** is used to automate the CI/CD pipeline, specifically for building container images, pushing them to AWS ECR, and updating image tags in Helm charts.

## Prerequisites

Before you begin, ensure you have the following tools installed on your machine:

- **Docker**: Required for building and running container images. Follow the installation instructions for your platform:
  - [Docker installation guide](https://docs.docker.com/get-docker/)

- **KinD** (Kubernetes in Docker): Used to create local Kubernetes clusters using Docker. Install KinD by following the official documentation:
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

## Cluster Setup

To set up your Kubernetes cluster with **kind** and configure the necessary components, follow the steps below:

### Step 1: Create the KinD Cluster

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

### Step 3: Create a Private GitHub Repository

Create a private GitHub repository and copy all folders and files from the `helmcharts-repo-template` folder into your repository without changing the structure. Ensure that all files are uploaded correctly.

### Step 4: Create a GitHub App for Argo CD

Create a **GitHub App** to allow Argo CD to access this private repository.

1. Go to [GitHub Developer Settings](https://github.com/settings/apps) and click **New GitHub App**.

2. Configure the GitHub App for Argo CD:
   - GitHub App Name: Any valid name.
   - Description: "Allows Argo CD to access a private repository with Helm charts."
   - Homepage URL: In our case it can be any valid URL, for exmaple: https://argo-cd.readthedocs.io.
   - Expire user authorization tokens: **Disabled**.
   - Webhook: **Disabled**.

3. Set **Repository Permissions** for the GitHub App:
   - Contents: `Read-only`
   - Metadata: `Read-only`

4. Click **Create GitHub App**.

5. Also you need to generate a Private Key for the app. Scroll down and click **Generate a private key** (After that it will be downloaded to your machine).

### Step 5: Install the GitHub App

Now, install the GitHub App for your GitHub repository.

1. In your newly created GitHub App settings, navigate to **Install App**.
2. Click **Install** and select only the repository you created in the Step 3.
3. Confirm the installation.

### Step 6: Connect the GitHub App to Argo CD

1. Open the **Argo CD UI**.
2. Navigate to **Repositories** → **Connect Repo**.
3. Choose **GitHub App** as the connection method.
4. Fill in the required fields:
   - Type: `GitHub`
   - Project: `default`
   - Repository URL: *(Copy your GitHub repo HTTPS URL)*
   - GitHub App ID: *(Find it in your GitHub App settings)*
   - GitHub App Installation ID: *(Find it in the search bar on the Installed Apps page)*
   - GitHub App Private Key: *(Upload the private key downloaded when creating the app)*
5. Click **Connect**.
6. Verify that the repository is connected successfully

## AWS Configuring

> [!WARNING]
> It is your responsibility to monitor the costs of your AWS services and delete any unused resources to avoid unexpected charges. We are using **AWS Secrets Manager** and **AWS ECR**, which are not free by default. However, in our case, we will take advantage of AWS's free tier benefits:
  >- **AWS Secrets Manager** provides a **free plan for the first 30 days** from the creation of your first secret.
  >- **AWS ECR** offers **50 GB per month of always-free storage** for public repositories.

### Step 1: Create AWS ECR Public Repositories

In real-world projects, we typically configure **private** Amazon Elastic Container Registry (ECR) repositories to ensure security and access control. However, for this setup, we will use **public** repositories to simplify the process of pulling images for our **local Kubernetes cluster**.

> [!NOTE]
> In production, always use **private repositories** with proper authentication and permissions.

> [!IMPORTANT]
These AWS ECR repositories in our case should be created **only in the `us-east-1` region** because it will be necessary for further GitHub Actions configuration.

Go to **AWS ECR** in the AWS Console and create two **public** repositories named `backend` and `frontend` in the **us-east-1** region. 

Once these repositories are created, they will be used for storing and pulling container images for deployment in our local Kubernetes cluster.

### Step 2: Connect your AWS account with the GitHub repository

To securely connect GitHub Actions with AWS, we will use **OpenID Connect (OIDC)** and assume an AWS IAM role.

1. Go to AWS IAM and create a new OpenID Connect (OIDC) identity provider
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
2. Assign an IAM role to the identity provider
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

### Step 3: Create secrets in AWS Secrets Manager

To securely store sensitive information, we will create secrets in **AWS Secrets Manager** for both **dev** and **prod** environments.

> [!WARNING]
> The credentials provided below are not real and are for testing purposes only.
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

## Installing Public Helm Charts in GitOps Way

### Step 1: Create Secrets with AWS User Credentials

Since we are using a local Kubernetes cluster (KinD), we need to **hardcode static AWS credentials** for authentication. This approach is necessary because there is no convenient AWS identity integration available in a local setup.

> [!IMPORTANT]
> In **real projects**, you should **never** use static credentials like this. Hardcoding AWS credentials is insecure and not a best practice. Since we are using a **local KinD cluster**, static credentials are required for this setup. However, when deploying on AWS EKS or any production environment use IAM Roles for Service Accounts (IRSA) to securely grant permissions to Kubernetes workloads without exposing secrets.

Run the following commands to create the necessary Kubernetes namespaces and secrets:

```bash
kubectl create ns dev
```

```bash
kubectl create ns prod
```

```bash
kubectl create secret generic awssm-secret \
  --namespace=dev \
  --from-literal=access-key=YOUR_ACCESS_KEY \
  --from-literal=secret-access-key=YOUR_SECRET_ACCESS_KEY
```

```bash
kubectl create secret generic awssm-secret \
  --namespace=prod \
  --from-literal=access-key=YOUR_ACCESS_KEY \
  --from-literal=secret-access-key=YOUR_SECRET_ACCESS_KEY
```

### Step 2: Create an Argo CD root Application

> [!NOTE]
> In a **GitOps** workflow, we do not manually install Helm charts using `helm install` or `kubectl apply`. Instead, we define the desired state in a Git repository, and a GitOps tool like **Argo CD** continuously syncs the cluster with that state.

> [!NOTE]
> One of the best practices in Argo CD is the **App of Apps** pattern. Instead of defining each application separately, we use a **root Application** that manages multiple child applications, simplifying deployment and management.

To implement the **App of Apps** pattern, you need to create the `app-of-apps.yaml` file. The beginning of this file is commented out. **Uncomment it and populate the file with the following values**:

- repoURL: `<Your GitHub Repository HTTPS URL>`
- targetRevision: `HEAD`
- path: `argocd-apps`
- destination server: `https://kubernetes.default.svc`
- destination namespace: `argocd`
- syncPolicy: manual
- syncOptions: `CreateNamespace=true`
- revisionHistoryLimit: `5`

```bash
kubectl apply -f app-of-apps.yaml
```

### Step 3: Bootstrap Argo CD with Argo CD

Now that we have set up the **App of Apps** pattern, the next step is to define an Argo CD application for **third-party tools**, including a trick where **Argo CD manages itself**.  

Inside the **`argocd-apps`** directory, update the `argocd.yaml` file with the following values:

- repoURL: `https://argoproj.github.io/argo-helm`
- targetRevision: `7.8.2`
- chart: `argo-cd`
- helm values:
  ```yaml
  releaseName: argocd
  valuesObject:
    dex:
      enabled: false
    notifications:
      enabled: false
  ```
- destination server: `https://kubernetes.default.svc`
- destination namespace: `argocd`
- syncPolicy: `automated`
  - prune: `true`
  - selfHeal: `true`
- syncOptions: `CreateNamespace=true`
- revisionHistoryLimit: `5`

Once you have updated `argocd.yaml`, commit and push the changes to your **GitHub repository** and you will see the changes in your Argo CD UI. Next, you need to manually sync the `root` application to install Argo CD application.

### Step 4: Create an Argo CD Application for Ingress-Nginx

Next, we will create an **Argo CD application for Ingress-Nginx**, which will handle ingress traffic for our Kubernetes cluster.

Inside the **`argocd-apps`** directory, update the `ingress-nginx.yaml` file with the following values:

- repoURL: `https://kubernetes.github.io/ingress-nginx`
- targetRevision: `4.12.0`
- chart: `ingress-nginx`

- helm values:  
  ```yaml
  releaseName: ingress-nginx
  valuesObject:
    controller:
      hostPort:
        enabled: true
      service:
        enabled: false
      nodeSelector:
        ingress-ready: "true"
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
          operator: Exists
  ```
- destination server: `https://kubernetes.default.svc`
- destination namespace: `ingress-nginx`
- syncPolicy: `automated`
  - prune: `true`
  - selfHeal: `true`
- syncOptions: `CreateNamespace=true`
- revisionHistoryLimit: `5`

Once you have updated `ingress-nginx.yaml`, commit and push the changes to your **GitHub repository** and you will see the changes in your Argo CD UI. Next, you need to manually sync the `root` application to install Argo CD application.

### Step 5: Create an Argo CD Application for External Secrets

Next, we will create an **Argo CD application for External Secrets**, which allows Kubernetes to securely retrieve secrets from external secret management systems like AWS Secrets Manager.

Inside the **`argocd-apps`** directory, update the `external-secrets.yaml` file with the following values:

- repoURL: `https://charts.external-secrets.io`
- targetRevision: `0.14.1`
- chart: `external-secrets`

- helm values:  
  ```yaml
  releaseName: external-secrets
  valuesObject:
    extraObjects:
      - apiVersion: external-secrets.io/v1alpha1
        kind: ClusterSecretStore
        metadata:
          name: awssm-store
        spec:
        # You need to write the spec section yourself according to the External Secrets documentation

        # Provider: aws
        # Service: AWS Secrets Manager
        # Region: <Your AWS region where secrets are located>
        # AccessKey should be referenced from the awssm-secret Kubernetes secret you have created in the Step 1
        # SecretAccessKey should be referenced from the awssm-secret Kubernetes secret you have created in the Step 1

      - apiVersion: external-secrets.io/v1alpha1
        kind: ExternalSecret
        metadata:
          name: backend-dev-secret
          namespace: dev
        spec:
        # You need to write the spec section yourself according to the External Secrets documentation

        # Refresh Interval: 5m0s
        # Secret Store Reference to awssm-store ClusterSecretStore
        # Target Name: backend-secret
        # Data From AWS secret: myapp/dev/backend

      - apiVersion: external-secrets.io/v1alpha1
        kind: ExternalSecret
        metadata:
          name: backend-prod-secret
          namespace: prod
        spec:
        # You need to write the spec section yourself according to the External Secrets documentation

        # Refresh Interval: 5m0s
        # Secret Store Reference to awssm-store ClusterSecretStore
        # Target Name: backend-secret
        # Data From AWS secret: myapp/prod/backend
  ```

- destination server: `https://kubernetes.default.svc`
- destination namespace: `external-secrets`
- syncPolicy: `automated`
  - prune: `true`
  - selfHeal: `true`
- syncOptions: `CreateNamespace=true`
- revisionHistoryLimit: `5`

Once you have updated `external-secrets.yaml`, commit and push the changes to your **GitHub repository** and you will see the changes in your Argo CD UI. Next, you need to manually sync the `root` application to install Argo CD application.

### Step 6: Create an Argo CD ApplicationSet for PostgreSQL

We plan to use **two different environments**: `dev` and `prod`, each requiring a separate PostgreSQL database. Instead of manually creating two Argo CD `Application` resources, we can use an `ApplicationSet`, which allows us to dynamically generate multiple similar applications based on predefined parameters.

Inside the **`argocd-apps`** directory, update the `postgres.yaml` file with the following values:

- list generator with 2 elements:
  ```yaml
  - env: dev
    storageSize: 1Gi 
  - env: prod
    storageSize: 2Gi
  ```

- repoURL: `https://charts.bitnami.com/bitnami`
- targetRevision: `16.0.3`
- chart: `postgresql`

- helm values:
  ```yaml
  releaseName: "postgres-{{env}}"
  valuesObject:
    auth:
      existingSecret: "backend-secret"
      secretKeys:
        adminPasswordKey: POSTGRES_PASSWORD
    architecture: standalone
    primary:
      persistence:
        size: "{{storageSize}}"
      resourcesPreset: none
      resources:
        requests:
          cpu: 125m
          memory: 256Mi
        limits:
          cpu: 250m
          memory: 512Mi
  ```

- destination server: `https://kubernetes.default.svc`
- destination namespace: `{{env}}`
- syncPolicy: `automated`
  - prune: `true`
  - selfHeal: `true`
- syncOptions: `CreateNamespace=true`
- revisionHistoryLimit: `5`

Once you have updated `postgres.yaml`, commit and push the changes to your **GitHub repository** and you will see the changes in your Argo CD UI. Next, you need to manually sync the `root` application to install Argo CD application.

## Creating Custom Helm Charts

To ensure a structured and reusable deployment, we will use **custom Helm charts** for both the **backend** and **frontend** applications.

> [!IMPORTANT]
> We are going to create **similar Kubernetes manifests** (`Deployment`, `Service`, `Ingress`) as we did in the previous **`kube-essentials`** project. If you have completed that project, it should significantly help you **rewrite and convert** those manifests into Helm templates.

I have already predefined a **Helm chart structure** for each service, which includes:

- `Chart.yaml`
- `values.yaml`
- `values-dev.yaml`
- `values-prod.yaml`
- `templates/` directory with:
  - `deployment.yaml`
  - `service.yaml`
  - `ingress.yaml`

The YAML configuration files `values.yaml`, `values-dev.yaml`, and `values-prod.yaml` have been prepared for both the frontend and backend. By default, you don’t need to modify these files except for the `image.repository` parameter.

Your task is to populate the template files using the values provided in `values.yaml` for both the frontend and backend.

> [!NOTE]
> Not everything needs to be set through `values.yaml`. Some parameters can be hardcoded in template files if they are constant across all deployments. The key is to find a balance and avoid overcomplicating the chart by exposing every possible field in `values.yaml`, but ensure flexibility where needed.

Commit and push the updated custom Helm charts to GitHub.

## GitHub Actions Configuring

Your task is to update two GitHub workflow files: **`backend.yaml`** and **`frontend.yaml`**. Each workflow should build a Docker image, push it to a container registry, and update the Docker image tag in the Helm chart values. Make sure both files are located in the `.github/workflows` directory.

> [!NOTE]
> In real-world projects, GitHub Actions workflows are typically placed in the repository containing the source code. However, since we are using a public code template, we can place the workflows in our own repository.

For the `backend.yaml` you should:

- Define a `workflow_dispatch` trigger with a required input variable named `environment`, allowing two options: `dev` or `prod`. This input will let you manually trigger the workflow and select the target environment.

- Add the permissions:
  ```yaml
  permissions:
    id-token: write
    contents: write
  ```

- Add a step to checkout code using the `actions/checkout` action. This step will clone the repository so the workflow can access the files.

- Add a step to configure AWS credentials with `aws-actions/configure-aws-credentials`, using the `AWS_ECR_ROLE_TO_ASSUME` and `AWS_REGION` secrets.

> [!NOTE]
> Always use **IAM role assuming** instead of access and secret keys whenever possible for better security and compliance.

- Add a step to login to Amazon ECR Public using the `aws-actions/amazon-ecr-login` action with the `registry-type` set to `public`. This step authenticates the workflow to push images to Amazon ECR Public.

- Add a step to generate a short version of the Git commit SHA using the first 8 characters of `$GITHUB_SHA` and store it in the `SHORT_SHA` environment variable for use in later steps.  

  ```yaml
  - name: Get Short SHA
    run: echo "SHORT_SHA=$(echo $GITHUB_SHA | cut -c1-8)" >> $GITHUB_ENV
  ```

- Add a step to build, tag, and push Docker images to Amazon ECR Public:

  Set Environment Variables:
   - `AWS_ECR_REGISTRY`: The ECR registry URL obtained from the previous login step. 
   - `AWS_ECR_REGISTRY_ALIAS`: The registry alias stored in the `AWS_ECR_REGISTRY_ALIAS` secret.  
   - `AWS_ECR_REPOSITORY`: The name of the target repository (`backend`).
   - `BACKEND_REPOSITORY`: The source code URL of the backend application (`https://github.com/fastapi/full-stack-fastapi-template.git#0.7.1:backend`).
   - `IMAGE_TAG`: The Docker image tag set to the short Git commit SHA (`SHORT_SHA`).

  Add the commands to build a Docker image, tag it with the short Git commit SHA, and push it to Amazon ECR Public. If the environment is **`prod`**, the image is also tagged as **`latest`**.

  ```yaml
  run: |
    docker build -t $AWS_ECR_REGISTRY/$AWS_ECR_REGISTRY_ALIAS/$AWS_ECR_REPOSITORY:$IMAGE_TAG $BACKEND_REPOSITORY
    docker push $AWS_ECR_REGISTRY/$AWS_ECR_REGISTRY_ALIAS/$AWS_ECR_REPOSITORY:$IMAGE_TAG

    if [ "${{ inputs.environment }}" == "prod" ]; then
      docker tag $AWS_ECR_REGISTRY/$AWS_ECR_REGISTRY_ALIAS/$AWS_ECR_REPOSITORY:$IMAGE_TAG $AWS_ECR_REGISTRY/$AWS_ECR_REGISTRY_ALIAS/$AWS_ECR_REPOSITORY:latest
      docker push $AWS_ECR_REGISTRY/$AWS_ECR_REGISTRY_ALIAS/$AWS_ECR_REPOSITORY:latest
    fi
  ```

- Add a step to update the Docker image tag in the environment-specific `values.yaml` file of the target Helm chart using the `fjogeleit/yaml-update-action` action. The image tag is set to the short Git commit SHA.

  - valueFile: `custom-charts/backend/values-${{ github.event.inputs.environment }}.yaml`
  - propertyPath: `deployment.image.tag`
  - value: `<SHORT_SHA env variable>`
  - branch: `main`
  - message: `Update Image Version to <SHORT_SHA env variable>`

For `frontend.yaml`, follow the same steps as in `backend.yaml`, but replace **backend** with **frontend** everywhere. Additionally, add one extra step before building the image:

- Add a step to set the `BACKEND_URL` environment variable based on the selected `environment`.  
  - If `environment` is `dev`, set `BACKEND_URL` to `https://dev.myapp.local`.  
  - If `environment` is `prod`, set `BACKEND_URL` to `https://myapp.local`.  

  And when building the Docker image for the frontend, pass the `BACKEND_URL` as a build argument using:  
  
  ```bash
  docker build ... --build-arg VITE_API_URL=$BACKEND_URL ...
  ```

After configuring both `backend.yaml` and `frontend.yaml`, manually trigger the pipelines to verify that everything works correctly.

## Creating an Argo CD ApplicationSet for Custom Aplications

> [!NOTE]
> We have two applications, `frontend` and `backend`, and we need to deploy them in two different environments: `dev` and `prod`. This results in a total of four separate applications. Instead of manually defining each application, we can use an `ApplicationSet` in Argo CD to dynamically generate and manage all four applications from a single configuration. This approach ensures consistency, reduces duplication, and makes future updates easier to maintain.

Inside the **`argocd-apps`** directory, update the `custom-apps.yaml` file with the following values:

- matrix generator with 2 lists generators:
  ```yaml
  - list:
      elements:
        - app: backend
        - app: frontend
  - list:
      elements:
        - env: dev
        - env: prod
  ```

  And the result of the matrix will be 4 Argo CD Applications: `backend-prod`, `frontend-prod`, `backend-dev`, and `frontend-dev`.

- repoURL: `<Link to your GitHub repository>`
- targetRevision: `HEAD`
- chart: `custom-charts/{{app}}`
- application name: `{{app}}-{{env}}`

- helm:
    ```yaml
    valueFiles:
      - values.yaml
      - "values-{{env}}.yaml"
    ```

- destination server: `https://kubernetes.default.svc`
- destination namespace: `{{env}}`
- syncPolicy: `automated`
  - prune: `true`
  - selfHeal: `true`
- syncOptions: `CreateNamespace=true`
- revisionHistoryLimit: `5`

## Fake local domains using the `/etc/hosts` file

> [!NOTE]
> Since we are testing the application locally, we need to **map fake domains** to our local cluster. This is done by modifying the `/etc/hosts` file to point our custom domains to `127.0.0.1`. In a real-world setup, you would configure a proper DNS service, but for local testing, this approach is sufficient.

Update the `/etc/hosts` file by adding the following lines:

```
127.0.0.1 myapp.local
127.0.0.1 dev.myapp.local
```

## Test the Apps

Now that the setup is complete, it's time to **test the applications** in both environments.

1. **Make sure all pods are in the Running state**

    ```bash
    kubectl get pods -A
    ```

2. **Access the Applications**
    - UI Production: [`https://myapp.local`](https://myapp.local)
    - API Docs (Production): [`https://myapp.local/docs`](https://myapp.local/docs)
    - UI Development: [`https://dev.myapp.local`](https://dev.myapp.local)
    - API Docs (Development): [`https://dev.myapp.local/docs`](https://dev.myapp.local/docs)

3. **Test Authentication & Features**
   - Try to **log in** to both environments.
   - Create some items in each environment.
   - Make sure that items created in **one environment** are **not accessible** in the other.
   - Since `dev` and `prod` use **separate databases**, data should not overlap.

4. **Try updating some Helm charts to a newer version**

Update the Helm chart version in one of the Argo CD applications by modifying the `targetRevision` field. After making changes, go to the Argo CD UI, compare the differences, and manually sync the application to apply the update.

## Further Recommendations

Once you've tested the setup and confirmed everything works, take a moment to evaluate your efforts. This project is not easy, but it reflects real-world practices you'll see in actual work. Now, try to understand each part — Helm templating, Argo CD concepts like Application, ApplicationSet, and the App of Apps pattern, as well as External Secrets management. If something isn’t clear, take your time to explore and experiment. Learning these concepts will give you a strong foundation in Kubernetes and GitOps. Keep going — you’re building real, valuable skills!

## CleanUp

After completing the project and testing your setup, make sure to clean up all resources to avoid unnecessary use:

- Delete the fake domains from the /etc/hosts file.
- Remove the AWS resources if you don't need them anymore.
- Delete the KinD cluster to remove all Kubernetes resources and the cluster itself:

  ```bash
  kind delete cluster
  ```
