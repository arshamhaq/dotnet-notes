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

# Tracing a problem and fixing it

here I will use as an example a problem I faced and steps to fix it. The api in staging could not access the SQL DB, although the configurations were correct and the DB was online and alive.

**The main problem is that our deployment script registered the API’s identity in SQL using the wrong identifier.** The database account exists and has permissions, but its identity mapping does not match Microsoft’s documented requirements for this kind of application account.

We have prepared a correction and a repair. **We have not yet proved recovery in staging.** That happens when the running API successfully reads the roles after deployment.

Let’s build the explanation from the beginning.

**Your application runs across several separate resources.**

Azure hosts those resources. Azure DevOps stores your repository and runs the automation that builds and deploys your code.

| Component              | Its job in Hydrachem                                         |
| ---------------------- | ------------------------------------------------------------ |
| Microsoft Entra tenant | Holds identities for people and applications.                |
| Azure subscription     | Organizes Azure resources, billing, and access.              |
| Resource group         | Groups related resources—for example, your staging resources. |
| Web App Service        | Runs the website that displays the user interface.           |
| API App Service        | Runs the backend that handles requests and accesses data.    |
| Azure SQL database     | Stores ERP data, including users and roles.                  |
| Azure DevOps pipeline  | Runs the build, tests, infrastructure scripts, database scripts, and app deployment. |

When you open the role dropdown, the intended request path is:

````
```mermaid
flowchart LR
    B["Your browser"] --> W["Web App Service"]
    W --> A["API App Service"]
    A -->|"Authenticated SQL connection"| D["Staging SQL database"]
```
````

The website cannot display roles unless the API successfully retrieves them from SQL.

That is why a database login failure can appear in the browser as an empty dropdown or an internal error.

**The API needs its own identity to access SQL.**

Your own Azure login does not automatically give the running API access.

The API has a **system-assigned managed identity**. Think of it as an account Azure creates and manages specifically for that App Service. The application can obtain a short-lived identity token without us maintaining a database password. The identity is tied to that Azure resource’s lifecycle. [Microsoft’s managed identity explanation](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)

There are several separate identities involved:

| Identity                           | What it does                                                 |
| ---------------------------------- | ------------------------------------------------------------ |
| Your Azure/Cloud Shell identity    | Lets you inspect resources and run administrative SQL queries. |
| The pipeline’s deployment identity | Lets Azure DevOps provision resources, run database scripts, and deploy packages. |
| The API’s managed identity         | Lets the running API connect to SQL during normal use.       |
| An ERP application user            | Represents someone using Hydrachem, with a business role such as Admin or SalesRep. |

This distinction explains something that initially looked confusing:

> You could query SQL successfully, and previous deployments could succeed, while the API still failed.

Those operations used different identities.

Also, **ERP roles and SQL permissions are different things**:

- `Admin`, `SalesRep`, and `Customer` control behavior inside Hydrachem.
- `db_datareader`, `db_datawriter`, `CONNECT`, and `EXECUTE` control what a SQL database account can do.

An ERP Admin is not automatically a SQL administrator.

**The mistake happened when deployment created the API’s SQL account.**

The managed identity has two identifiers relevant here:

| Identifier                            | What it identifies                                           |
| ------------------------------------- | ------------------------------------------------------------ |
| Object ID, also called principal ID   | The service principal object in your Entra directory.        |
| Client ID, also called application ID | The application identity associated with that service principal. |

Both are legitimate identifiers. Different Azure operations require different ones.

