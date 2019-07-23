# Azure Web App for Containers deployed thru Travis CI

```
az group create --location "West US" --name webapptemplate

az acr create --name webapptemplate --resource-group webapptemplate --sku Basic --admin-enabled --location "West US"

az ad sp create-for-rbac --name http://webapptemplate
```
