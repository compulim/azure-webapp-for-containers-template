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

The simple web server used in this template is [`serve`](https://www.npmjs.com/package/serve) from NPM. You can modify `Dockerfile` to change to another language or hosting server.

This template enables SSH access to the Docker image via Azure Web App (a.k.a. SCM or Kudu). It is a secure service provided by Azure.

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

## Setup steps

1. Azure
   1. Create resource group to hold all your resources
      - `az group create --location "West US" --name myapp-rg`
   1. Create Azure Container Registry
      - `az acr create --name myapp-acr --resource-group myapp-rg --sku Basic --admin-enabled --location "West US"`
   1. Create Service Principal
      - `az ad sp create-for-rbac --name http://myapp-sp`
      - https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest#create-a-service-principal
      - Remember to write down the password
   1. Create Azure Web App for Containers
      - https://ms.portal.azure.com/#create/Microsoft.AppSvcLinux
      - For container settings, please use default values (Nginx in single image), we will override them in our Travis CI deployment script
   1. Configure Service Principal in Azure Web App
      - Select "Access Control (IAM)"
      - Click "+ Add" and then "Add role assignment"
      - In "Role", select "Contributor"
      - In "Select", type "http://myapp-sp"
      - Click "Save"
1. Travis CI
   1. Configure Travis CI and set environment variables
      - Please refer to the section [Travis CI environment variables](#travis-ci-environment-variables)

### Travis CI environment variables

In order to deploy to Azure, you will need to set up the following environment variables in your Travis CI project settings page.

| Name                          | Description                       | Secured | Sample value                |
|-------------------------------|-----------------------------------|---------|-----------------------------|
| `ACR_NAME`                    | Azure Container Registry name     | No      | `myapp-acr`                 |
| `AZURE_SP_PASSWORD`           | Service principal password        | Yes     |                             |
| `AZURE_SP_TENANT`             | Service principal tenant ID       | Yes     | `mycompany.onmicrosoft.com` |
| `AZURE_SP_USERNAME`           | Service principal username        | Yes     | `http://myapp-sp`           |
| `AZURE_WEBAPP_NAME`           | Azure Web App name                | No      | `myapp-web`                 |
| `AZURE_WEBAPP_RESOURCE_GROUP` | Azure Web App resource group name | No      | `myapp-rg`                  |
| `DOCKER_IMAGE_NAME`           | Docker image name                 | No      | `myapp-image`               |

## Diagnostics

- Log stream at https://myapp-web.scm.azurewebsites.net/api/logstream
- SSH access at https://myapp-web.scm.azurewebsites.net/webssh/host
- Kudu at https://myapp-web.scm.azurewebsites.net/