For the particular SQL command our deployment uses—`CREATE USER ... WITH SID ... TYPE = E`—Microsoft specifies the **client ID** for applications and managed identities. [Microsoft’s exact SQL example](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-user-transact-sql?view=sql-server-ver17#k-create-a-contained-database-user-from-a-microsoft-entra-principal-without-validation)

A **SID** is SQL’s stored security identifier. Here, it is the identifier converted into a 16-byte representation. It is not a password.

Our old deployment effectively did this:

```
Get the API's object ID
        ↓
Convert it into SQL's binary SID format
        ↓
Create the API database user with that SID
        ↓
Grant reader, writer, and execute permissions
```

The conversion itself followed the expected byte format. **The wrong identifier went into it.**

The intended process is:

```
Get the API's client ID
        ↓
Convert it into SQL's binary SID format
        ↓
Create or validate the matching API database user
        ↓
Ensure its required permissions
```

This SQL creation method does not ask Entra to validate the supplied identity. Consequently, the script could create an account successfully even though the supplied SID represented the wrong identifier.

That explains why “the deployment succeeded” was insufficient evidence that the API could log in.

There was also a second defect. The original script essentially asked:

```
Does a database user with this name exist?

No  → Create it.
Yes → Assume it is correct.
```

It checked the name, but not the identity behind the name.

Therefore, the incorrect account survived later deployments.

**Our investigation separated the identity problem from missing data and missing permissions.**

Your checks established that:

- The three ERP roles exist.
- The API SQL account exists.
- It is an Entra account with a 16-byte SID.
- That SID matches the object ID.
- It has `CONNECT`, `EXECUTE`, reader, and writer access.
- The dependency categories inspected were empty.
- The staging `master` inspection found no matching conflicting account.
- Your `loginmanager` membership gave that inspection the necessary login visibility.
- You have full control of the staging application database.

The wrong SID is therefore our strongest explanation for the API’s login rejection.

We still have one distinction to preserve: those checks inspected the database and Azure configuration. They did not directly observe which credential the running API selected. The successful API request after deployment remains the decisive test.

And to revisit your earlier question: **`master` is a system database on the staging SQL server.** It is unrelated to the Git `main` branch and does not mean production.

**The fix needs to address both future deployments and the account that already exists.**

This is why I separated the work into two parts:

| Part                            | Purpose                                                      |
| ------------------------------- | ------------------------------------------------------------ |
| Permanent deployment correction | Create correct accounts in the future and reject incompatible existing accounts. |
| Incident repair                 | Correct the specific account already created incorrectly in staging. |

Changing the deployment recipe does not automatically modify an account already stored in SQL.

That was the missing piece in “couldn’t we just push the code?”

**We can deliver everything through a PR and the pipeline.** The pipeline must explicitly perform the database repair as part of that release.

Uploading the compiled API package by itself would leave the SQL account unchanged.

**The file extensions tell you which system executes each piece.**

| Extension       | Meaning                         | Who uses it?                                             |
| --------------- | ------------------------------- | -------------------------------------------------------- |
| `.cs`, `.razor` | Application code                | The running API/Web application after compilation.       |
| `.ps1`          | PowerShell script               | A developer’s terminal or a pipeline worker.             |
| `.sql`          | SQL instructions                | The SQL database engine, when a script sends them to it. |
| `.yml`          | Pipeline configuration          | Azure DevOps, which uses it to schedule tasks.           |
| `.bicep`        | Azure infrastructure definition | Azure’s deployment system.                               |
| `.md`           | Human-readable documentation    | Developers and reviewers.                                |

A pipeline worker, also called an **agent**, is the machine Azure DevOps uses to run the job.

For example, a `.ps1` script running on that worker can:

1. Ask Azure for identity information using the `az` command.
2. Construct a SQL command.
3. Send that command to SQL using `Invoke-Sqlcmd`.
4. Stop the job if a check fails.

Opening a `.ps1` or `.sql` file in your editor does not execute it.

Likewise, putting a SQL file in `deployment/sql/` does not make Azure run it automatically. Our PowerShell scripts explicitly load those files.

**Here is what each changed file does.**

| File                                                | What it does                                                 |
| --------------------------------------------------- | ------------------------------------------------------------ |
| azure-pipelines.yml                                 | The main pipeline instructions. I added the focused identity tests and enabled the repair parameter for the `DeployDev` stage. |
| deployment/templates/deploy-environment.yml         | The reusable deployment sequence. It now has an explicitly gated staging repair task before normal database deployment. It also passes the tenant and resource group to the database script. |
| deployment/scripts/deploy-database.ps1              | The regular database deployment script. It now resolves the client ID and validates the existing account before baseline or migration writes. It retains the existing database initialization and migration responsibilities. |
| deployment/scripts/SqlIdentity.ps1                  | Shared helper functions for obtaining and validating identity IDs, converting the client ID to a SID, and constructing validation/authorization SQL. |
| deployment/sql/ApiPrincipalGuard.sql                | The permanent account checks. It rejects an incorrect SID, account type, authentication type, conflicting account, or direct CONNECT/EXECUTE denial. |
| deployment/scripts/Repair-StagingApiSqlIdentity.ps1 | The incident-specific repair tool. It supports inspection, applying the repair, and explicitly reversing it. |
| deployment/sql/StagingApiRepairGuard.sql            | The stricter checks needed before replacing this existing account. It verifies the exact reviewed permissions, roles, identity, schema, and dependencies. |
| deployment/tests/SqlIdentity.Tests.ps1              | Tests the identity logic, SQL checks, and deployment behavior under success and failure conditions. |
| deployment/tests/Invoke-DeploymentFixture.ps1       | Runs deployment scripts in an isolated test process with fake Azure and SQL commands. This lets us test their behavior without contacting staging. |
| deployment/STAGING-SQL-IDENTITY-REPAIR.md           | The runbook: evidence, intended changes, execution behavior, rollback, limitations, and acceptance checks. |
| deployment/README.md                                | Points future developers to the repair and explains that a `dev` release currently includes this SQL operation. |
| docs/NAVIGATION-PERFORMANCE-MILESTONE.md            | Records how this investigation relates to the existing staging work and makes clear that hosted recovery is still pending. |

The two SQL guard files are **validation fragments**, not ordinary application-data migrations. The PowerShell scripts supply their variables and execute them.

The existing `database/migrations/` directory has a different purpose: it contains forward database changes that the normal deployment script tracks in its migration journal.

**The shared helper fixes the original mistake without requiring you to manually copy IDs each time.**

`SqlIdentity.ps1` asks Azure Resource Manager—the Azure resource-management API—for the API App Service’s managed identity information.

It then checks:

- Are we addressing the expected subscription and tenant?
- Is the resource the expected API?
- Does its object ID match the infrastructure output?
- Is the returned client ID a valid, nonempty identifier?

It uses the **client ID** to construct the SQL SID.

The object ID remains useful for confirming which Azure identity we retrieved. We have not globally replaced every object ID with a client ID.

For regular deployments, the result is:

| Existing database state                               | New behavior                                 |
| ----------------------------------------------------- | -------------------------------------------- |
| API account is absent                                 | Create it with the correct client-ID SID.    |
| API account already has the correct mapping           | Keep it and ensure the required permissions. |
| API account has an unexpected mapping                 | Stop and report the problem.                 |
| Another account represents the same expected identity | Stop for review.                             |

That last behavior matters. A routine deployment should not silently replace an unexpected security account.

**“Repair” means correcting the already-existing staging account.**

The repair script is deliberately more specific than the regular deployment script.

It knows the exact staging resource names and the exact old and proposed identifiers from this incident. It checks that they still match before proceeding.

Its three actions are:

| Action     | Behavior                                                     |
| ---------- | ------------------------------------------------------------ |
| `Inspect`  | Check the current state and report it. This is the default.  |
| `Apply`    | Replace the incorrect mapping with the client-ID mapping.    |
| `Rollback` | Restore the original mapping, after checking the expected state. |

I did not find a documented `ALTER USER ... WITH SID` operation for changing this particular Azure SQL contained Entra user in place.

The prepared repair therefore performs this sequence inside a SQL transaction:

```
Check the exact account and permissions
        ↓
Remove the old SQL account
        ↓
Create the same-named account with the client-ID SID
        ↓
Restore its reader/writer memberships and CONNECT/EXECUTE grants
        ↓
Check the resulting state
        ↓
Commit the transaction
```

Here, “remove the account” means removing a **SQL security principal**.

It does not mean deleting the ERP users table, customers, orders, or business roles. It also does not remove the API’s managed identity from Azure.

Replacing an account can nevertheless lose permissions or affect objects it owns. That is why we inspected dependencies and why the repair repeats those checks immediately before making changes.

The script does not promise to preserve every possible unknown configuration. Instead, it says:

> Proceed only if the account still matches the exact configuration we reviewed. Otherwise, stop.

The repair and normal authorization scripts also share an application lock so cooperating deployments cannot modify the account simultaneously.

**“Rollback” means undoing a change. “Staging” describes where a change runs.**

They describe different things.

A normal staging deployment moves staging forward to a new release. A rollback moves some part of the system back toward a previous state.

There are two rollback situations in this patch.

First, **automatic transaction rollback during the repair**:

Suppose account replacement starts, but restoring a permission fails. The SQL error handler rolls back the transaction. The intended result is that the database does not commit a half-completed replacement.

Second, **an explicitly requested reversal after a successful repair**:

The script’s `Rollback` action reconstructs the original object-ID mapping with the reviewed permissions.

That would restore the previous mapping—and likely the previous login failure. It is a controlled reversal mechanism if we need to investigate unexpected effects, not an action we expect to perform routinely.

Also, a SQL transaction covers only its SQL work.

If the SQL repair commits successfully and a later app-deployment task fails:

- The pipeline reports a failure.
- The completed SQL repair remains committed.
- Azure DevOps does not automatically reverse every earlier infrastructure and database operation.

Similarly, reverting a Git commit or deploying an older API package does not automatically restore the database’s previous SID.

There are two separate meanings of “commit” here:

- A **Git commit** records a version of your files.
- A **SQL transaction commit** makes the database changes in that transaction permanent.

**The staging-only part is intentionally temporary. The permanent fix remains useful.**

The reusable identity lookup and validation apply to normal deployments. When production is eventually provisioned, that code should obtain the production API’s client ID and create its SQL account correctly.

The incident repair is restricted to staging because that is where we inspected an existing broken account.

There are two protections:

1. The deployment template requires the environment to be `dev` and the repair parameter to be enabled.
2. The repair script itself checks the exact staging resources and identifiers.

Production does not receive the incident repair merely because it uses the same deployment template.

The current staging pipeline contains:

```
repairStagingSqlIdentity: true
```

While that remains enabled, each staging deployment runs the repair checks. If the account is already in the exact corrected state, the script skips replacement.

**It does not automatically turn itself off.**

After we confirm recovery, we should remove that flag in a follow-up change. The normal deployment correction stays.

The incident script can remain temporarily for rollback/history and later be archived or removed from active deployment files. Git retains its history. That is normal for a repair written for a specific incident.

**After your PR merges into `dev`, the proposed pipeline runs this sequence.**

1. Build the application and run the required tests.
2. Package the application and deployment files.
3. Run the existing infrastructure deployment.
4. Run the explicitly enabled staging identity repair.
5. Run normal database validation and any applicable migrations.
6. Deploy the API and Web application.
7. Run the existing health checks.
8. Perform the real application acceptance checks.

The pipeline’s deployment identity performs those operations. Your browser session or Cloud Shell identity is not transferred into the pipeline.

If the repair detects unexpected state, it stops the job before the later API/Web deployment steps.

The `/alive` checks are useful, but they mainly establish that the processes are running. **A green health check does not establish that the API can retrieve SQL data.**

Our acceptance check must include an authenticated request that actually loads the roles.

**The tests provide evidence about the implementation, with a clear limit.**

The 49 focused tests passed. They check situations such as:

- Azure identity lookup fails.
- The returned subscription, tenant, or identity is wrong.
- The database user has the wrong SID or type.
- Permissions or dependencies have changed.
- Inspection accidentally attempts writes.
- Deployment should stop before database changes.

Some tests execute the real SQL guards against temporary catalog-like tables in LocalDB. Others use fake Azure/SQL commands to test script behavior.

Generated SQL also passed syntax checking without execution.

Those tests do **not** prove that Azure SQL accepts the real API token, or that the full account replacement has succeeded in Azure. We have not run that live operation yet.

The application’s existing CI tests remain required too. The number 49 describes the new focused checks, not the entire ERP test suite.

The earlier session, authentication, and navigation changes on this branch are separate work. If those commits have not yet reached `dev`, they will also be part of the PR and release, so we need to review the complete branch difference.