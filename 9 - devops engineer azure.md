# Azure DevOps Engineering

The goal isn’t memorizing Azure commands. It’s being able to answer:

> What runs where? Who can access it? How does new code get there? How do I know it works? What do I do when it breaks?

AI can help write commands and configurations. Your responsibility is understanding their purpose, scope, and risk.

## 1. What a DevOps engineer actually does

You already write application code. DevOps adds responsibility for **delivering and operating that application reliably**.

That means:

- Turning code into a tested, deployable package.
- Preparing the infrastructure where it runs.
- Managing configuration, permissions, and secrets.
- Monitoring errors, performance, and costs.
- Recovering safely when something goes wrong.

“DevOps” is the practice. **Azure DevOps** is a Microsoft product that provides tools for that practice. You can do DevOps without using Azure DevOps. [Microsoft’s explanation](https://learn.microsoft.com/en-us/devops/what-is-devops)

For Hydrachem, distinguish these three things:

| Thing             | What it is                                                   |
| ----------------- | ------------------------------------------------------------ |
| Azure DevOps      | Your repositories, work items, build pipelines, and deployment workflows |
| Azure             | The cloud platform hosting the running applications and database |
| Azure Cloud Shell | A terminal where you run commands using your Azure identity  |

<mark>Cloud Shell is **not a terminal inside your API application**. It runs in a separate environment. Commands such as `az webapp ...` communicate with Azure to inspect or manage your app.</mark>

## 2. How Azure organizes everything

Start with four terms:

**<mark>Tenant:</mark>** the organization’s identity directory, managed through Microsoft Entra. It contains identities such as your guest account and application identities.

**<mark>Subscription:</mark>** a billing and resource-management boundary. Yours is named `Hydrachem`.

**<mark>Resource group:</mark>** a container for related resources. Your staging group is:

```
rg-hydrachem-dev-canadacentral
```

**Resource:** an individual cloud service, such as your API application or SQL database.

For your project:

```
Company’s Entra tenant
└── Hydrachem subscription
    └── Staging resource group
        ├── Web application
        ├── API application
        ├── App Service plan
        ├── SQL server and database
        ├── Monitoring resources
        ├── Key Vault
        └── Email resources
```

A resource group is **not a server or a private network**. Putting resources in the same group doesn’t automatically connect them or give them permission to access each other. [Azure’s resource-management model](https://learn.microsoft.com/en-gb/azure/azure-resource-manager/management/overview)

Also, `dev`, `prod`, and `canadacentral` in a resource’s **name** are naming conventions. Verify the actual configuration rather than relying solely on the name.

## 3. What your resources do

Here is the resource list you shared, translated into plain language. Long generated suffixes are omitted.

| Your resource                         | Azure type                                                | Purpose                                                      |
| ------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| `app-hydrachem-web-dev`               | `Microsoft.Web/sites`                                     | Runs your Blazor Web application                             |
| `app-hydrachem-api-dev`               | `Microsoft.Web/sites`                                     | Runs your ASP.NET API                                        |
| `asp-hydrachem-dev`                   | `Microsoft.Web/serverFarms`                               | Provides the computing capacity used by the applications     |
| `sql-hydrachem-dev`                   | `Microsoft.Sql/servers`                                   | Logical SQL server: connection endpoint and server-level settings |
| `sqldb-hydrachem-dev`                 | `Microsoft.Sql/servers/databases`                         | Stores your ERP tables and data                              |
| `log-hydrachem-dev`                   | `Microsoft.OperationalInsights/workspaces`                | Stores collected monitoring data for querying                |
| `appi-hydrachem-dev`                  | `Microsoft.Insights/components`                           | Application Insights: investigates requests, failures, and performance |
| `ag-hydrachem-dev`                    | `Microsoft.Insights/actiongroups`                         | Defines who gets notified or what happens when an alert fires |
| `alert-hydrachem-api-http5xx-dev`     | `Microsoft.Insights/metricalerts`                         | Detects a configured level of API server errors              |
| `kv-hc-dev`                           | `Microsoft.KeyVault/vaults`                               | Secure storage for secrets, keys, and certificates           |
| `acs-hydrachem-dev`                   | `Microsoft.Communication/communicationServices`           | Communication service used by the email integration          |
| `email-hydrachem-dev` and its domains | `Microsoft.Communication/emailServices` and child domains | Email sending setup and verified sending domains             |
| The Web certificate resource          | `Microsoft.Web/certificates`                              | Certificate material associated with HTTPS hosting           |

Two especially important distinctions:

### Application versus App Service plan

The **application** contains your deployed code and settings.

The **plan** supplies CPU and memory—the computing resources that run it.

Your infrastructure code currently places Web and API in the **same Linux B1 plan**, so they share that plan’s computing capacity. They aren’t two completely independent machines. [App Service plans](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans)

### SQL server versus SQL database

Your Azure SQL “server” is a **logical management boundary**, not a Windows machine you should log into and maintain.

The database beneath it contains tables such as:

```
UserProfiles
webpages_Roles
webpages_UsersInRoles
```

Microsoft operates the underlying database platform. You still own your schema, queries, access rules, and recovery planning.

## 4. Follow one request through the system

A simplified Hydrachem request looks like this:

```
Browser
   │ HTTPS
   ▼
Web application
   │ HTTP API request
   ▼
API application
   │ SQL connection
   ▼
SQL database
```

Each connection can fail independently.

For example:

- The browser can reach Web, but Web cannot reach API.
- API can receive requests, but SQL rejects its login.
- SQL accepts the login, but a query references a missing table.
- Everything succeeds, but a query takes ten seconds.

This is why “the website opens” doesn’t prove the database works.

There are four questions to ask about a connection:

1. **Address:** Am I contacting the correct hostname and service?
2. **Network:** Can my request reach it?
3. **Identity:** Who am I claiming to be?
4. **Permission:** Is that identity allowed to perform this operation?

Your investigation is an excellent example:

| Observation                                                | What it established                              |
| ---------------------------------------------------------- | ------------------------------------------------ |
| Resource listing succeeded                                 | Your Azure account can see the staging resources |
| Connection setting showed the expected SQL server/database | That setting targets the expected database       |
| Browser query editor blocked your IP                       | That access path hit a network restriction       |
| Cloud Shell SQL query returned “login failed”              | SQL rejected your account’s login on that path   |
| API logs showed the same error category                    | SQL also rejected the API’s separate login       |

That discipline—knowing what evidence proves—is a core engineering skill.

## 5. Identities and permissions: the part that confused us

<mark>Azure calls managing resources the **control plane**, and accessing their contents the **data plane**.</mark>

In your example:

```
Inspect the SQL resource’s configuration → Azure management operation
SELECT rows from webpages_Roles         → SQL database operation
```

That is why the first worked while the second failed. [Control plane versus data plane](https://learn.microsoft.com/azure/azure-resource-manager/management/control-plane-and-data-plane)

### <mark>What is a managed identity?</mark>

<mark>It is an identity for an application.</mark>

Your API uses its system-assigned managed identity to request access to Azure SQL. Azure manages the credentials rather than you storing a permanent database password.

But:

> Having an identity does not automatically give that identity database access.

The database must recognize and authorize it.

A system-assigned identity belongs to the application’s lifecycle. Deleting and recreating it can produce a different identity ID. Therefore, casually toggling identity settings can break permissions elsewhere. [Managed identities](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-identity)

Also, these are different identities:

- You, investigating through Cloud Shell.
- The pipeline, deploying the application.
- The API, connecting to SQL while serving users.

**The pipeline successfully connecting to SQL does not prove the API can connect.**

## 6. The Azure portal pages worth learning first

Names and menu locations can vary slightly, but these are the useful areas.

| Area                                              | Question it helps answer                                     |
| ------------------------------------------------- | ------------------------------------------------------------ |
| **Overview**                                      | Which resource is this? Is it running? What region and hostname does it use? |
| **Activity log**                                  | What management changes happened, when, and by whom?         |
| **Access control — IAM**                          | Who can manage this Azure resource?                          |
| **Identity** on an App Service                    | User assigned managed identities enable Azure resources to authenticate to cloud services (e.g. Azure Key Vault) without storing credentials in code. |
| **Environment variables / Configuration**         | Which runtime settings does the app receive?                 |
| **Log stream**                                    | What is the application reporting right now?                 |
| **Metrics**                                       | Are requests, errors, CPU, or memory increasing?             |
| **Application Insights → Failures / Performance** | Which operations fail or take too long, if telemetry is configured? |
| **Networking**                                    | Which network access restrictions apply?                     |
| **Scale up / Scale out**                          | How much computing capacity is allocated?                    |
| **Resource group → Deployments**                  | Did infrastructure deployment succeed?                       |
| **SQL Query editor**                              | Can an authorized SQL account execute a query?               |
| **SQL backup/restore settings**                   | What recovery options exist?                                 |
| **Cost Management**, where permitted              | What is costing money and how is spending changing?          |

Two traps:

**Activity log is not application logging.**

“Someone changed an App Service setting” belongs in Activity log.

“`RoleRepository.GetAllRoles()` threw a SQL exception” belongs in application logs. [Activity log documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/activity-log)

**An Application Insights resource existing doesn’t prove your app sends useful telemetry to it.** Collection and instrumentation must actually be configured.

### Configuration changes can affect availability

In normal ASP.NET configuration, an environment variable like:

```
ConnectionStrings__DefaultConnection
```

maps to:

```
ConnectionStrings:DefaultConnection
```

Environment settings can override values from `appsettings.json`.

Changing App Service application settings triggers an app restart. So “just changing a setting” can affect active users. [App Service configuration](https://learn.microsoft.com/azure/app-service/configure-common?tabs=portal)

## 7. Cloud Shell commands you should understand

You don’t need to memorize the long resource names. Use variables:

```
$hcRg  = 'rg-hydrachem-dev-canadacentral'
$hcApi = 'app-hydrachem-api-dev-ftmeixbzsjkx2'
$hcSql = 'sql-hydrachem-dev-ftmeixbzsjkx2'
```

These assignments only store text in your current shell.

### Confirm your context

```
az account show --query "{Subscription:name, Id:id, Tenant:tenantId}" -o table
```

Before any consequential operation, verify the subscription and target environment.

### List resources

```
az resource list -g $hcRg --query "[].{Name:name, Type:type}" -o table
```

### Inspect the API

```
az webapp show -g $hcRg -n $hcApi --query "{State:state, Host:defaultHostName}" -o table
```

“Running” means the app is started—not that every business operation succeeds.

### Inspect its managed identity

```
az webapp identity show -g $hcRg -n $hcApi
```

### Watch application logs

```
az webapp log tail -g $hcRg -n $hcApi
```

This continues until you press **Ctrl+C**. Streaming availability depends on the app’s logging setup and hosting environment.

### Inspect firewall rules

```
az sql server firewall-rule list -g $hcRg -s $hcSql -o table
```

### Inspect database status

```
az sql db show -g $hcRg -s $hcSql -n sqldb-hydrachem-dev --query "{Name:name, Status:status, Tier:sku.tier}" -o table
```

An `Online` database can still reject your login.

### Inspect recent management changes

```
az monitor activity-log list -g $hcRg --offset 24h --query "[].{Time:eventTimestamp, Operation:operationName.localizedValue, Status:status.localizedValue}" -o table
```

This is useful when something worked yesterday but not today. [Activity-log commands](https://learn.microsoft.com/en-us/cli/azure/monitor/activity-log)

### Learn a command without executing a change

```
az webapp --help
az webapp show --help
```

Useful syntax:

- `az`: Azure CLI.
- `-g`: resource group.
- `-n`: name.
- `-o table`: display results as a table.
- `--query`: select/filter returned JSON using a language called **JMESPath**. It is not SQL.
- $variable: a PowerShell variable.
- { ... }: a PowerShell collection of named values.
- `@sqlProbeOptions`: pass that collection as command parameters.
- `|`: pass output to the next command.

`Invoke-Sqlcmd` is different from `az sql db show`:

```
az sql db show → asks Azure about the database resource
Invoke-Sqlcmd  → connects to SQL and executes SQL statements
```

That difference explains the different permission requirements.

Be cautious with commands containing `delete`, `set`, `update`, `create`, `restart`, `restore`, or `assign`. Also, **read-only commands can expose secrets**, so don’t blindly share their output.

## 8. How your deployment pipeline works

A **pipeline** is an automated sequence of jobs.

A **pipeline agent** is the machine that runs those jobs.

An **artifact** is a saved output, such as your compiled application package.

Your repository’s pipeline roughly does this:

```
Get source code
→ Restore dependencies
→ Build
→ Run tests
→ Validate infrastructure code
→ Publish application packages
→ Deploy infrastructure
→ Apply database scripts
→ Deploy API and Web
→ Check /alive
```

**CI—continuous integration:** regularly combine changes and automatically build/test them.

**Continuous delivery:** keep tested changes ready to deploy, potentially with approval before production.

**Continuous deployment:** automatically deploy qualifying changes without that manual release decision.

Your YAML currently maps the `dev` branch to development deployment and `main` to production deployment. Any configured environment approvals are an additional control; don’t assume the YAML alone tells you all approval rules.

### Why a green pipeline wasn’t enough here

Your deployment checks `/alive`.

Your current health-check code tests whether the application responds. It does **not** test a real SQL read.

So these can both be true:

```
Deployment succeeded.
The Users page is broken.
```

A stronger verification strategy distinguishes:

- <mark>**Liveness:** is the process responsive?</mark>
- <mark>**Readiness:** can it serve its intended work?</mark>
- <mark>**Smoke test:** does a small, important user flow actually work?</mark>

Don’t indiscriminately put every dependency into liveness checks: restarting applications during a database outage can make recovery worse.

## 9. Infrastructure as code: why those Bicep files matter

Instead of relying on someone to remember dozens of portal clicks, your repository describes resources in **Bicep**, Azure’s infrastructure-definition language. For example, your environment template defines:

- Web and API applications.
- Their hosting plan.
- SQL server and database.
- Application settings.
- Managed identities.
- Monitoring and alerts.

This gives you reviewable, versioned infrastructure. 

here is an example visualization of the main.bicep : 

![](C:\WORK FREELANCE\dotnet-notes\assets\main.png)

<mark>A key problem is **configuration drift**:</mark>

> Someone changes Azure manually, but the infrastructure code still describes something different.

A later deployment may overwrite the manual change, or environments may become inconsistent.

Professional habit:

1. Inspect the current state.
2. Make the smallest authorized change.
3. Verify the result.
4. Record lasting configuration changes in the appropriate code or documented process.

Also distinguish:

- **Infrastructure deployment:** create/configure the SQL service.
- **Database migration:** change tables, indexes, or reference data inside SQL.
- **Application deployment:** replace the running code.

They are connected but not interchangeable.

## 10. Monitoring, performance, and recovery

### Three kinds of evidence

**Logs:** individual events.

> “SQL login failed at 19:34.”

**Metrics:** numbers over time.

> “5% of requests failed in the last ten minutes.”

**Traces:** follow a request through multiple components.

> “Web took 12 seconds; most of that time was spent waiting on API calls.”

Application Insights and Azure Monitor support these investigations when configured appropriately. [Azure Monitor overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)

<mark>Learn **p95 latency**: a response time at or below which 95% of measured requests finish. It helps reveal slow experiences that an average can hide.</mark>

### More servers are not automatically the answer

<mark>**Scale up:** give a machine more capacity.</mark>

<mark>**Scale out:** run more instances.</mark>

More capacity will not repair:

- Invalid SQL permissions.
- Broken authentication.
- Unnecessary retries.
- Inefficient queries.
- Requests that load far more data than needed.

Multiple Web instances also introduce state questions. For example, if login sessions live only in one server’s memory, another server may not recognize them. <mark>Scaling requires checking session storage, encryption keys, connections, and database capacity—not just moving a slider.</mark>

### Backups and rollback are different

**Rollback:** return to an earlier application/configuration version.

**Restore:** recover data from a backup.

Deploying yesterday’s API does not undo today’s database migration. Restoring yesterday’s database may lose today’s valid orders.

Learn two recovery terms:

- <mark>**RPO:** how much recent data loss the business can tolerate.</mark>
- <mark>**RTO:** how long the service can remain unavailable.</mark>

Azure SQL provides automated backup capabilities, but retention and recovery procedures still need verification. A backup policy is not a substitute for testing a restore. [Azure SQL backups](https://learn.microsoft.com/en-us/azure/azure-sql/database/automated-backups-overview)

## 11. How to work professionally with AI

Before executing a generated command, ask:

1. Which subscription, environment, and resource does it target?
2. Does it read information or change something?
3. Could it expose secrets?
4. Could it restart the app, delete data, widen access, or increase cost?
5. How will I verify success?
6. How would I recover if it goes wrong?

A useful prompt is:

> Investigate using read-only checks first. Explain what each command proves. Don’t print secrets. Before proposing a change, identify its scope, permission requirements, user impact, verification steps, and recovery plan.

And remember: a more confident AI answer is not stronger evidence. Our actual SQL logs were stronger evidence than all the earlier guesses.

**The important transition is from “I can run this command” to “I understand why this is the right command, what it will affect, and how I’ll verify the outcome.”** That’s the foundation of doing DevOps engineering well.