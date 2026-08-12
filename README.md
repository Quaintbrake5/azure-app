# Deploy Node.js Express App to Azure with GitHub Actions

## 1. Open Azure Cloud Shell

In Azure Portal, click the **Cloud Shell** icon at the top.

Choose **Bash**.

First run:

```bash
az account show
```

Then:

```bash
az webapp show \
  --resource-group azure_cloud_computing \
  --name azure-node \
  --query "{name:name,state:state,hostName:defaultHostName,kind:kind}" \
  --output table
```

You should see your app.

---

## 2. Find Your Subscription ID

Run:

```bash
az account show --query id -o tsv
```

Copy the result.

Then get your tenant ID:

```bash
az account show --query tenantId -o tsv
```

Save both somewhere temporarily.

---

## 3. Create a User-Assigned Managed Identity

Run:

```bash
az identity create \
  --name azure-node-web-app-github \
  --resource-group cloudcomputinglab
```

Then get its client ID:

```bash
az identity show \
  --name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --query clientId \
  --output tsv
```

**Copy this value.**

Also get its principal ID:

```bash
az identity show \
  --name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --query principalId \
  --output tsv
```

And its resource ID:

```bash
az identity show \
  --name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --query id \
  --output tsv
```

---

## 4. Give the Identity Permission to Deploy

This is important.

Run:

```bash
az role assignment create \
  --assignee-object-id "$(az identity show \
    --name azure-node-web-app-github \
    --resource-group cloudcomputinglab \
    --query principalId -o tsv)" \
  --assignee-principal-type ServicePrincipal \
  --role "Website Contributor" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/cloudcomputinglab/providers/Microsoft.Web/sites/azure-node-web-app"
```

Microsoft documents **Website Contributor** as the role used to grant the identity permission to deploy to the App Service.

---

## 5. Create the GitHub OIDC Connection

Now we need Azure to trust GitHub Actions from this exact repository:

```text
Quaintbrake5/azure-app
```

Run:

```bash
az identity federated-credential create \
  --name github-azure-app \
  --identity-name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --issuer https://token.actions.githubusercontent.com \
  --subject repo:Quaintbrake5/azure-app:ref:refs/heads/main \
  --audiences api://AzureADTokenExchange
```

This means:

> Allow GitHub Actions running from `Quaintbrake5/azure-app` on the `main` branch to authenticate as this Azure identity.

This replaces the problematic GitHub `SourceControlToken` connection.

---

## 6. Get the Three Values GitHub Needs

Run:

```bash
echo "CLIENT ID:"
az identity show \
  --name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --query clientId -o tsv

echo "TENANT ID:"
az account show --query tenantId -o tsv

echo "SUBSCRIPTION ID:"
az account show --query id -o tsv
```

You'll get something like:

```text
CLIENT ID:
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

TENANT ID:
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

SUBSCRIPTION ID:
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**Don't send these values to anyone unnecessarily.**

---

## 7. Add the Values to GitHub

Open your repository:

**GitHub → `Quaintbrake5/azure-app`**

Then:

**Settings → Secrets and variables → Actions**

Click:

**New repository secret**

Create these three secrets.

### Secret 1

Name:

```text
AZURE_CLIENT_ID
```

Value:

```text
your client ID
```

### Secret 2

Name:

```text
AZURE_TENANT_ID
```

Value:

```text
your tenant ID
```

### Secret 3

Name:

```text
AZURE_SUBSCRIPTION_ID
```

Value:

```text
your subscription ID
```

These are the values used by the GitHub Actions Azure OIDC login.

---

## 8. Create the GitHub Actions Workflow

In your repository, create this folder:

```text
.github/workflows
```

Then create:

```text
deploy.yml
```

Put this inside:

```yaml
name: Deploy Node.js app to Azure

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

env:
  AZURE_WEBAPP_NAME: azure-node-web-app
  NODE_VERSION: '24.x'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build --if-present

      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}

      - name: Logout from Azure
        run: az logout
```

---

## 9. Commit and Push

From your local project:

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: deploy to Azure App Service"
git push origin main
```

Then go to:

**GitHub → `Quaintbrake5/azure-app` → Actions**

You should see:

```text
Deploy Node.js app to Azure
```

Click it.

You should see steps similar to:

```text
✓ Checkout repository
✓ Set up Node.js
✓ Install dependencies
✓ Build application
✓ Login to Azure
✓ Deploy to Azure App Service
✓ Logout from Azure
```

---

## 10. Check Your Node.js Project Before Running the Workflow

Because your application is a simple Express server, make sure your repository contains:

```text
azure-app/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── package.json
├── package-lock.json
└── server.js
```

Check locally:

```bash
ls
```

Then:

```bash
cat package.json
```

Your `package.json` should have a start script similar to:

```json
{
  "name": "azure-app",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^5.0.0"
  }
}
```

For a simple Express application, there is normally **no build step required**. The important steps are installing dependencies and deploying the application to App Service.


# Azure Node.js Express Deployment Guide

## 1. Fix the GitHub → Azure OIDC Authentication

