# 🚀 Azure CI/CD Complete Guide with Best Practices

## **Table of Contents**
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Azure Setup](#azure-setup)
4. [Infrastructure as Code (Bicep)](#infrastructure-as-code-bicep)
5. [Minimal API Setup](#minimal-api-setup)
6. [CI/CD Pipeline with GitHub Actions](#cicd-pipeline-with-github-actions)
7. [Running Unit Tests](#running-unit-tests)
8. [Feature Flagging with LaunchDarkly](#feature-flagging-with-launchdarkly)
9. [Automated Changelog](#automated-changelog)
10. [MS Teams Notifications](#ms-teams-notifications)
11. [Trunk-Based Development & Squash Merging](#trunk-based-development--squash-merging)
12. [Security Best Practices](#security-best-practices)
13. [Azure Service Principal Creation](#azure-service-principal-creation)
14. [Changelog Practices](#changelog-practices)

---

## **1️⃣ Introduction**
This guide provides a **comprehensive step-by-step** approach to setting up a **CI/CD pipeline** for deploying a **.NET 8 Minimal API** to **Azure App Service** using **GitHub Actions** and **Infrastructure as Code (Bicep)**. It incorporates **feature flagging with LaunchDarkly**, **automated changelogs**, **Trunk-Based Development (TBD)**, **MS Teams notifications**, and **security best practices** to ensure a **scalable and secure workflow**.

---

## **2️⃣ Prerequisites**
Ensure you have the following tools installed before proceeding:

### **📌 Required Tools**
- **[Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)**
  ```sh
  az --version
  ```
- **[Bicep CLI](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/install)**
  ```sh
  az bicep install
  ```
- **[GitHub CLI](https://cli.github.com/)**
  ```sh
  gh --version
  ```
- **.NET 8 SDK** ([Download](https://dotnet.microsoft.com/en-us/download))
  ```sh
  dotnet --version
  ```
- **[Docker](https://www.docker.com/get-started)** (Optional for containerization)
  ```sh
  docker --version
  ```

### **📌 Azure Access**
Login to Azure:
```sh
az login
```

Set your Azure Subscription:
```sh
az account set --subscription "YOUR_SUBSCRIPTION_ID"
```

---

## **3️⃣ Azure Setup**
### **Create a Resource Group**
```sh
az group create --name MyResourceGroup --location westeurope
```

### **Create Azure Key Vault**
```sh
az keyvault create --name my-keyvault --resource-group MyResourceGroup --location westeurope
```

### **Store Secrets in Key Vault**
```sh
az keyvault secret set --vault-name my-keyvault --name sqlAdminUsername --value 'myAdminUser'
az keyvault secret set --vault-name my-keyvault --name sqlAdminPassword --value 'SuperSecureP@ssw0rd!'
```

---

## **4️⃣ Infrastructure as Code (Bicep)**
Define **Azure infrastructure** in `infra/main.bicep`:
```bicep
param location string = 'westeurope'

resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: 'my-keyvault'
  location: location
}

resource sqlServer 'Microsoft.Sql/servers@2022-03-01' = {
  name: 'sqlserverdemo'
  location: location
  properties: {
    administratorLogin: secretUsername.properties.value
    administratorLoginPassword: secretPassword.properties.value
  }
}
```

### **Deploy Infrastructure**
```sh
az deployment group create --resource-group MyResourceGroup --template-file infra/main.bicep
```

---
## **5️⃣ Minimal API Setup**

### **Project Structure**
Your project should have the following structure:
```
/MinimalApiProject
 ├── Program.cs
 ├── appsettings.json
 ├── Dockerfile
 ├── MinimalApiProject.csproj
 ├── /Controllers
 ├── /Models
 ├── /Services
```

### **`Program.cs` (Minimal API)**
```csharp
using LaunchDarkly.Sdk;
using LaunchDarkly.Sdk.Server;

var builder = WebApplication.CreateBuilder(args);
var sdkKey = builder.Configuration["LaunchDarkly:SdkKey"];
var ldClient = new LdClient(sdkKey);

var app = builder.Build();
app.MapGet("/", async () => {
    var user = User.WithKey("default-user");
    bool featureEnabled = await ldClient.BoolVariationAsync("enableNewFeature", user, false);
    return featureEnabled ? "🚀 New Feature is Enabled!" : "🔧 Old Version!";
});
app.Run();
```

### **Dockerfile**
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
COPY . .
ENTRYPOINT ["dotnet", "MinimalApiProject.dll"]
```

---

## **6️⃣ CI/CD Pipeline with GitHub Actions**
### **Optimized `.github/workflows/deploy.yml`**

```yaml
name: Deploy to Azure
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy (staging/production)'
        required: true
        default: 'staging'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Run Unit Tests
        run: dotnet test --no-build --configuration Release --verbosity normal

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy Bicep
        run: |
          az deployment group create \
            --resource-group MyResourceGroup \
            --template-file infra/main.bicep \
            --parameters environment=${{ github.event.inputs.environment }}

      - name: Post-Deployment Health Check
        run: |
          curl --fail https://minimal-api-demo.azurewebsites.net || exit 1

      - name: Send Deployment Notification
        if: success()
        uses: mspnp/teams-notify-action@v1
        with:
          webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
          message: "🚀 Deployment to Azure (${{ github.event.inputs.environment }}) successful! 🎉"
```

---

## **7️⃣ Running Unit Tests**
### **GitHub Actions: Run Tests on PRs & Deployment**
```yaml
name: Run Unit Tests
on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Run Unit Tests
        run: dotnet test --no-build --configuration Release --verbosity normal
```

---

## **8️⃣ Feature Flagging with LaunchDarkly**
1. **Sign up** at [LaunchDarkly](https://launchdarkly.com/).
2. **Get your SDK key** and store it in **Azure Key Vault**:
   ```sh
   az keyvault secret set --vault-name my-keyvault --name LaunchDarklySdkKey --value 'your-sdk-key'
   ```
3. **Modify `Program.cs` to use feature flags** (see above).

---

## **9️⃣ Automated Changelog**
Modify **GitHub Actions** to generate a changelog.
```yaml
- name: Generate Changelog
  uses: TriPSs/conventional-changelog-action@v3
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    output-file: "CHANGELOG.md"
```
Add this step to your deployment workflow to automatically generate a changelog after each deployment.

---

## **🔟 MS Teams Notifications**
1. **Create an Incoming Webhook in MS Teams** → Copy Webhook URL.
2. **Store in GitHub Secrets** → `TEAMS_WEBHOOK_URL`.
3. **Modify GitHub Actions**:
```yaml
- name: Send MS Teams Notification
  uses: mspnp/teams-notify-action@v1
  with:
    webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
    message: "🚀 Deployment to Azure successful! 🎉"
```
Add this step to your deployment workflow to send notifications to MS Teams after successful deployments.

---

## **1️⃣1️⃣ Trunk-Based Development & Squash Merging**
Trunk-Based Development (TBD) follows these best practices:

✅ **Short-lived feature branches** – Merge daily or multiple times a day.
✅ **Squash merging** – Ensures a clean history by consolidating commits.
✅ **Require pull requests (PRs) and code reviews** before merging to `main`.
✅ **CI/CD tests must pass** before merging into `main`.

### **Configuring GitHub Repository for Squash Merging**
1. **Go to** your GitHub repo → **Settings** → **Merge Button**.
2. Enable:
   ✅ **Allow squash merging**  
   ✅ **Require pull requests before merging**  
   ✅ **Require passing CI checks before merging**  

---

## **🔐 1️⃣2️⃣ Security Best Practices**
✅ **Use Azure Key Vault** for storing secrets.
✅ **Use GitHub Actions secrets** (`secrets.AZURE_CREDENTIALS`).
✅ **Enable Role-Based Access Control (RBAC)** to restrict access.
✅ **Run unit tests before merging and deploying** to prevent regressions.
✅ **Implement Trunk-Based Development (TBD) with squash merging** to maintain code quality and prevent conflicts.
✅ **Monitor deployments with MS Teams notifications** to ensure visibility into production changes.
✅ **Use Azure Managed Identities** to avoid hardcoding credentials.
✅ **Enable Multi-Factor Authentication (MFA)** for all Azure accounts.
✅ **Regularly rotate secrets and credentials** to minimize risk.

---

## **1️⃣3️⃣ Azure Service Principal Creation**
### **Create a Service Principal**
Create a service principal for GitHub Actions to authenticate with Azure:
```sh
az ad sp create-for-rbac --name "github-actions-sp" --role contributor --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/MyResourceGroup --sdk-auth
```
Copy the output JSON and add it to your GitHub repository secrets as `AZURE_CREDENTIALS`.

---

## **1️⃣4️⃣ Changelog Practices**
Maintaining a changelog is essential for tracking changes, improvements, and fixes in your project. Follow these practices to ensure a consistent and useful changelog:

### **Commit Message Guidelines**
- Use conventional commit messages to categorize changes:
  - `feat`: A new feature
  - `fix`: A bug fix
  - `docs`: Documentation changes
  - `style`: Code style changes (formatting, missing semi-colons, etc.)
  - `refactor`: Code refactoring without changing functionality
  - `test`: Adding or updating tests
  - `chore`: Maintenance tasks (updating dependencies, etc.)

### **Generating Changelog**
- Use the **conventional-changelog-action** in your GitHub Actions workflow to automatically generate a changelog.
- Ensure the changelog is updated with each deployment by including the changelog generation step in your CI/CD pipeline.

### **Example Workflow**
```yaml
name: Deploy to Azure
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy (staging/production)'
        required: true
        default: 'staging'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Run Unit Tests
        run: dotnet test --no-build --configuration Release --verbosity normal

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Deploy Bicep
        run: |
          az deployment group create \
            --resource-group MyResourceGroup \
            --template-file infra/main.bicep \
            --parameters environment=${{ github.event.inputs.environment }}

      - name: Generate Changelog
        uses: TriPSs/conventional-changelog-action@v3
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          output-file: "CHANGELOG.md"

      - name: Post-Deployment Health Check
        run: |
          curl --fail https://minimal-api-demo.azurewebsites.net || exit 1

      - name: Send Deployment Notification
        if: success()
        uses: mspnp/teams-notify-action@v1
        with:
          webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
          message: "🚀 Deployment to Azure (${{ github.event.inputs.environment }}) successful! 🎉"
```

By following these practices, you can maintain a clear and organized changelog that provides valuable insights into the evolution of your project.

---

## **🎯 Final Notes**
✅ **Fully automated CI/CD pipeline** with **Azure, GitHub Actions, Feature Flags, Changelogs, Trunk-Based Development, MS Teams Notifications, and Unit Testing!** 🚀

