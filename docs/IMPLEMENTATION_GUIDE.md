# End-to-End Implementation Guide
## Microsoft Teams Bot + Graph Integration for IRIS CARBON
### Production-Ready Setup — C# / .NET 8 / Azure

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Azure App Registration Setup](#2-azure-app-registration-setup)
3. [Azure Resources Provisioning](#3-azure-resources-provisioning)
4. [Bot Framework Setup](#4-bot-framework-setup)
5. [Teams App Manifest](#5-teams-app-manifest)
6. [.NET Core Solution Scaffold](#6-net-core-solution-scaffold)
7. [NuGet Package References](#7-nuget-package-references)
8. [Database Design and MongoDB Setup](#8-database-design-and-mongodb-setup)
9. [Domain Layer Implementation](#9-domain-layer-implementation)
10. [Infrastructure Layer Implementation](#10-infrastructure-layer-implementation)
11. [Application Layer Implementation](#11-application-layer-implementation)
12. [Bot Layer Implementation](#12-bot-layer-implementation)
13. [API Layer Implementation](#13-api-layer-implementation)
14. [Key Vault and Configuration Wiring](#14-key-vault-and-configuration-wiring)
15. [Adaptive Card JSON Templates](#15-adaptive-card-json-templates)
16. [Local Development and Testing](#16-local-development-and-testing)
17. [CI/CD Pipeline (Azure DevOps)](#17-cicd-pipeline-azure-devops)
18. [Production Deployment Checklist](#18-production-deployment-checklist)
19. [Post-Deployment Verification](#19-post-deployment-verification)

---

## 1. Prerequisites

### 1.1 Required Accounts and Licenses

| Requirement | Detail |
|---|---|
| Azure Subscription | Owner or Contributor role required |
| Microsoft 365 Tenant | Teams license required for users |
| Entra ID | Global Admin or Application Admin role for app registration |
| Teams Admin | Teams admin access to upload custom apps |

### 1.2 Developer Workstation Tools

```text
- .NET 8 SDK                    https://dotnet.microsoft.com/download
- Visual Studio 2022 (17.8+)    Community or Enterprise
  OR VS Code with C# Dev Kit
- Azure CLI (2.55+)             https://learn.microsoft.com/cli/azure/install-azure-cli
- Azure Functions Core Tools    v4 (if using Azure Functions)
- MongoDB Compass               https://www.mongodb.com/products/compass
- Bot Framework Emulator        https://github.com/microsoft/BotFramework-Emulator
- ngrok                         https://ngrok.com  (local tunnel for bot testing)
- Node.js 18+                   Required for Teams Toolkit CLI
- Teams Toolkit CLI             npm install -g @microsoft/teamsfx-cli
- Git
```

### 1.3 Azure CLI Login

```bash
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
az account show
```

### 1.4 Environment Variable Naming Convention

Throughout this guide, placeholders use this pattern:

```text
<TENANT_ID>         - Your Entra / Azure AD tenant GUID
<SUBSCRIPTION_ID>   - Your Azure subscription GUID
<BOT_APP_ID>        - App registration client ID for the bot
<GRAPH_APP_ID>      - App registration client ID for Graph (can be the same as bot)
<RESOURCE_GROUP>    - e.g. rg-carbon-teams-prod
<APP_SERVICE_NAME>  - e.g. app-carbon-teams-prod
<COSMOS_ACCOUNT>    - e.g. cosmos-carbon-teams-prod
<KV_NAME>           - e.g. kv-carbon-teams-prod
<AI_NAME>           - e.g. appi-carbon-teams-prod
```

---

## 2. Azure App Registration Setup

### 2.1 Create the Bot and Graph App Registration

You can use one app registration for both bot identity and Graph access.

**Step 1: Create App Registration in the Azure Portal**

```
Azure Portal > Entra ID > App Registrations > New Registration
  Name:           Carbon Teams Bot
  Supported accounts: Accounts in any organizational directory (Multitenant)
  Redirect URI:   Leave blank for now
  Click Register
```

Record:
- **Application (client) ID** → `<BOT_APP_ID>`
- **Directory (tenant) ID**   → `<TENANT_ID>`

---

**Step 2: Create a Client Secret**

```
App Registration > Certificates & Secrets > Client Secrets > New Client Secret
  Description: prod-bot-secret
  Expires:     24 months
  Click Add
```

**Copy the Value immediately** — it is shown only once.  
Store it as `<BOT_CLIENT_SECRET>`.

> **Production recommendation:** Use a certificate instead of a client secret.

**Step 2b (Production): Certificate credential**

```bash
# Generate a self-signed cert (or use your CA-issued cert)
openssl req -x509 -newkey rsa:4096 -keyout bot-key.pem -out bot-cert.pem \
  -days 730 -nodes -subj "/CN=CarbonTeamsBot"

# Convert to PFX
openssl pkcs12 -export -out bot-cert.pfx -inkey bot-key.pem -in bot-cert.pem

# Upload bot-cert.pem (public key) to App Registration > Certificates & Secrets > Certificates
```

---

**Step 3: Configure API Permissions**

```
App Registration > API Permissions > Add a permission

Microsoft Graph > Application permissions:
  Team.Create
  TeamMember.ReadWrite.All
  Channel.Create
  Channel.ReadBasic.All
  ChannelMember.ReadWrite.All
  ChatMessage.Read.All        (for reading channel messages if needed)
  User.Read.All               (to resolve user identity)
  Group.ReadWrite.All                          (required for Team creation via Groups)
  TeamsApp.ReadWrite.All                       (upload app to customer org catalog)
  TeamsAppInstallation.ReadWriteForTeam.All    (auto-install bot in provisioned Team)

Click "Grant admin consent for <your tenant>"
```

> Note: `Group.ReadWrite.All` and `Team.Create` require **admin consent**. These are application permissions (no user sign-in required) for the backend daemon flow.

---

**Step 4: Expose an API (for bot token validation)**

```
App Registration > Expose an API > Set Application ID URI
  Value: api://<BOT_APP_ID>
  Click Save

Add a Scope:
  Scope Name: access_as_bot
  Admin Consent Display Name: Access as bot
  Admin Consent Description: Allows the app to access as a bot
  State: Enabled
```

---

**Step 5: Authentication settings**

```
App Registration > Authentication > Add a Platform > Web
  Redirect URIs: https://<APP_SERVICE_NAME>.azurewebsites.net/auth/callback
  
Front-channel logout URL: (leave blank or set if needed)

Implicit grant and hybrid flows:
  - Do NOT enable implicit grant for production

Click Save
```

---

### 2.2 App Registration Summary

| Setting | Value |
|---|---|
| App Name | Carbon Teams Bot |
| Client ID | `<BOT_APP_ID>` |
| Tenant ID | `<TENANT_ID>` |
| Supported Accounts | **Multitenant** — any organizational directory |
| Client Secret | stored in Key Vault |
| Graph API Permissions | Team.Create, Channel.Create, TeamsApp.ReadWrite.All, TeamsAppInstallation.ReadWriteForTeam.All, etc. |
| Admin Consent | Granted once by each customer tenant admin |

---

## 3. Azure Resources Provisioning

### 3.1 Create Resource Group

```bash
az group create \
  --name <RESOURCE_GROU
  --location eastus
```

---

### 3.2 Create Azure Cosmos DB for MongoDB API

```bash
# Create Cosmos DB account with MongoDB API
az cosmosdb create \
  --name <COSMOS_ACCOUNT> \
  --resource-group <RESOURCE_GROUP> \
  --kind MongoDB \
  --server-version 6.0 \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=false

# Create the MongoDB database
az cosmosdb mongodb database create \
  --account-name <COSMOS_ACCOUNT> \
  --resource-group <RESOURCE_GROUP> \
  --name CarbonTeamsDb

# Create collections
for collection in OrgChannelMappings ProvisionedTeams ProvisionedChannels \
  ApprovalCardInstances ApprovalAuditRecords ValidationAlertInstances; do
  az cosmosdb mongodb collection create \
    --account-name <COSMOS_ACCOUNT> \
    --resource-group <RESOURCE_GROUP> \
    --database-name CarbonTeamsDb \
    --name $collection \
    --shard _id
done
```

Retrieve the connection string (store in Key Vault):
```bash
az cosmosdb keys list \
  --name <COSMOS_ACCOUNT> \
  --resource-group <RESOURCE_GROUP> \
  --type connection-strings \
  --query "connectionStrings[0].connectionString" -o tsv
```

Connection string format:
```text
mongodb://<COSMOS_ACCOUNT>:<PRIMARY_KEY>@<COSMOS_ACCOUNT>.mongo.cosmos.azure.com:10255/?ssl=true&retrywrites=false&maxIdleTimeMS=120000&appName=@<COSMOS_ACCOUNT>@
```

> **Alternatively:** Use MongoDB Atlas hosted on Azure — create a cluster in the same region, enable Private Endpoint, and use the Atlas connection string.

> **Production:** Enable Cosmos DB firewall to allow only App Service outbound IPs, or configure a Private Endpoint.

---

### 3.3 Create Azure Key Vault

```bash
az keyvault create \
  --name <KV_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --location eastus \
  --sku standard \
  --enable-rbac-authorization true
```

Grant your App Service Managed Identity access later (see section 14).

---

### 3.4 Create Application Insights

```bash
# Create Log Analytics Workspace first
az monitor log-analytics workspace create \
  --resource-group <RESOURCE_GROUP> \
  --workspace-name law-carbon-teams-prod \
  --location eastus

# Create Application Insights
az monitor app-insights component create \
  --app <AI_NAME> \
  --location eastus \
  --resource-group <RESOURCE_GROUP> \
  --workspace law-carbon-teams-prod
```

Record `instrumentationKey` and `connectionString` from the output.

---

### 3.5 Create App Service Plan and App Service

```bash
# App Service Plan (use P1v3 or higher for production)
az appservice plan create \
  --name asp-carbon-teams-prod \
  --resource-group <RESOURCE_GROUP> \
  --sku P1V3 \
  --is-linux false \
  --location eastus

# App Service
az webapp create \
  --name <APP_SERVICE_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --plan asp-carbon-teams-prod \
  --runtime "DOTNET|8.0"

# Enable System Assigned Managed Identity
az webapp identity assign \
  --name <APP_SERVICE_NAME> \
  --resource-group <RESOURCE_GROUP>
```

Record the `principalId` from the identity assign output — needed for Key Vault RBAC.

---

### 3.6 Create Azure Service Bus (Optional Async)

```bash
az servicebus namespace create \
  --name sb-carbon-teams-prod \
  --resource-group <RESOURCE_GROUP> \
  --location eastus \
  --sku Standard

az servicebus queue create \
  --name approval-events \
  --namespace-name sb-carbon-teams-prod \
  --resource-group <RESOURCE_GROUP>
```

---

### 3.7 Store Secrets in Key Vault

```bash
# Bot client secret
az keyvault secret set \
  --vault-name <KV_NAME> \
  --name "BotClientSecret" \
  --value "<BOT_CLIENT_SECRET>"

# MongoDB Connection String
az keyvault secret set \
  --vault-name <KV_NAME> \
  --name "MongoDbConnectionString" \
  --value "<FULL_MONGODB_CONNECTION_STRING>"

# Audit HMAC Secret (for integrity hash)
az keyvault secret set \
  --vault-name <KV_NAME> \
  --name "AuditHmacSecret" \
  --value "<RANDOM_256_BIT_HEX>"
```

---

## 4. Bot Framework Setup

### 4.1 Create Azure Bot Resource

```
Azure Portal > Create a resource > Azure Bot

  Bot handle:     carbon-teams-bot
  Subscription:   <your subscription>
  Resource group: <RESOURCE_GROUP>
  Pricing Tier:   F0 (free) or S1 (standard for production)
  
  App ID type:    Use existing app registration
  Existing App ID: <BOT_APP_ID>
  
  Click Create
```

### 4.2 Configure Teams Channel in Azure Bot

```
Azure Bot resource > Channels > Microsoft Teams

  Click on Microsoft Teams icon
  Accept the terms
  Save

  Channel Name: Microsoft Teams
  Status:       Running
```

### 4.3 Set Bot Messaging Endpoint

```
Azure Bot resource > Configuration

  Messaging Endpoint: https://<APP_SERVICE_NAME>.azurewebsites.net/api/messages

  Click Apply
```

> For local development, use ngrok:
> ```bash
> ngrok http 5000
> # Use the https ngrok URL as messaging endpoint during development
> ```

### 4.4 Enable App Service CORS (if needed)

```bash
az webapp cors add \
  --name <APP_SERVICE_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --allowed-origins "https://teams.microsoft.com"
```

---

## 5. Teams App Manifest

### 5.1 Folder Structure

```text
teams-manifest/
  ├── manifest.json
  ├── color.png         (192x192 pixels)
  └── outline.png       (32x32 pixels)
```

### 5.2 manifest.json

```json
{
  "$schema": "https://developer.microsoft.com/en-us/json-schemas/teams/v1.17/MicrosoftTeams.schema.json",
  "manifestVersion": "1.17",
  "version": "1.0.0",
  "id": "<BOT_APP_ID>",
  "packageName": "com.iriscarbon.teams",
  "developer": {
    "name": "IRIS CARBON",
    "websiteUrl": "https://iriscarbon.com",
    "privacyUrl": "https://iriscarbon.com/privacy",
    "termsOfUseUrl": "https://iriscarbon.com/terms"
  },
  "icons": {
    "color": "color.png",
    "outline": "outline.png"
  },
  "name": {
    "short": "IRIS CARBON",
    "full": "IRIS CARBON Approval Bot"
  },
  "description": {
    "short": "Approval workflows for IRIS CARBON",
    "full": "Post and respond to approval cards for IRIS CARBON disclosure workflows directly in Microsoft Teams."
  },
  "accentColor": "#0078D4",
  "bots": [
    {
      "botId": "<BOT_APP_ID>",
      "scopes": ["team", "groupchat"],
      "supportsFiles": false,
      "isNotificationOnly": false
    }
  ],
  "permissions": ["identity", "messageTeamMembers"],
  "validDomains": [
    "<APP_SERVICE_NAME>.azurewebsites.net",
    "iriscarbon.com"
  ],
  "webApplicationInfo": {
    "id": "<BOT_APP_ID>",
    "resource": "api://<BOT_APP_ID>"
  }
}
```

### 5.3 Package the Manifest

```bash
cd teams-manifest
zip -r carbon-teams-app.zip manifest.json color.png outline.png
```

### 5.4 Upload to Teams Admin Center

```
Teams Admin Center > Teams apps > Manage apps > Upload new app
  Upload the carbon-teams-app.zip file
  Set availability: specific users/groups or entire org
```

---

## 6. .NET Core Solution Scaffold

### 6.1 Create Solution and Projects

```bash
# Create solution
dotnet new sln -n Carbon.Teams

# Create projects
dotnet new classlib -n Carbon.Teams.Domain         -f net8.0
dotnet new classlib -n Carbon.Teams.Contracts      -f net8.0
dotnet new classlib -n Carbon.Teams.Application    -f net8.0
dotnet new classlib -n Carbon.Teams.Infrastructure -f net8.0
dotnet new webapi   -n Carbon.Teams.Api            -f net8.0
dotnet new webapi   -n Carbon.Teams.Bot            -f net8.0
dotnet new xunit    -n Carbon.Teams.Tests          -f net8.0

# Add projects to solution
dotnet sln Carbon.Teams.sln add \
  Carbon.Teams.Domain/Carbon.Teams.Domain.csproj \
  Carbon.Teams.Contracts/Carbon.Teams.Contracts.csproj \
  Carbon.Teams.Application/Carbon.Teams.Application.csproj \
  Carbon.Teams.Infrastructure/Carbon.Teams.Infrastructure.csproj \
  Carbon.Teams.Api/Carbon.Teams.Api.csproj \
  Carbon.Teams.Bot/Carbon.Teams.Bot.csproj \
  Carbon.Teams.Tests/Carbon.Teams.Tests.csproj

# Add project references
dotnet add Carbon.Teams.Application/Carbon.Teams.Application.csproj reference \
  Carbon.Teams.Domain/Carbon.Teams.Domain.csproj \
  Carbon.Teams.Contracts/Carbon.Teams.Contracts.csproj

dotnet add Carbon.Teams.Infrastructure/Carbon.Teams.Infrastructure.csproj reference \
  Carbon.Teams.Application/Carbon.Teams.Application.csproj \
  Carbon.Teams.Domain/Carbon.Teams.Domain.csproj

dotnet add Carbon.Teams.Bot/Carbon.Teams.Bot.csproj reference \
  Carbon.Teams.Application/Carbon.Teams.Application.csproj \
  Carbon.Teams.Infrastructure/Carbon.Teams.Infrastructure.csproj \
  Carbon.Teams.Contracts/Carbon.Teams.Contracts.csproj

dotnet add Carbon.Teams.Api/Carbon.Teams.Api.csproj reference \
  Carbon.Teams.Application/Carbon.Teams.Application.csproj \
  Carbon.Teams.Infrastructure/Carbon.Teams.Infrastructure.csproj \
  Carbon.Teams.Contracts/Carbon.Teams.Contracts.csproj

dotnet add Carbon.Teams.Tests/Carbon.Teams.Tests.csproj reference \
  Carbon.Teams.Application/Carbon.Teams.Application.csproj \
  Carbon.Teams.Domain/Carbon.Teams.Domain.csproj
```

---

## 7. NuGet Package References

### 7.1 Carbon.Teams.Bot

```xml
<PackageReference Include="Microsoft.Bot.Builder" Version="4.22.3" />
<PackageReference Include="Microsoft.Bot.Builder.Integration.AspNet.Core" Version="4.22.3" />
<PackageReference Include="Microsoft.Bot.Connector" Version="4.22.3" />
<PackageReference Include="AdaptiveCards" Version="3.1.0" />
<PackageReference Include="Azure.Identity" Version="1.11.4" />
<PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.22.0" />
```

### 7.2 Carbon.Teams.Api

```xml
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.0" />
<PackageReference Include="Azure.Identity" Version="1.11.4" />
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.6.0" />
<PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.22.0" />
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.6.2" />
```

### 7.3 Carbon.Teams.Infrastructure

```xml
<PackageReference Include="MongoDB.Driver" Version="2.26.0" />
<PackageReference Include="MongoDB.Bson" Version="2.26.0" />
<PackageReference Include="Microsoft.Graph" Version="5.55.0" />
<PackageReference Include="Azure.Identity" Version="1.11.4" />
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.6.0" />
<PackageReference Include="Microsoft.Bot.Builder" Version="4.22.3" />
```

### 7.4 Carbon.Teams.Application

```xml
<PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="8.0.0" />
<PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="8.0.0" />
```

---

## 8. Database Design and MongoDB Setup

### 8.1 Domain Entities (Carbon.Teams.Domain)

```csharp
// Carbon.Teams.Domain/Entities/OrganizationChannelMapping.cs
namespace Carbon.Teams.Domain.Entities
{
    public class OrganizationChannelMapping
    {
        public Guid Id { get; set; }
        public string CompanyId { get; set; } = string.Empty;
        public string TeamId { get; set; } = string.Empty;
        public string ChannelId { get; set; } = string.Empty;
        public string TenantId { get; set; } = string.Empty;
        public string ConversationId { get; set; } = string.Empty;
        public string ServiceUrl { get; set; } = string.Empty;
        public bool IsActive { get; set; }
        public DateTime CreatedUtc { get; set; }
        public DateTime UpdatedUtc { get; set; }
    }
}
```

```csharp
// Carbon.Teams.Domain/Entities/ProvisionedTeam.cs
namespace Carbon.Teams.Domain.Entities
{
    public class ProvisionedTeam
    {
        public Guid Id { get; set; }
        public string CompanyId { get; set; } = string.Empty;
        public string TeamId { get; set; } = string.Empty;
        public string TeamDisplayName { get; set; } = string.Empty;
        public string TenantId { get; set; } = string.Empty;
        public string ProvisioningStatus { get; set; } = string.Empty;
        public DateTime CreatedUtc { get; set; }
        public DateTime UpdatedUtc { get; set; }
    }
}
```

```csharp
// Carbon.Teams.Domain/Entities/ApprovalCardInstance.cs
namespace Carbon.Teams.Domain.Entities
{
    public class ApprovalCardInstance
    {
        public Guid CardInstanceId { get; set; }
        public string CompanyId { get; set; } = string.Empty;
        public string SectionId { get; set; } = string.Empty;
        public string DocumentId { get; set; } = string.Empty;
        public string DocumentVersion { get; set; } = string.Empty;
        public string TeamId { get; set; } = string.Empty;
        public string ChannelId { get; set; } = string.Empty;
        public string ConversationId { get; set; } = string.Empty;
        public string TeamsMessageId { get; set; } = string.Empty;
        public string SectionVersionHash { get; set; } = string.Empty;
        public DateTime IssuedAtUtc { get; set; }
        public string Status { get; set; } = ApprovalCardStatus.Active;
        public Guid? SupersededByCardInstanceId { get; set; }
        public DateTime LastUpdatedUtc { get; set; }
    }

    public static class ApprovalCardStatus
    {
        public const string Active = "Active";
        public const string Approved = "Approved";
        public const string Rejected = "Rejected";
        public const string Stale = "Stale";
        public const string Superseded = "Superseded";
        public const string Expired = "Expired";
    }
}
```

```csharp
// Carbon.Teams.Domain/Entities/ApprovalAuditRecord.cs
namespace Carbon.Teams.Domain.Entities
{
    public class ApprovalAuditRecord
    {
        public Guid Id { get; set; }
        public string ApproverUserId { get; set; } = string.Empty;
        public string DisplayName { get; set; } = string.Empty;
        public string TenantId { get; set; } = string.Empty;
        public DateTime ServerTimestampUtc { get; set; }
        public string Decision { get; set; } = string.Empty;
        public string? RejectReason { get; set; }
        public string? ApproveComment { get; set; }
        public string SectionId { get; set; } = string.Empty;
        public string DocumentId { get; set; } = string.Empty;
        public string DocumentVersion { get; set; } = string.Empty;
        public string SectionVersionHash { get; set; } = string.Empty;
        public string PreviousState { get; set; } = string.Empty;
        public string NewState { get; set; } = string.Empty;
        public string SourceChannel { get; set; } = "Teams";
        public string CorrelationId { get; set; } = string.Empty;
        public string TeamsConversationId { get; set; } = string.Empty;
        public string TeamsMessageId { get; set; } = string.Empty;
        public Guid CardInstanceId { get; set; }
        public string IntegrityHash { get; set; } = string.Empty;
        public DateTime CreatedUtc { get; set; }
    }
}
```

```csharp
// Carbon.Teams.Domain/Entities/ValidationAlertInstance.cs
namespace Carbon.Teams.Domain.Entities
{
    public class ValidationAlertInstance
    {
        public Guid AlertInstanceId { get; set; }
        public string CompanyId { get; set; } = string.Empty;
        public string? SectionId { get; set; }
        public string DocumentId { get; set; } = string.Empty;
        public string IssueType { get; set; } = string.Empty;
        public string Severity { get; set; } = string.Empty;
        public string IssueKey { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public string TeamId { get; set; } = string.Empty;
        public string ChannelId { get; set; } = string.Empty;
        public string ConversationId { get; set; } = string.Empty;
        public string TeamsMessageId { get; set; } = string.Empty;
        public string Status { get; set; } = "Active";
        public DateTime LastUpdatedUtc { get; set; }
        public DateTime CreatedUtc { get; set; }
    }
}
```

---

### 8.2 MongoDB Context (Carbon.Teams.Infrastructure)

> **BSON Mapping:** Keep domain entities free of framework attributes. Register `GuidSerializer` and conventions in Infrastructure via `MongoBsonConfiguration.Register()` (see section 8.4).

```csharp
// Carbon.Teams.Infrastructure/Persistence/MongoDbContext.cs
using Carbon.Teams.Domain.Entities;
using MongoDB.Driver;

namespace Carbon.Teams.Infrastructure.Persistence
{
    public class MongoDbContext
    {
        private readonly IMongoDatabase _database;

        public MongoDbContext(IMongoDatabase database)
        {
            _database = database;
        }

        public IMongoCollection<OrganizationChannelMapping> OrgChannelMappings =>
            _database.GetCollection<OrganizationChannelMapping>("OrgChannelMappings");

        public IMongoCollection<ProvisionedTeam> ProvisionedTeams =>
            _database.GetCollection<ProvisionedTeam>("ProvisionedTeams");

        public IMongoCollection<ProvisionedChannel> ProvisionedChannels =>
            _database.GetCollection<ProvisionedChannel>("ProvisionedChannels");

        public IMongoCollection<ApprovalCardInstance> ApprovalCardInstances =>
            _database.GetCollection<ApprovalCardInstance>("ApprovalCardInstances");

        public IMongoCollection<ApprovalAuditRecord> ApprovalAuditRecords =>
            _database.GetCollection<ApprovalAuditRecord>("ApprovalAuditRecords");

        public IMongoCollection<ValidationAlertInstance> ValidationAlertInstances =>
            _database.GetCollection<ValidationAlertInstance>("ValidationAlertInstances");
    }
}
```

### 8.3 Create MongoDB Indexes

Create indexes at application startup using `MongoIndexInitializer`. Add the call to `Program.cs` after the app is built.

```csharp
// Carbon.Teams.Infrastructure/Persistence/MongoIndexInitializer.cs
using Carbon.Teams.Domain.Entities;
using MongoDB.Driver;

namespace Carbon.Teams.Infrastructure.Persistence
{
    public static class MongoIndexInitializer
    {
        public static async Task CreateIndexesAsync(MongoDbContext context)
        {
            // OrgChannelMappings — unique index on active org
            await context.OrgChannelMappings.Indexes.CreateOneAsync(
                new CreateIndexModel<OrganizationChannelMapping>(
                    Builders<OrganizationChannelMapping>.IndexKeys.Ascending(x => x.CompanyId),
                    new CreateIndexOptions
                    {
                        Unique = true,
                        PartialFilterExpression = Builders<OrganizationChannelMapping>
                            .Filter.Eq(x => x.IsActive, true),
                        Name = "UX_OrgChannelMapping_ActiveOrg"
                    }));

            // ApprovalCardInstances — composite query index
            await context.ApprovalCardInstances.Indexes.CreateOneAsync(
                new CreateIndexModel<ApprovalCardInstance>(
                    Builders<ApprovalCardInstance>.IndexKeys
                        .Ascending(x => x.CompanyId)
                        .Ascending(x => x.SectionId)
                        .Ascending(x => x.Status)));

            // ApprovalAuditRecords — sort and correlation indexes
            await context.ApprovalAuditRecords.Indexes.CreateOneAsync(
                new CreateIndexModel<ApprovalAuditRecord>(
                    Builders<ApprovalAuditRecord>.IndexKeys
                        .Ascending(x => x.SectionId)
                        .Descending(x => x.CreatedUtc)));

            await context.ApprovalAuditRecords.Indexes.CreateOneAsync(
                new CreateIndexModel<ApprovalAuditRecord>(
                    Builders<ApprovalAuditRecord>.IndexKeys.Ascending(x => x.CorrelationId)));

            // ValidationAlertInstances — unique active alert per issue key
            await context.ValidationAlertInstances.Indexes.CreateOneAsync(
                new CreateIndexModel<ValidationAlertInstance>(
                    Builders<ValidationAlertInstance>.IndexKeys
                        .Ascending(x => x.CompanyId)
                        .Ascending(x => x.IssueKey),
                    new CreateIndexOptions
                    {
                        Unique = true,
                        PartialFilterExpression = Builders<ValidationAlertInstance>
                            .Filter.Eq(x => x.Status, "Active"),
                        Name = "UX_ValidationAlert_ActiveIssue"
                    }));
        }
    }
}
```

Call from `Program.cs` after building the app:
```csharp
using (var scope = app.Services.CreateScope())
{
    var mongoContext = scope.ServiceProvider.GetRequiredService<MongoDbContext>();
    await MongoIndexInitializer.CreateIndexesAsync(mongoContext);
}
```

### 8.4 BSON Serialization Configuration

Call `MongoBsonConfiguration.Register()` once at startup (before any MongoDB access) to handle `Guid` serialization and keep domain entities attribute-free.

```csharp
// Carbon.Teams.Infrastructure/Persistence/MongoBsonConfiguration.cs
using MongoDB.Bson;
using MongoDB.Bson.Serialization;
using MongoDB.Bson.Serialization.Conventions;
using MongoDB.Bson.Serialization.Serializers;

namespace Carbon.Teams.Infrastructure.Persistence
{
    public static class MongoBsonConfiguration
    {
        private static bool _registered;
        private static readonly object _lock = new object();

        public static void Register()
        {
            lock (_lock)
            {
                if (_registered) return;

                // Store Guid as string instead of binary
                BsonSerializer.RegisterSerializer(new GuidSerializer(BsonType.String));

                var pack = new ConventionPack
                {
                    new CamelCaseElementNameConvention(),
                    new IgnoreExtraElementsConvention(true)
                };
                ConventionRegistry.Register("CarbonTeamsConventions", pack, _ => true);

                _registered = true;
            }
        }
    }
}
```

---

## 9. Domain Layer Implementation

### 9.1 Service Interfaces (Carbon.Teams.Application/Interfaces)

```csharp
// ITeamsProvisioningService.cs
namespace Carbon.Teams.Application.Interfaces
{
    public interface ITeamsProvisioningService
    {
        Task<TeamProvisionResult> CreateTeamAsync(CreateTeamRequest request, CancellationToken ct = default);
        Task<ChannelProvisionResult> CreateChannelAsync(CreateChannelRequest request, CancellationToken ct = default);
        Task<OrgChannelMappingResult> MapOrganizationChannelAsync(MapOrgChannelRequest request, CancellationToken ct = default);
        Task DeactivateMappingAsync(string companyId, CancellationToken ct = default);
    }
}
```

```csharp
// IApprovalCardPostingService.cs
namespace Carbon.Teams.Application.Interfaces
{
    public interface IApprovalCardPostingService
    {
        Task<CardPostResult> PostApprovalCardAsync(PostApprovalCardRequest request, CancellationToken ct = default);
        Task<CardUpdateResult> UpdateApprovalCardAsync(UpdateApprovalCardRequest request, CancellationToken ct = default);
        Task<CardUpdateResult> MarkCardStaleAsync(Guid cardInstanceId, CancellationToken ct = default);
        Task<CardUpdateResult> MarkCardSupersededAsync(Guid cardInstanceId, Guid supersededByCardInstanceId, CancellationToken ct = default);
    }
}
```

```csharp
// IApprovalActionService.cs
namespace Carbon.Teams.Application.Interfaces
{
    public interface IApprovalActionService
    {
        Task<ApprovalActionResult> HandleApprovalActionAsync(ApprovalActionCommand command, CancellationToken ct = default);
    }
}
```

```csharp
// IAuditService.cs
namespace Carbon.Teams.Application.Interfaces
{
    public interface IAuditService
    {
        Task<Guid> WriteApprovalAuditAsync(ApprovalAuditWriteRequest request, CancellationToken ct = default);
        Task<AuditVerificationResult> VerifyAsync(Guid id, CancellationToken ct = default);
    }
}
```

```csharp
// IValidationAlertService.cs
namespace Carbon.Teams.Application.Interfaces
{
    public interface IValidationAlertService
    {
        Task<ValidationAlertResult> CreateOrUpdateAsync(CreateValidationAlertRequest request, CancellationToken ct = default);
        Task<ValidationAlertResult> MarkResolvedAsync(string issueKey, string companyId, CancellationToken ct = default);
    }
}
```

```csharp
// IStaleCardValidationService.cs
namespace Carbon.Teams.Application.Interfaces
{
    public interface IStaleCardValidationService
    {
        Task<StaleValidationResult> ValidateAsync(ApprovalCardValidationRequest request, CancellationToken ct = default);
    }
}
```

---

## 10. Infrastructure Layer Implementation

### 10.1 Graph Provisioning Service

```csharp
// Carbon.Teams.Infrastructure/Graph/GraphTeamsProvisioningService.cs
using Microsoft.Graph;
using Microsoft.Graph.Models;
using Microsoft.Graph.Teams.Item.Channels;

namespace Carbon.Teams.Infrastructure.Graph
{
    public class GraphTeamsProvisioningService : IGraphTeamsProvisioningService
    {
        private readonly IGraphClientFactory _graphClientFactory;
        private readonly ILogger<GraphTeamsProvisioningService> _logger;

        public GraphTeamsProvisioningService(
            IGraphClientFactory graphClientFactory,
            ILogger<GraphTeamsProvisioningService> logger)
        {
            _graphClientFactory = graphClientFactory;
            _logger = logger;
        }

        public async Task<string> CreateTeamAsync(GraphCreateTeamRequest request, CancellationToken ct = default)
        {
            _logger.LogInformation("Creating Teams team for company {CompanyId}", request.CompanyId);
            var graphClient = _graphClientFactory.CreateForTenant(request.CustomerTenantId);

            var team = new Team
            {
                DisplayName = request.TeamDisplayName,
                Description = request.TeamDescription,
                Visibility = TeamVisibilityType.Private,
                MemberSettings = new TeamMemberSettings
                {
                    AllowCreateUpdateChannels = false,
                    AllowDeleteChannels = false
                },
                MessagingSettings = new TeamMessagingSettings
                {
                    AllowUserEditMessages = false,
                    AllowUserDeleteMessages = false
                },
                Members = request.OwnerUpns.Select(upn => new AadUserConversationMember
                {
                    OdataType = "#microsoft.graph.aadUserConversationMember",
                    Roles = new List<string> { "owner" },
                    AdditionalData = new Dictionary<string, object>
                    {
                        { "user@odata.bind", $"https://graph.microsoft.com/v1.0/users('{upn}')" }
                    }
                }).Cast<ConversationMember>().ToList()
            };

            var createdTeam = await graphClient.Teams.PostAsync(team, cancellationToken: ct);

            if (createdTeam?.Id == null)
                throw new InvalidOperationException("Graph did not return a Team ID after creation.");

            _logger.LogInformation("Team created: {TeamId}", createdTeam.Id);
            return createdTeam.Id;
        }

        public async Task<string> CreateChannelAsync(GraphCreateChannelRequest request, CancellationToken ct = default)
        {
            _logger.LogInformation("Creating channel {ChannelName} in team {TeamId}", request.ChannelName, request.TeamId);
            var graphClient = _graphClientFactory.CreateForTenant(request.CustomerTenantId);

            var channel = new Channel
            {
                DisplayName = request.ChannelName,
                Description = request.Description,
                MembershipType = ChannelMembershipType.Standard
            };

            var created = await graphClient.Teams[request.TeamId].Channels.PostAsync(channel, cancellationToken: ct);

            if (created?.Id == null)
                throw new InvalidOperationException("Graph did not return a Channel ID after creation.");

            _logger.LogInformation("Channel created: {ChannelId}", created.Id);
            return created.Id;
        }

        public async Task AddMembersAsync(GraphAddMembersRequest request, CancellationToken ct = default)
        {
            var graphClient = _graphClientFactory.CreateForTenant(request.CustomerTenantId);
            foreach (var upn in request.MemberUpns)
            {
                var member = new AadUserConversationMember
                {
                    OdataType = "#microsoft.graph.aadUserConversationMember",
                    Roles = new List<string>(),
                    AdditionalData = new Dictionary<string, object>
                    {
                        { "user@odata.bind", $"https://graph.microsoft.com/v1.0/users('{upn}')" }
                    }
                };
                await graphClient.Teams[request.TeamId].Members.PostAsync(member, cancellationToken: ct);
            }
        }
    }
}
```

### 10.2 Graph Client Registration

```csharp
// Carbon.Teams.Infrastructure/DependencyInjection/InfrastructureServiceExtensions.cs
using Azure.Identity;
using Microsoft.Graph;

namespace Carbon.Teams.Infrastructure.DependencyInjection
{
    public static class InfrastructureServiceExtensions
    {
        public static IServiceCollection AddInfrastructure(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            // MongoDB
            MongoBsonConfiguration.Register();
            services.AddSingleton<IMongoClient>(sp =>
                new MongoClient(configuration.GetConnectionString("MongoDb")));
            services.AddSingleton<IMongoDatabase>(sp =>
                sp.GetRequiredService<IMongoClient>().GetDatabase("CarbonTeamsDb"));
            services.AddSingleton<MongoDbContext>();

            // Microsoft Graph (multi-tenant — per-tenant client factory)
            services.AddSingleton<IGraphClientFactory, GraphClientFactory>();

            // Repositories
            services.AddScoped<IOrgChannelMappingRepository, OrgChannelMappingRepository>();
            services.AddScoped<IApprovalCardInstanceRepository, ApprovalCardInstanceRepository>();
            services.AddScoped<IAuditRepository, AuditRepository>();
            services.AddScoped<IValidationAlertRepository, ValidationAlertRepository>();

            // Graph services
            services.AddScoped<IGraphTeamsProvisioningService, GraphTeamsProvisioningService>();

            // Bot card service
            services.AddScoped<IBotMessageService, BotMessageService>();

            // Audit hash service
            services.AddScoped<IAuditHashService, AuditHashService>();

            return services;
        }
    }
}
```

### 10.2.1 GraphClientFactory (Multi-Tenant)

```csharp
// Carbon.Teams.Infrastructure/Graph/GraphClientFactory.cs
using Azure.Identity;
using Microsoft.Graph;

namespace Carbon.Teams.Infrastructure.Graph
{
    public interface IGraphClientFactory
    {
        GraphServiceClient CreateForTenant(string customerTenantId);
    }

    public class GraphClientFactory : IGraphClientFactory
    {
        private readonly IConfiguration _configuration;

        public GraphClientFactory(IConfiguration configuration)
        {
            _configuration = configuration;
        }

        public GraphServiceClient CreateForTenant(string customerTenantId)
        {
            // Acquires token scoped to the CUSTOMER'S tenant authority
            var credential = new ClientSecretCredential(
                tenantId: customerTenantId,
                clientId: _configuration["AzureAd:ClientId"],
                clientSecret: _configuration["AzureAd:ClientSecret"]
            );
            return new GraphServiceClient(credential);
        }
    }
}
```

> **Production:** Replace `ClientSecretCredential` with `ClientCertificateCredential` using a cert loaded from Key Vault — no expiry risk and stronger security.

### 10.3 Audit Hash Service

```csharp
// Carbon.Teams.Infrastructure/Security/AuditHashService.cs
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

namespace Carbon.Teams.Infrastructure.Security
{
    public class AuditHashService : IAuditHashService
    {
        private readonly string _hmacSecret;

        public AuditHashService(IConfiguration configuration)
        {
            _hmacSecret = configuration["AuditHmacSecret"]
                ?? throw new InvalidOperationException("AuditHmacSecret is not configured.");
        }

        public string ComputeHash(object payload)
        {
            // Serialize to canonical sorted JSON
            var options = new JsonSerializerOptions { WriteIndented = false };
            var json = JsonSerializer.Serialize(payload, options);
            var canonical = json + _hmacSecret;

            var bytes = Encoding.UTF8.GetBytes(canonical);
            var hash = SHA256.HashData(bytes);
            return Convert.ToHexString(hash).ToLowerInvariant();
        }

        public bool Verify(object payload, string storedHash)
        {
            var computed = ComputeHash(payload);
            return string.Equals(computed, storedHash, StringComparison.OrdinalIgnoreCase);
        }
    }
}
```

### 10.4 Bot Message Service

```csharp
// Carbon.Teams.Infrastructure/Bot/BotMessageService.cs
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Integration.AspNet.Core;
using Microsoft.Bot.Schema;

namespace Carbon.Teams.Infrastructure.Bot
{
    public class BotMessageService : IBotMessageService
    {
        private readonly IBotFrameworkHttpAdapter _adapter;
        private readonly IConfiguration _configuration;
        private readonly ILogger<BotMessageService> _logger;

        public BotMessageService(
            IBotFrameworkHttpAdapter adapter,
            IConfiguration configuration,
            ILogger<BotMessageService> logger)
        {
            _adapter = adapter;
            _configuration = configuration;
            _logger = logger;
        }

        public async Task<string> PostProactiveCardAsync(
            string serviceUrl,
            string conversationId,
            string tenantId,
            Attachment cardAttachment,
            CancellationToken ct = default)
        {
            string? messageId = null;

            var conversationReference = new ConversationReference
            {
                ServiceUrl = serviceUrl,
                Conversation = new ConversationAccount { Id = conversationId, TenantId = tenantId },
                Bot = new ChannelAccount { Id = _configuration["BotAppId"] }
            };

            await ((BotAdapter)_adapter).ContinueConversationAsync(
                botId: _configuration["BotAppId"]!,
                reference: conversationReference,
                callback: async (turnContext, cancellationToken) =>
                {
                    var message = MessageFactory.Attachment(cardAttachment);
                    var response = await turnContext.SendActivityAsync(message, cancellationToken);
                    messageId = response.Id;
                },
                cancellationToken: ct);

            return messageId ?? throw new InvalidOperationException("No message ID returned from bot post.");
        }

        public async Task UpdateCardAsync(
            string serviceUrl,
            string conversationId,
            string tenantId,
            string messageId,
            Attachment cardAttachment,
            CancellationToken ct = default)
        {
            var conversationReference = new ConversationReference
            {
                ServiceUrl = serviceUrl,
                Conversation = new ConversationAccount { Id = conversationId, TenantId = tenantId },
                Bot = new ChannelAccount { Id = _configuration["BotAppId"] }
            };

            await ((BotAdapter)_adapter).ContinueConversationAsync(
                botId: _configuration["BotAppId"]!,
                reference: conversationReference,
                callback: async (turnContext, cancellationToken) =>
                {
                    var message = MessageFactory.Attachment(cardAttachment);
                    message.Id = messageId;
                    await turnContext.UpdateActivityAsync(message, cancellationToken);
                },
                cancellationToken: ct);
        }
    }
}
```

---

## 11. Application Layer Implementation

### 11.1 Teams Provisioning Service

```csharp
// Carbon.Teams.Application/Services/TeamsProvisioningService.cs
namespace Carbon.Teams.Application.Services
{
    public class TeamsProvisioningService : ITeamsProvisioningService
    {
        private readonly IGraphTeamsProvisioningService _graphService;
        private readonly IOrgChannelMappingRepository _mappingRepo;
        private readonly IProvisionedTeamRepository _teamRepo;
        private readonly IProvisionedChannelRepository _channelRepo;
        private readonly ILogger<TeamsProvisioningService> _logger;

        public TeamsProvisioningService(
            IGraphTeamsProvisioningService graphService,
            IOrgChannelMappingRepository mappingRepo,
            IProvisionedTeamRepository teamRepo,
            IProvisionedChannelRepository channelRepo,
            ILogger<TeamsProvisioningService> logger)
        {
            _graphService = graphService;
            _mappingRepo = mappingRepo;
            _teamRepo = teamRepo;
            _channelRepo = channelRepo;
            _logger = logger;
        }

        public async Task<TeamProvisionResult> CreateTeamAsync(CreateTeamRequest request, CancellationToken ct = default)
        {
            _logger.LogInformation("Provisioning team for company {CompanyId}", request.CompanyId);

            var graphRequest = new GraphCreateTeamRequest
            {
                CompanyId = request.CompanyId,
                TeamDisplayName = request.TeamDisplayName,
                TeamDescription = request.TeamDescription,
                OwnerUpns = request.Owners
            };

            var teamId = await _graphService.CreateTeamAsync(graphRequest, ct);

            if (request.Members.Any())
            {
                await _graphService.AddMembersAsync(new GraphAddMembersRequest
                {
                    TeamId = teamId,
                    MemberUpns = request.Members
                }, ct);
            }

            var provisionedTeam = new ProvisionedTeam
            {
                Id = Guid.NewGuid(),
                CompanyId = request.CompanyId,
                TeamId = teamId,
                TeamDisplayName = request.TeamDisplayName,
                TenantId = request.TenantId,
                ProvisioningStatus = "Created",
                CreatedUtc = DateTime.UtcNow,
                UpdatedUtc = DateTime.UtcNow
            };

            await _teamRepo.AddAsync(provisionedTeam, ct);

            return new TeamProvisionResult
            {
                CompanyId = request.CompanyId,
                TeamId = teamId,
                Status = "Created"
            };
        }

        public async Task<ChannelProvisionResult> CreateChannelAsync(CreateChannelRequest request, CancellationToken ct = default)
        {
            var channelId = await _graphService.CreateChannelAsync(new GraphCreateChannelRequest
            {
                TeamId = request.TeamId,
                ChannelName = request.ChannelName,
                Description = request.Description
            }, ct);

            await _channelRepo.AddAsync(new ProvisionedChannel
            {
                Id = Guid.NewGuid(),
                CompanyId = request.CompanyId,
                TeamId = request.TeamId,
                ChannelId = channelId,
                ChannelName = request.ChannelName,
                TenantId = request.TenantId,
                IsDefaultApprovalChannel = true,
                CreatedUtc = DateTime.UtcNow,
                UpdatedUtc = DateTime.UtcNow
            }, ct);

            return new ChannelProvisionResult
            {
                CompanyId = request.CompanyId,
                TeamId = request.TeamId,
                ChannelId = channelId,
                Status = "Created"
            };
        }

        public async Task<OrgChannelMappingResult> MapOrganizationChannelAsync(MapOrgChannelRequest request, CancellationToken ct = default)
        {
            await _mappingRepo.DeactivateExistingAsync(request.CompanyId, ct);

            await _mappingRepo.AddAsync(new OrganizationChannelMapping
            {
                Id = Guid.NewGuid(),
                CompanyId = request.CompanyId,
                TeamId = request.TeamId,
                ChannelId = request.ChannelId,
                TenantId = request.TenantId,
                ConversationId = request.ConversationId,
                ServiceUrl = request.ServiceUrl,
                IsActive = true,
                CreatedUtc = DateTime.UtcNow,
                UpdatedUtc = DateTime.UtcNow
            }, ct);

            return new OrgChannelMappingResult { CompanyId = request.CompanyId, Status = "Mapped" };
        }

        public async Task DeactivateMappingAsync(string companyId, CancellationToken ct = default)
        {
            await _mappingRepo.DeactivateExistingAsync(companyId, ct);
        }
    }
}
```

### 11.2 Approval Action Service

```csharp
// Carbon.Teams.Application/Services/ApprovalActionService.cs
namespace Carbon.Teams.Application.Services
{
    public class ApprovalActionService : IApprovalActionService
    {
        private readonly IIdentityValidationService _identityService;
        private readonly IAuthorizationService _authService;
        private readonly IStaleCardValidationService _staleValidator;
        private readonly IAuditService _auditService;
        private readonly IApprovalCardPostingService _cardService;
        private readonly IApprovalCardInstanceRepository _cardRepo;
        private readonly ILogger<ApprovalActionService> _logger;

        public ApprovalActionService(
            IIdentityValidationService identityService,
            IAuthorizationService authService,
            IStaleCardValidationService staleValidator,
            IAuditService auditService,
            IApprovalCardPostingService cardService,
            IApprovalCardInstanceRepository cardRepo,
            ILogger<ApprovalActionService> logger)
        {
            _identityService = identityService;
            _authService = authService;
            _staleValidator = staleValidator;
            _auditService = auditService;
            _cardService = cardService;
            _cardRepo = cardRepo;
            _logger = logger;
        }

        public async Task<ApprovalActionResult> HandleApprovalActionAsync(
            ApprovalActionCommand command,
            CancellationToken ct = default)
        {
            // 1. Validate identity
            var identityCtx = await _identityService.ValidateAndResolveIdentityAsync(command.TeamsContext, ct);
            if (!identityCtx.IsValid)
                return ApprovalActionResult.Unauthorized("Identity could not be validated.");

            // 2. Validate RBAC
            var authResult = await _authService.CanApproveSectionAsync(
                identityCtx.UserId, command.CompanyId, command.SectionId, ct);

            if (!authResult.IsAuthorized)
                return ApprovalActionResult.Unauthorized("User is not authorized to approve this section.");

            // 3. Validate stale state (version hash, workflow state, card status)
            var staleResult = await _staleValidator.ValidateAsync(new ApprovalCardValidationRequest
            {
                CardInstanceId = command.CardInstanceId,
                SectionId = command.SectionId,
                DocumentId = command.DocumentId,
                DocumentVersion = command.DocumentVersion,
                SectionVersionHash = command.SectionVersionHash
            }, ct);

            if (!staleResult.IsValid)
            {
                await _cardService.MarkCardStaleAsync(command.CardInstanceId, ct);
                return ApprovalActionResult.Stale(staleResult.Reason);
            }

            // 4. Write audit (insert-only)
            await _auditService.WriteApprovalAuditAsync(new ApprovalAuditWriteRequest
            {
                ApproverUserId = identityCtx.UserId,
                DisplayName = identityCtx.DisplayName,
                TenantId = identityCtx.TenantId,
                Decision = command.Decision,
                RejectReason = command.RejectReason,
                SectionId = command.SectionId,
                DocumentId = command.DocumentId,
                DocumentVersion = command.DocumentVersion,
                SectionVersionHash = command.SectionVersionHash,
                PreviousState = staleResult.CurrentWorkflowState,
                NewState = command.Decision == "Approve" ? "APPROVED" : "REJECTED",
                CorrelationId = command.CorrelationId,
                TeamsConversationId = command.TeamsContext.ConversationId,
                TeamsMessageId = command.TeamsContext.MessageId,
                CardInstanceId = command.CardInstanceId
            }, ct);

            // 5. Update card to completed state
            await _cardService.UpdateApprovalCardAsync(new UpdateApprovalCardRequest
            {
                CardInstanceId = command.CardInstanceId,
                Decision = command.Decision,
                DecidedByDisplayName = identityCtx.DisplayName,
                DecidedAtUtc = DateTime.UtcNow,
                RejectReason = command.RejectReason
            }, ct);

            _logger.LogInformation(
                "Approval action {Decision} processed for section {SectionId} by {UserId}",
                command.Decision, command.SectionId, identityCtx.UserId);

            return ApprovalActionResult.Success(command.Decision);
        }
    }
}
```

---

## 12. Bot Layer Implementation

### 12.1 CarbonTeamsBot

```csharp
// Carbon.Teams.Bot/Handlers/CarbonTeamsBot.cs
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Teams;
using Microsoft.Bot.Schema;
using Microsoft.Bot.Schema.Teams;

namespace Carbon.Teams.Bot.Handlers
{
    public class CarbonTeamsBot : TeamsActivityHandler
    {
        private readonly TeamsInvokeRouter _invokeRouter;
        private readonly IOrgChannelMappingRepository _mappingRepo;
        private readonly ILogger<CarbonTeamsBot> _logger;

        public CarbonTeamsBot(
            TeamsInvokeRouter invokeRouter,
            IOrgChannelMappingRepository mappingRepo,
            ILogger<CarbonTeamsBot> logger)
        {
            _invokeRouter = invokeRouter;
            _mappingRepo = mappingRepo;
            _logger = logger;
        }

        protected override async Task OnInstallationUpdateActivityAsync(
            ITurnContext<IInstallationUpdateActivity> turnContext,
            CancellationToken cancellationToken)
        {
            if (turnContext.Activity.Action == "add")
            {
                _logger.LogInformation("Bot installed in conversation {ConversationId}",
                    turnContext.Activity.Conversation.Id);

                var conversationRef = turnContext.Activity.GetConversationReference();

                // Persist ConversationReference keyed by TenantId + TeamId
                // so proactive posting can retrieve it later
                await _mappingRepo.UpdateConversationReferenceAsync(
                    tenantId: conversationRef.Conversation.TenantId ?? string.Empty,
                    conversationId: conversationRef.Conversation.Id,
                    serviceUrl: conversationRef.ServiceUrl ?? string.Empty,
                    cancellationToken: cancellationToken);

                _logger.LogInformation(
                    "ConversationReference stored for tenant {TenantId}",
                    conversationRef.Conversation.TenantId);
            }
        }

        protected override async Task<InvokeResponse> OnInvokeActivityAsync(
            ITurnContext<IInvokeActivity> turnContext,
            CancellationToken cancellationToken)
        {
            if (turnContext.Activity.Name == "adaptiveCard/action")
            {
                return await _invokeRouter.RouteAsync(turnContext, cancellationToken);
            }

            return new InvokeResponse { Status = 200 };
        }
    }
}
```

### 12.2 TeamsInvokeRouter

```csharp
// Carbon.Teams.Bot/Routing/TeamsInvokeRouter.cs
using Microsoft.Bot.Builder;
using Microsoft.Bot.Schema;
using System.Text.Json;

namespace Carbon.Teams.Bot.Routing
{
    public class TeamsInvokeRouter
    {
        private readonly IApprovalActionService _approvalActionService;
        private readonly ILogger<TeamsInvokeRouter> _logger;

        public TeamsInvokeRouter(
            IApprovalActionService approvalActionService,
            ILogger<TeamsInvokeRouter> logger)
        {
            _approvalActionService = approvalActionService;
            _logger = logger;
        }

        public async Task<InvokeResponse> RouteAsync(
            ITurnContext<IInvokeActivity> turnContext,
            CancellationToken ct)
        {
            var valueJson = turnContext.Activity.Value?.ToString();
            if (string.IsNullOrWhiteSpace(valueJson))
                return new InvokeResponse { Status = 400 };

            ApprovalActionPayload? payload;
            try
            {
                payload = JsonSerializer.Deserialize<ApprovalActionPayload>(valueJson,
                    new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }
            catch (JsonException ex)
            {
                _logger.LogWarning(ex, "Failed to parse invoke payload");
                return new InvokeResponse { Status = 400 };
            }

            if (payload == null)
                return new InvokeResponse { Status = 400 };

            var aadObjectId = turnContext.Activity.From?.AadObjectId;
            var displayName = turnContext.Activity.From?.Name;
            var tenantId = turnContext.Activity.Conversation?.TenantId;

            if (string.IsNullOrWhiteSpace(aadObjectId))
            {
                _logger.LogWarning("No AAD object ID in activity from");
                return new InvokeResponse { Status = 401 };
            }

            var command = new ApprovalActionCommand
            {
                CardInstanceId = payload.CardInstanceId,
                CompanyId = payload.CompanyId,
                SectionId = payload.SectionId,
                DocumentId = payload.DocumentId,
                DocumentVersion = payload.DocumentVersion,
                SectionVersionHash = payload.SectionVersionHash,
                Decision = payload.ActionType,
                RejectReason = payload.RejectReason,
                CorrelationId = Guid.NewGuid().ToString(),
                TeamsContext = new TeamsActionContext
                {
                    AadObjectId = aadObjectId,
                    DisplayName = displayName ?? string.Empty,
                    TenantId = tenantId ?? string.Empty,
                    ConversationId = turnContext.Activity.Conversation?.Id ?? string.Empty,
                    MessageId = turnContext.Activity.ReplyToId ?? string.Empty,
                    ServiceUrl = turnContext.Activity.ServiceUrl ?? string.Empty
                }
            };

            var result = await _approvalActionService.HandleApprovalActionAsync(command, ct);

            if (!result.IsSuccess)
            {
                _logger.LogWarning("Approval action failed: {Reason}", result.Reason);
                return new InvokeResponse
                {
                    Status = result.IsUnauthorized ? 403 : 400,
                    Body = new { message = result.Reason }
                };
            }

            return new InvokeResponse { Status = 200 };
        }
    }
}
```

### 12.3 Bot Controller Endpoint

```csharp
// Carbon.Teams.Bot/Controllers/BotController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Integration.AspNet.Core;

namespace Carbon.Teams.Bot.Controllers
{
    [ApiController]
    [Route("api/messages")]
    public class BotController : ControllerBase
    {
        private readonly IBotFrameworkHttpAdapter _adapter;
        private readonly IBot _bot;

        public BotController(IBotFrameworkHttpAdapter adapter, IBot bot)
        {
            _adapter = adapter;
            _bot = bot;
        }

        [HttpPost]
        public async Task PostAsync(CancellationToken ct)
        {
            await _adapter.ProcessAsync(Request, Response, _bot, ct);
        }
    }
}
```

---

## 13. API Layer Implementation

### 13.1 Program.cs (Carbon.Teams.Api)

```csharp
// Carbon.Teams.Api/Program.cs
using Carbon.Teams.Application.Services;
using Carbon.Teams.Infrastructure.DependencyInjection;
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Integration.AspNet.Core;
using Microsoft.Bot.Connector.Authentication;

var builder = WebApplication.CreateBuilder(args);

// Configuration — Key Vault in production
builder.Configuration
    .AddJsonFile("appsettings.json")
    .AddEnvironmentVariables();

if (builder.Environment.IsProduction())
{
    var keyVaultUri = builder.Configuration["KeyVaultUri"];
    if (!string.IsNullOrEmpty(keyVaultUri))
    {
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUri),
            new Azure.Identity.ManagedIdentityCredential());
    }
}

// Infrastructure
builder.Services.AddInfrastructure(builder.Configuration);

// Application services
builder.Services.AddScoped<ITeamsProvisioningService, TeamsProvisioningService>();
builder.Services.AddScoped<IApprovalCardPostingService, ApprovalCardPostingService>();
builder.Services.AddScoped<IApprovalActionService, ApprovalActionService>();
builder.Services.AddScoped<IAuditService, AuditService>();
builder.Services.AddScoped<IValidationAlertService, ValidationAlertService>();
builder.Services.AddScoped<IStaleCardValidationService, StaleCardValidationService>();
builder.Services.AddScoped<IIdentityValidationService, IdentityValidationService>();
builder.Services.AddScoped<IAuthorizationService, CarbonAuthorizationService>();

// Bot Framework
builder.Services.AddSingleton<BotFrameworkAuthentication, ConfigurationBotFrameworkAuthentication>();
builder.Services.AddSingleton<IBotFrameworkHttpAdapter, AdapterWithErrorHandler>();
builder.Services.AddTransient<IBot, CarbonTeamsBot>();

// Application Insights
builder.Services.AddApplicationInsightsTelemetry(builder.Configuration["ApplicationInsights:ConnectionString"]);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### 13.2 Provisioning Controller

```csharp
// Carbon.Teams.Api/Controllers/ProvisioningController.cs
using Microsoft.AspNetCore.Mvc;

namespace Carbon.Teams.Api.Controllers
{
    [ApiController]
    [Route("api/teams")]
    public class ProvisioningController : ControllerBase
    {
        private readonly ITeamsProvisioningService _provisioningService;
        private readonly ILogger<ProvisioningController> _logger;

        public ProvisioningController(
            ITeamsProvisioningService provisioningService,
            ILogger<ProvisioningController> logger)
        {
            _provisioningService = provisioningService;
            _logger = logger;
        }

        [HttpPost("provision/team")]
        public async Task<IActionResult> ProvisionTeam(
            [FromBody] CreateTeamRequest request,
            CancellationToken ct)
        {
            if (!ModelState.IsValid) return BadRequest(ModelState);

            var result = await _provisioningService.CreateTeamAsync(request, ct);
            return Ok(result);
        }

        [HttpPost("provision/channel")]
        public async Task<IActionResult> ProvisionChannel(
            [FromBody] CreateChannelRequest request,
            CancellationToken ct)
        {
            if (!ModelState.IsValid) return BadRequest(ModelState);

            var result = await _provisioningService.CreateChannelAsync(request, ct);
            return Ok(result);
        }

        [HttpPost("channels")]
        public async Task<IActionResult> MapOrgChannel(
            [FromBody] MapOrgChannelRequest request,
            CancellationToken ct)
        {
            if (!ModelState.IsValid) return BadRequest(ModelState);

            var result = await _provisioningService.MapOrganizationChannelAsync(request, ct);
            return Ok(result);
        }

        [HttpDelete("channels/{companyId}")]
        public async Task<IActionResult> DeactivateMapping(string companyId, CancellationToken ct)
        {
            await _provisioningService.DeactivateMappingAsync(companyId, ct);
            return NoContent();
        }
    }
}
```

### 13.3 Approval Cards Controller

```csharp
// Carbon.Teams.Api/Controllers/ApprovalCardsController.cs
using Microsoft.AspNetCore.Mvc;

namespace Carbon.Teams.Api.Controllers
{
    [ApiController]
    [Route("api/teams/cards")]
    public class ApprovalCardsController : ControllerBase
    {
        private readonly IApprovalCardPostingService _cardService;
        private readonly IValidationAlertService _alertService;

        public ApprovalCardsController(
            IApprovalCardPostingService cardService,
            IValidationAlertService alertService)
        {
            _cardService = cardService;
            _alertService = alertService;
        }

        [HttpPost("approval")]
        public async Task<IActionResult> PostApprovalCard(
            [FromBody] PostApprovalCardRequest request,
            CancellationToken ct)
        {
            if (!ModelState.IsValid) return BadRequest(ModelState);

            var result = await _cardService.PostApprovalCardAsync(request, ct);
            return Ok(result);
        }

        [HttpPost("validation-alert")]
        public async Task<IActionResult> PostValidationAlert(
            [FromBody] CreateValidationAlertRequest request,
            CancellationToken ct)
        {
            if (!ModelState.IsValid) return BadRequest(ModelState);

            var result = await _alertService.CreateOrUpdateAsync(request, ct);
            return Ok(result);
        }
    }
}
```

---

### 13.4 Admin Consent and Bot Auto-Installation Controller

This controller serves two purposes:
1. **Generate consent URL** — send this link to each customer admin (one-time).
2. **Handle consent callback** — on callback, persist the customer tenant and trigger automated Team + bot provisioning.

```csharp
// Carbon.Teams.Api/Controllers/ConsentController.cs
using Microsoft.AspNetCore.Mvc;

namespace Carbon.Teams.Api.Controllers
{
    [ApiController]
    [Route("api/consent")]
    public class ConsentController : ControllerBase
    {
        private readonly IConfiguration _configuration;
        private readonly ITenantOnboardingService _onboardingService;
        private readonly ILogger<ConsentController> _logger;

        public ConsentController(
            IConfiguration configuration,
            ITenantOnboardingService onboardingService,
            ILogger<ConsentController> logger)
        {
            _configuration = configuration;
            _onboardingService = onboardingService;
            _logger = logger;
        }

        // GET /api/consent/url?companyId=COMP-001&customerTenantId=<GUID>
        // Returns the admin consent URL to send to the customer admin
        [HttpGet("url")]
        public IActionResult GetConsentUrl(
            [FromQuery] string companyId,
            [FromQuery] string customerTenantId)
        {
            var clientId = _configuration["AzureAd:ClientId"];
            var appBase = _configuration["AppBaseUrl"];
            var redirectUri = Uri.EscapeDataString($"{appBase}/api/consent/callback");
            var state = Uri.EscapeDataString(companyId);

            var consentUrl =
                $"https://login.microsoftonline.com/{customerTenantId}/adminconsent" +
                $"?client_id={clientId}" +
                $"&redirect_uri={redirectUri}" +
                $"&state={state}";

            return Ok(new { consentUrl, companyId, customerTenantId });
        }

        // GET /api/consent/callback  (Entra redirects here after admin clicks Accept)
        [HttpGet("callback")]
        public async Task<IActionResult> ConsentCallback(
            [FromQuery] string? tenant,
            [FromQuery] string? state,
            [FromQuery] string? error,
            [FromQuery] string? error_description,
            CancellationToken ct)
        {
            if (!string.IsNullOrEmpty(error))
            {
                _logger.LogWarning("Consent denied by admin: {Error} — {Desc}", error, error_description);
                return BadRequest(new { error, error_description });
            }

            var companyId = Uri.UnescapeDataString(state ?? string.Empty);
            var customerTenantId = tenant ?? string.Empty;

            if (string.IsNullOrEmpty(companyId) || string.IsNullOrEmpty(customerTenantId))
                return BadRequest(new { message = "Missing companyId or tenantId in callback." });

            _logger.LogInformation(
                "Admin consent granted. Onboarding company {CompanyId} tenant {TenantId}",
                companyId, customerTenantId);

            // Trigger full automated provisioning:
            // 1. Store customer tenant mapping
            // 2. Create Teams Team in customer tenant via Graph
            // 3. Create approval channel
            // 4. Auto-install bot in the team (triggers OnInstallationUpdateActivityAsync)
            await _onboardingService.CompleteConsentAsync(companyId, customerTenantId, ct);

            return Ok(new
            {
                message = "Consent accepted. Team, channel, and bot are being provisioned automatically.",
                companyId,
                tenantId = customerTenantId
            });
        }
    }
}
```

### 13.5 Tenant Onboarding Service

```csharp
// Carbon.Teams.Application/Interfaces/ITenantOnboardingService.cs
namespace Carbon.Teams.Application.Interfaces
{
    public interface ITenantOnboardingService
    {
        Task CompleteConsentAsync(string companyId, string customerTenantId, CancellationToken ct = default);
    }
}
```

```csharp
// Carbon.Teams.Application/Services/TenantOnboardingService.cs
namespace Carbon.Teams.Application.Services
{
    public class TenantOnboardingService : ITenantOnboardingService
    {
        private readonly ITeamsProvisioningService _provisioningService;
        private readonly IGraphBotInstallService _botInstallService;
        private readonly IConfiguration _configuration;
        private readonly ILogger<TenantOnboardingService> _logger;

        public TenantOnboardingService(
            ITeamsProvisioningService provisioningService,
            IGraphBotInstallService botInstallService,
            IConfiguration configuration,
            ILogger<TenantOnboardingService> logger)
        {
            _provisioningService = provisioningService;
            _botInstallService = botInstallService;
            _configuration = configuration;
            _logger = logger;
        }

        public async Task CompleteConsentAsync(
            string companyId,
            string customerTenantId,
            CancellationToken ct = default)
        {
            _logger.LogInformation("Starting automated onboarding for company {CompanyId}", companyId);

            // Step 1: Provision Team in customer tenant
            var teamResult = await _provisioningService.CreateTeamAsync(new CreateTeamRequest
            {
                CompanyId = companyId,
                TenantId = customerTenantId,
                TeamDisplayName = $"IRIS CARBON — {companyId}",
                TeamDescription = "IRIS CARBON disclosure approval workspace",
                Owners = new List<string>(), // populated from CARBON org config
                Members = new List<string>()
            }, ct);

            // Step 2: Create approval channel
            var channelResult = await _provisioningService.CreateChannelAsync(new CreateChannelRequest
            {
                CompanyId = companyId,
                TenantId = customerTenantId,
                TeamId = teamResult.TeamId,
                ChannelName = "carbon-approvals",
                Description = "IRIS CARBON approval workflow channel"
            }, ct);

            // Step 3: Auto-install bot in the Team
            // This triggers OnInstallationUpdateActivityAsync which stores ConversationReference
            await _botInstallService.InstallBotInTeamAsync(customerTenantId, teamResult.TeamId, ct);

            _logger.LogInformation(
                "Onboarding complete for company {CompanyId} — TeamId {TeamId} ChannelId {ChannelId}",
                companyId, teamResult.TeamId, channelResult.ChannelId);
        }
    }
}
```

### 13.6 Bot Auto-Installation via Graph

```csharp
// Carbon.Teams.Infrastructure/Graph/GraphBotInstallService.cs
using Microsoft.Graph.Models;

namespace Carbon.Teams.Infrastructure.Graph
{
    public interface IGraphBotInstallService
    {
        Task InstallBotInTeamAsync(string customerTenantId, string teamId, CancellationToken ct = default);
    }

    public class GraphBotInstallService : IGraphBotInstallService
    {
        private readonly IGraphClientFactory _graphClientFactory;
        private readonly IConfiguration _configuration;
        private readonly ILogger<GraphBotInstallService> _logger;

        public GraphBotInstallService(
            IGraphClientFactory graphClientFactory,
            IConfiguration configuration,
            ILogger<GraphBotInstallService> logger)
        {
            _graphClientFactory = graphClientFactory;
            _configuration = configuration;
            _logger = logger;
        }

        public async Task InstallBotInTeamAsync(
            string customerTenantId,
            string teamId,
            CancellationToken ct = default)
        {
            _logger.LogInformation(
                "Auto-installing bot in team {TeamId} for tenant {TenantId}",
                teamId, customerTenantId);

            var graphClient = _graphClientFactory.CreateForTenant(customerTenantId);

            // teamsAppCatalogId: use your AppSource global catalog ID
            // OR the org-catalog ID returned after uploading the manifest package
            var teamsAppCatalogId = _configuration["TeamsApp:CatalogId"];

            var installation = new TeamsAppInstallation
            {
                AdditionalData = new Dictionary<string, object>
                {
                    {
                        "teamsApp@odata.bind",
                        $"https://graph.microsoft.com/v1.0/appCatalogs/teamsApps/{teamsAppCatalogId}"
                    }
                }
            };

            await graphClient.Teams[teamId].InstalledApps.PostAsync(
                installation, cancellationToken: ct);

            // After this call succeeds, Teams delivers an installation event
            // to the bot endpoint -> OnInstallationUpdateActivityAsync fires
            // -> ConversationReference is persisted automatically
            _logger.LogInformation("Bot installation triggered for team {TeamId}", teamId);
        }
    }
}
```

Register in `InfrastructureServiceExtensions`:
```csharp
services.AddScoped<IGraphBotInstallService, GraphBotInstallService>();
services.AddScoped<ITenantOnboardingService, TenantOnboardingService>();
```

Add `TeamsApp:CatalogId` and `AppBaseUrl` to `appsettings.json`:
```json
"TeamsApp": {
  "CatalogId": "<YOUR_APPSTORE_OR_ORG_CATALOG_APP_ID>"
},
"AppBaseUrl": "https://<APP_SERVICE_NAME>.azurewebsites.net"
```

---

## 14. Key Vault and Configuration Wiring

### 14.1 appsettings.json (Dev — no secrets)

```json
{
  "AzureAd": {
    "TenantId": "common",
    "ClientId": "<BOT_APP_ID>",
    "ClientSecret": ""
  },
  "BotAppId": "<BOT_APP_ID>",
  "MicrosoftAppType": "MultiTenant",
  "KeyVaultUri": "https://<KV_NAME>.vault.azure.net/",
  "ApplicationInsights": {
    "ConnectionString": ""
  },
  "ConnectionStrings": {
    "MongoDb": ""
  }
}
```

### 14.2 Key Vault RBAC for Managed Identity

```bash
# Get the App Service Managed Identity principalId
PRINCIPAL_ID=$(az webapp identity show \
  --name <APP_SERVICE_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --query principalId -o tsv)

# Get Key Vault resource ID
KV_ID=$(az keyvault show \
  --name <KV_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --query id -o tsv)

# Assign Key Vault Secrets User role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope $KV_ID
```

### 14.3 App Service Application Settings

```bash
az webapp config appsettings set \
  --name <APP_SERVICE_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --settings \
    "AzureAd__TenantId=common" \
    "AzureAd__ClientId=<BOT_APP_ID>" \
    "BotAppId=<BOT_APP_ID>" \
    "MicrosoftAppType=MultiTenant" \
    "KeyVaultUri=https://<KV_NAME>.vault.azure.net/" \
    "ApplicationInsights__ConnectionString=@Microsoft.KeyVault(VaultName=<KV_NAME>;SecretName=AppInsightsConnectionString)" \
    "ConnectionStrings__MongoDb=@Microsoft.KeyVault(VaultName=<KV_NAME>;SecretName=MongoDbConnectionString)" \
    "AzureAd__ClientSecret=@Microsoft.KeyVault(VaultName=<KV_NAME>;SecretName=BotClientSecret)"
```

---

## 15. Adaptive Card JSON Templates

### 15.1 Approval Card (Pending State)

```json
{
  "type": "AdaptiveCard",
  "version": "1.5",
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "body": [
    {
      "type": "TextBlock",
      "text": "Approval Required",
      "weight": "Bolder",
      "size": "Large",
      "color": "Accent"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Section", "value": "${sectionName}" },
        { "title": "Document", "value": "${documentId} v${documentVersion}" },
        { "title": "Last Editor", "value": "${lastEditor}" },
        { "title": "Last Edited", "value": "${lastEditedUtc}" },
        { "title": "Status", "value": "${workflowState}" }
      ]
    },
    {
      "type": "TextBlock",
      "text": "Summary of Changes",
      "weight": "Bolder",
      "spacing": "Medium"
    },
    {
      "type": "TextBlock",
      "text": "${diffSummary}",
      "wrap": true
    }
  ],
  "actions": [
    {
      "type": "Action.Execute",
      "title": "Approve",
      "style": "positive",
      "data": {
        "actionType": "Approve",
        "companyId": "${companyId}",
        "sectionId": "${sectionId}",
        "documentId": "${documentId}",
        "documentVersion": "${documentVersion}",
        "sectionVersionHash": "${sectionVersionHash}",
        "cardInstanceId": "${cardInstanceId}",
        "issuedAtUtc": "${issuedAtUtc}"
      }
    },
    {
      "type": "Action.ShowCard",
      "title": "Reject",
      "style": "destructive",
      "card": {
        "type": "AdaptiveCard",
        "body": [
          {
            "type": "Input.Text",
            "id": "rejectReason",
            "label": "Reason for rejection",
            "isRequired": true,
            "placeholder": "Enter reason..."
          }
        ],
        "actions": [
          {
            "type": "Action.Execute",
            "title": "Confirm Reject",
            "style": "destructive",
            "data": {
              "actionType": "Reject",
              "companyId": "${companyId}",
              "sectionId": "${sectionId}",
              "documentId": "${documentId}",
              "documentVersion": "${documentVersion}",
              "sectionVersionHash": "${sectionVersionHash}",
              "cardInstanceId": "${cardInstanceId}"
            }
          }
        ]
      }
    },
    {
      "type": "Action.OpenUrl",
      "title": "Open in CARBON",
      "url": "https://app.iriscarbon.com/sections/${sectionId}"
    }
  ]
}
```

### 15.2 Completed Card State (Post-Action)

```json
{
  "type": "AdaptiveCard",
  "version": "1.5",
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "body": [
    {
      "type": "TextBlock",
      "text": "${decision} — ${sectionName}",
      "weight": "Bolder",
      "size": "Medium",
      "color": "${decision == 'Approved' ? 'Good' : 'Attention'}"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Decision", "value": "${decision}" },
        { "title": "By", "value": "${decidedByDisplayName}" },
        { "title": "At", "value": "${decidedAtUtc}" },
        { "title": "Reason", "value": "${rejectReason}" }
      ]
    }
  ]
}
```

### 15.3 Store Card Templates in Infrastructure

```csharp
// Carbon.Teams.Infrastructure/Cards/CardTemplateProvider.cs
namespace Carbon.Teams.Infrastructure.Cards
{
    public class CardTemplateProvider : ICardTemplateProvider
    {
        private readonly IWebHostEnvironment _env;

        public CardTemplateProvider(IWebHostEnvironment env)
        {
            _env = env;
        }

        public string GetApprovalCardTemplate()
        {
            var path = Path.Combine(_env.ContentRootPath, "Cards", "approval-card.json");
            return File.ReadAllText(path);
        }

        public string GetCompletedCardTemplate()
        {
            var path = Path.Combine(_env.ContentRootPath, "Cards", "completed-card.json");
            return File.ReadAllText(path);
        }
    }
}
```

---

## 16. Local Development and Testing

### 16.1 Local Configuration

Create `appsettings.Development.json` (never commit to Git):

```json
{
  "AzureAd": {
    "TenantId": "<TENANT_ID>",
    "ClientId": "<BOT_APP_ID>",
    "ClientSecret": "<DEV_SECRET>"
  },
  "BotAppId": "<BOT_APP_ID>",
  "MicrosoftAppType": "MultiTenant",
  "ConnectionStrings": {
    "MongoDb": "mongodb://localhost:27017"
  },
  "ApplicationInsights": {
    "ConnectionString": ""
  }
}
```

Add to `.gitignore`:

```
appsettings.Development.json
*.pfx
*.pem
*.key
secrets/
```

### 16.2 ngrok Tunnel for Local Bot Testing

```bash
# Start your API
dotnet run --project Carbon.Teams.Api

# In another terminal, start ngrok
ngrok http 5000

# Copy the HTTPS URL (e.g. https://abc123.ngrok.io)
# Update Azure Bot messaging endpoint to: https://abc123.ngrok.io/api/messages
```

### 16.3 Bot Framework Emulator

```
1. Open Bot Framework Emulator
2. Click Open Bot
3. Bot URL: http://localhost:5000/api/messages
4. Microsoft App ID: <BOT_APP_ID>
5. Microsoft App Password: <BOT_CLIENT_SECRET>
6. Click Connect
```

### 16.4 Run Tests

```bash
dotnet test Carbon.Teams.Tests/Carbon.Teams.Tests.csproj --verbosity normal
```

---

## 17. CI/CD Pipeline (Azure DevOps)

### 17.1 azure-pipelines.yml

```yaml
trigger:
  branches:
    include:
      - main
      - release/*

pool:
  vmImage: windows-latest

variables:
  buildConfiguration: Release
  appServiceName: $(APP_SERVICE_NAME)
  resourceGroup: $(RESOURCE_GROUP)

stages:
  - stage: Build
    displayName: Build and Test
    jobs:
      - job: Build
        steps:
          - task: UseDotNet@2
            inputs:
              version: 8.x

          - script: dotnet restore Carbon.Teams.sln
            displayName: Restore

          - script: dotnet build Carbon.Teams.sln -c $(buildConfiguration) --no-restore
            displayName: Build

          - script: dotnet test Carbon.Teams.Tests/Carbon.Teams.Tests.csproj -c $(buildConfiguration) --no-build
            displayName: Tests

          - task: DotNetCoreCLI@2
            displayName: Publish API
            inputs:
              command: publish
              publishWebProjects: false
              projects: Carbon.Teams.Api/Carbon.Teams.Api.csproj
              arguments: -c $(buildConfiguration) -o $(Build.ArtifactStagingDirectory)/api

          - publish: $(Build.ArtifactStagingDirectory)/api
            artifact: drop

  - stage: Deploy_Prod
    displayName: Deploy to Production
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployProd
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  displayName: Deploy to App Service
                  inputs:
                    azureSubscription: $(AZURE_SERVICE_CONNECTION)
                    appType: webApp
                    appName: $(appServiceName)
                    package: $(Pipeline.Workspace)/drop/**/*.zip

                # MongoDB indexes are created automatically at application startup
                # via MongoIndexInitializer — no migration step required here
```

---

## 18. Production Deployment Checklist

### 18.1 Azure Infrastructure

- [ ] Resource group created in target region
- [ ] App Registration created with correct permissions
- [ ] Admin consent granted for all Graph permissions
- [ ] Azure Bot resource created and Teams channel enabled
- [ ] App Service Plan P1v3 or higher
- [ ] App Service created with .NET 8 runtime
- [ ] System-assigned Managed Identity enabled on App Service
- [ ] Azure Cosmos DB for MongoDB API account and database created
- [ ] Azure Key Vault created with RBAC enabled
- [ ] Managed Identity granted `Key Vault Secrets User` role on KV
- [ ] All secrets stored in Key Vault
- [ ] App Service configured with Key Vault references
- [ ] Application Insights created and connection string in KV
- [ ] App Service messaging endpoint set in Azure Bot
- [ ] HTTPS only enforced on App Service
- [ ] TLS 1.2 minimum on App Service
- [ ] Cosmos DB firewall configured: App Service IPs allowed or Private Endpoint enabled

### 18.2 Application Configuration

- [ ] `appsettings.Production.json` has no embedded secrets
- [ ] All sensitive config reads from Key Vault via managed identity
- [ ] Bot App ID and tenant ID correct in app settings
- [ ] MongoDB indexes created at startup (MongoIndexInitializer runs automatically on app start)
- [ ] Unique partial indexes verified via MongoDB Compass or Azure Cosmos DB Data Explorer
- [ ] Application Insights SDK integrated and telemetry verified

### 18.3 Teams App

- [ ] Teams app manifest `id` matches `<BOT_APP_ID>`
- [ ] Manifest uploaded to Teams Admin Center
- [ ] App approved and published to tenant
- [ ] Bot tested with real Teams user (invite bot to a channel)

### 18.4 Security

- [ ] Client secret rotation scheduled (or certificate used)
- [ ] Least-privilege Graph permissions only
- [ ] No hard-coded secrets in source code
- [ ] `.gitignore` excludes dev secrets and PFX files
- [ ] OWASP Top 10 review complete
- [ ] All card payload values treated as untrusted server-side
- [ ] Audit records verified insert-only (DB permission review)
- [ ] Correlation IDs present on all log entries

---

## 19. Post-Deployment Verification

### 19.1 Smoke Tests

```bash
# 1. Provision a Team
curl -X POST https://<APP_SERVICE_NAME>.azurewebsites.net/api/teams/provision/team \
  -H "Content-Type: application/json" \
  -d '{
    "companyId": "ORG-TEST-001",
    "teamDisplayName": "Carbon - Test Org",
    "teamDescription": "Test",
    "owners": ["admin@yourtenant.com"],
    "members": []
  }'

# 2. Provision a Channel
curl -X POST https://<APP_SERVICE_NAME>.azurewebsites.net/api/teams/provision/channel \
  -H "Content-Type: application/json" \
  -d '{
    "companyId": "ORG-TEST-001",
    "teamId": "<RETURNED_TEAM_ID>",
    "channelName": "carbon-approvals",
    "description": "Approval workflow channel"
  }'

# 3. Map Org to Channel
curl -X POST https://<APP_SERVICE_NAME>.azurewebsites.net/api/teams/channels \
  -H "Content-Type: application/json" \
  -d '{
    "companyId": "ORG-TEST-001",
    "teamId": "<TEAM_ID>",
    "channelId": "<CHANNEL_ID>",
    "tenantId": "<TENANT_ID>"
  }'

# 4. Post Approval Card
curl -X POST https://<APP_SERVICE_NAME>.azurewebsites.net/api/teams/cards/approval \
  -H "Content-Type: application/json" \
  -d '{
    "companyId": "ORG-TEST-001",
    "sectionId": "SEC-5001",
    "documentId": "DOC-2001",
    "documentVersion": "v1",
    "sectionName": "Test Section",
    "lastEditor": "Test User",
    "lastEditedUtc": "2026-04-16T10:00:00Z",
    "workflowState": "PENDING_APPROVAL",
    "sectionVersionHash": "abc123",
    "lastModifiedUtc": "2026-04-16T10:00:00Z"
  }'
```

### 19.2 Application Insights Verification

```
Azure Portal > Application Insights > <AI_NAME>
  Live Metrics    — verify requests flowing
  Failures        — verify 0 errors on smoke tests
  Performance     — verify response times acceptable
  Logs            — query:
    requests | where cloud_RoleName == "Carbon.Teams.Api"
    | summarize count() by resultCode
```

### 19.3 Audit Verification

```bash
# Verify audit record integrity
curl https://<APP_SERVICE_NAME>.azurewebsites.net/api/audit/<AUDIT_ID>/verify
```

Expected response:
```json
{
  "id": "<AUDIT_ID>",
  "isValid": true,
  "verifiedAt": "2026-04-16T..."
}
```

---

## Appendix: Environment Variable Quick Reference

| Variable | Source | Used In |
|---|---|---|
| `AzureAd__TenantId` | App Settings | Graph, Bot auth |
| `AzureAd__ClientId` | App Settings | Graph, Bot auth |
| `AzureAd__ClientSecret` | KV Reference | Graph, Bot auth |
| `BotAppId` | App Settings | Bot identity |
| `ConnectionStrings__MongoDb` | KV Reference | MongoDB |
| `ApplicationInsights__ConnectionString` | KV Reference | Telemetry |
| `AuditHmacSecret` | KV Secret (via config) | Audit hash |
| `KeyVaultUri` | App Settings | KV SDK bootstrap |

---

*End of Implementation Guide — Version 1.0 — 2026-04-16*