Your Express app is fine. The failure is **100% in the GitHub → Azure OIDC authentication step**, before Azure even gets to your Node app.

The important line is:

```text
presented assertion subject
'repo:Quaintbrake5/azure-app@1329745931:ref:refs/heads/main'
```

But the federated credential we created was:

```text
repo:Quaintbrake5/azure-app:ref:refs/heads/main
```

Those do **not** match, so Azure rejects the GitHub token. Microsoft requires the federated credential's subject to match the GitHub OIDC subject exactly.

### 1.1 First, see your current credential

Run:

```bash
az identity federated-credential list \
  --identity-name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --output json
```

You'll probably see the old subject:

```text
repo:Quaintbrake5/azure-app:ref:refs/heads/main
```

### 1.2 Delete the incorrect credential

Run:

```bash
az identity federated-credential delete \
  --name github-azure-app \
  --identity-name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --yes
```

### 1.3 Create the credential using the subject GitHub is actually sending

Based on your error, use:

```bash
az identity federated-credential create \
  --name github-azure-app \
  --identity-name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "repo:Quaintbrake5/azure-app@1329745931:ref:refs/heads/main" \
  --audiences "api://AzureADTokenExchange"
```

Then verify:

```bash
az identity federated-credential list \
  --identity-name azure-node-web-app-github \
  --resource-group cloudcomputinglab \
  --output table
```

The subject should now contain:

```text
repo:Quaintbrake5/azure-app@1329745931:ref:refs/heads/main
```

---

# 2. About the Node 20 Warning

This:

```text
Node.js 20 is deprecated.
```

is **not** what's causing the deployment failure.

Your workflow can use Node 24:

```yaml
- name: Set up Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '24.x'
```

Because your application is just Express, your deployment can be very simple.

You don't actually need a build step at all.

---

# 3. Repository Structure

Your repository should have roughly:

```text
azure-app/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── package.json
└── server.js
```

---

# 4. package.json

Your `package.json` should have Express as a dependency and a start script:

```json
{
  "name": "azure-app",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^5.0.0"
  }
}
```

---

# 5. Express Server

Your Express server can remain simple:

```javascript
const express = require("express");

const app = express();
const PORT = process.env.PORT || 3000;

app.get("/", (req, res) => {
  res.send(`
    <h1>Hello from Azure App Service!</h1>
    <p>This application is running on Azure.</p>
  `);
});

app.get("/students", (req, res) => {
  res.json([
    { name: "Student 1", course: "Python" },
    { name: "Student 2", course: "Java" }
  ]);
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

The important part is:

```javascript
const PORT = process.env.PORT || 3000;
```

Azure provides the `PORT` environment variable, so you should **not hard-code the production port**.

---

# 6. GitHub Actions Workflow

For a simple Express application, you don't need a complicated build process.

Your workflow can be:

```yaml
name: Deploy Node.js app to Azure

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

env:
  AZURE_WEBAPP_NAME: azure-node-web-app
  NODE_VERSION: "24.x"

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
```

---

# 7. Azure App Service Startup Command

If Azure tries to run:

```text
node index.js
```

but your file is called `server.js`, you'll get:

```text
Error: Cannot find module '/home/site/wwwroot/index.js'
```

Go to:

**Azure Portal → App Service → `azure-node-web-app` → Settings → Configuration → General settings**

Find:

**Startup Command**

Set it to:

```bash
npm start
```

Save the configuration and restart the App Service.

Azure will then use:

```text
npm start
```

which runs:

```text
node server.js
```

---

# 8. Check Azure Logs

Go to:

**Azure Portal → App Service → `azure-node-web-app` → Monitoring → Log stream**

A successful startup should eventually show something similar to:

```text
Server running on port 8080
```

Azure may provide port `8080`, even though your local development environment uses `3000`. That's why this is correct:

```javascript
const PORT = process.env.PORT || 3000;
```

---

# 9. Test the Application

Your Azure App Service URL is:

```text
https://azure-node-web-app.azurewebsites.net
```

Test:

```text
/
```

and:

```text
/students
```

The `/students` endpoint should return:

```json
[
  {
    "name": "Student 1",
    "course": "Python"
  },
  {
    "name": "Student 2",
    "course": "Java"
  }
]
```

---

# 10. Monitor Logs from Azure CLI

You can also stream the application logs from Cloud Shell:

```bash
az webapp log tail \
  --name azure-node-web-app \
  --resource-group cloudcomputinglab
```

---

# 11. Overall Architecture

The deployment you're demonstrating is:

```text
                    GitHub
                       │
                       │ git push
                       ▼
                GitHub Actions
                       │
                       │ OIDC
                       ▼
              Azure Authentication
                       │
                       ▼
                 Azure App Service
                       │
                       ▼
                 Node.js 24
                       │
                       ▼
                    Express
                  ┌────┴────┐
                  │         │
                  ▼         ▼
                  /      /students
```

GitHub Actions handles the **CI/CD pipeline**, while Azure App Service handles the **hosting and runtime**.
