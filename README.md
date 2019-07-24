# Azure Web App for Containers app with CI/CD

This repository is a template for setting up a simple web server on Azure Web App using Docker single image CI/CD from GitHub and Travis CI.

We assume you have hands-on experience on these key technologies:

- Azure
   - AZ CLI
   - Azure Container Registry
   - Azure Web Apps
   - Service Principal
- Docker
- GitHub
- Travis CI

You can extend this template to use Docker Compose or Kubernetes on Azure Web App for Containers.

This template will enable SSH access to the Docker image via Azure Web App (a.k.a. SCM or Kudu). The secure tunneling service is provided by Azure.

## Setup

### GitHub and Travis CI

1. Create a new GitHub repository based on this template
1. Enable Travis CI on the GitHub repository
   - Set environment variable `DOCKER_IMAGE_NAME` to `myapp-image`

### Microsoft Azure

There are multiple resources needed to create and configure.

- [Create resource group](#create-resource-group)
- [Create Azure Container Registry](#create-azure-container-registry)
- [Create App Service Plan for Linux](#create-app-service-plan-for-linux)
- [Create Azure Web App for Containers](#create-azure-web-app-for-containers)
- [Create service principal](#create-service-principal)

#### Create resource group

Resource group groups all resources in a single place. We will walk you through one-by-one:

```sh
az group create \
  --name myapp-rg \
  --location "West US"
```

#### Create Azure Container Registry

Azure Container Registry keeps your Docker images.

```sh
az acr create \
  --resource-group myapp-rg \
  --name myapp-acr \
  --admin-enabled \
  --location "West US" \
  --sku Basic
```

> `--admin-enabled` turns on password-based access, which is required for Azure Web App.

After the registry is created, copy the name of the registry to Travis CI as environment variable named `ACR_NAME`.

Also, save the `id` value. We will use it in "[Create service principal](#create-service-principal)" step.

#### Create App Service Plan for Linux

App Service Plan is the computational resources for your web app.

```sh
az appservice plan create \
  --resource-group myapp-rg
  --name myapp-plan \
  --is-linux \
  --location "West US" \
  --sku B1
```

#### Create Azure Web App for Containers

Web App is a website hosted under the App Service Plan. One App Service Plan can host multiple Web App.

```sh
az webapp create \
  --resource-group myapp-rg \
  --name myapp \
  --plan myapp-plan \
  --deployment-container-image-name nginx
```

> Note: We will temporarily deploying NGINX image before our CI/CD pipeline is up.

After Azure Web App is created:

1. Copy the name of the web app to Travis CI as environment variable named `AZURE_WEBAPP_NAME`
1. Copy the resource group of the web app to Travis CI as environment variable named `AZURE_WEBAPP_RESOURCE_GROUP`

Also, save the `id` value. We will use it in "[Create service principal](#create-service-principal)" step.

#### Create service principal

Service principal is the service account to access your resources. We will grant "Contributor" role to both Azure Container Registry (for reading admin password) and Azure Web App (for deployment).

> You will need to replace `scopes` with the `id` values from Azure Container Registry and Azure Web App.

```sh
az ad sp create-for-rbac \
  --role Contributor \
  --scopes \
    /subscriptions/12345678-1234-5678-abcd-12345678abcd/resourceGroups/apptemplate-rg/providers/Microsoft.ContainerRegistry/registries/apptemplateacr \
    /subscriptions/12345678-1234-5678-abcd-12345678abcd/resourceGroups/apptemplate-rg/providers/Microsoft.Web/sites/apptemplateapp
```

After service principal is created, copy these values to Travis CI environment variables
   - Value of `appId` should copy to `AZURE_SP_USERNAME`
   - Value of `password` should copy to `AZURE_SP_PASSWORD`
   - Value of `tenant` should copy to `AZURE_SP_TENANT`

### Kick off the build

Go to Travis CI of your repository, and then click "Trigger build". In about 2-3 minutes, you should see your website up and running.

## How it works

- GitHub repository
   - Hosting web server using [`serve` package from NPM](https://www.npmjs.com/package/serve)
   - Static content at `/public/`
   - In `Dockerfile`, notably
      - Copy repository to `/var/web/`
      - Expose port 80 (web) and 2222 (SSH for Kudu)
      - Install SSH
      - Set entrypoint to `/usr/local/bin/init.sh`
         - Start SSH server on port 2222
         - Start the web server under `/var/web/` by calling `npm start`
- Travis CI
   - Build phase
      1. Build the Docker image and tag it `myapp-acr.azurecr.io/myapp:a1b2c3d`
         - The tag is based on Azure Container Registry name and Git commit SHA
      1. Test the Docker image by running on port 5000 and pinging it by `curl http://localhost:5000/health.txt`
   - Deployment phase
      1. Deploy to Azure by the `docker_push` script
         1. Log into Azure and Azure Container Registry
         1. Push the Docker image to Azure Container Registry
         1. Obtain Azure Container Registry credentials
         1. Configure the container image on Azure Web App
- Azure
   - Azure Container Registry to store images
   - Azure Web App for Containers for hosting Docker image
       - No webhook is required because we explicitly set the Docker image name on every continuous deployment

## Frequently asked questions

### Travis CI environment variables

You should have the following environment variables set in Travis CI:

| Name                          | Description                       | Secured | Sample value                                   |
|-------------------------------|-----------------------------------|---------|------------------------------------------------|
| `ACR_NAME`                    | Azure Container Registry name     | No      | `myapp-acr`                                    |
| `AZURE_SP_PASSWORD`           | Service principal password        | Yes     |                                                |
| `AZURE_SP_TENANT`             | Service principal tenant ID       | Yes     | GUID or `mycompany.onmicrosoft.com`            |
| `AZURE_SP_USERNAME`           | Service principal username        | Yes     | GUID or `http://azure-cli-2019-01-01-12-34-56` |
| `AZURE_WEBAPP_NAME`           | Azure Web App name                | No      | `myapp-web`                                    |
| `AZURE_WEBAPP_RESOURCE_GROUP` | Azure Web App resource group name | No      | `myapp-rg`                                     |
| `DOCKER_IMAGE_NAME`           | Docker image name                 | No      | `myapp-image`                                  |

### Diagnosing deployment issues

- For deployment log stream, https://myapp-web.scm.azurewebsites.net/api/logstream
- For SSH access into the Docker container, https://myapp-web.scm.azurewebsites.net/webssh/host

### How to rollback to previous image?

Rollback is easy. We use Git commit when versioning Docker images. You will need to find out the commit you want to rollback to:

```sh
az webapp config container set \
  --resource-group myapp-rg \
  --name myapp-web \
  --docker-custom-image-name myapp-acr.azurecr.io/myapp-image:a1b2c3d4e5f6
```

> Note: Git commit must be in long format.
