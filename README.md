# Joe Security Report Phishing – Outlook Add-in

**Version:** 1.0.0

The Joe Security Report Phishing add-in lets Outlook users report a suspicious email to **Joe Sandbox** with one click. The full original email (headers, body and all attachments) is submitted to Joe Sandbox as an `.eml` file for analysis.

Supported clients: Outlook on Windows and Mac, Outlook on the web and new Outlook.

---

## Contents

- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Deployment guide](#deployment-guide)
  - [Phase 1 – Create the App Registration](#phase-1--create-the-app-registration)
  - [Phase 2 – Deploy the Web App](#phase-2--deploy-the-web-app)
  - [Phase 3 – Finish the App Registration](#phase-3--finish-the-app-registration)
  - [Phase 4 – Get the manifest](#phase-4--get-the-manifest)
  - [Phase 5 – Deploy the add-in in the Microsoft 365 Admin Center](#phase-5--deploy-the-add-in-in-the-microsoft-365-admin-center)
  - [Phase 6 – Verify](#phase-6--verify)
- [Configuration reference](#configuration-reference)
- [User guide](#user-guide)
- [Troubleshooting](#troubleshooting)
- [Development](#development)
- [Version history](#version-history)

---

## How it works

```
Outlook task pane ──(SSO token, message ID)──▶ Web App (FastAPI middle tier)
                                                 │ 1. validate the token
                                                 │ 2. On-Behalf-Of → Microsoft Graph token
                                                 │ 3. GET /me/messages/{id}/$value  (MIME, in memory)
                                                 │ 4. submit .eml to Joe Sandbox (jbxapi)
                                                 │ 5. optional: move email to a folder
Outlook task pane ◀──────── "Submitted" or a clear error message
```

1. The user opens an email and clicks **Report Phishing** in the ribbon or message action bar, then confirms in the task pane.
2. The task pane gets a Microsoft 365 single sign-on (SSO) token. If SSO is unavailable, it falls back to a sign-in dialog.
3. The middle tier validates the token and exchanges it for a Microsoft Graph token (On-Behalf-Of flow).
4. The middle tier downloads the email's MIME content and submits it to Joe Sandbox. The submission is tagged `Outlook Add-in` plus the reporting user's address in tag form (Joe Sandbox tags allow only letters, numbers, `_`, `-` and spaces, so `first.last@contoso.com` becomes `first_last_contoso_com`). The comment carries the exact address: `Reported via Outlook Add-in by <user>`.
5. If enabled, the email is moved to a dedicated folder (for example `Phishing Report`) after a successful submission.

**Data handling.** The email is never forwarded to a mailbox and never written to disk. It is held in memory only for the duration of the request. No user credentials or user tokens are stored. The reporting user's address is sent to Joe Sandbox in the analysis tag and comment, so that analysts can see who reported the email.

---

## Prerequisites

- An **Azure subscription**, to host the Web App (Azure App Service, Linux, Python 3.12).
- A **Microsoft 365 administrator** account, to create the App Registration, grant admin consent and deploy the add-in.
- A **Joe Sandbox** account (Joe Sandbox Cloud or on-premise) and its **API key**. To use Joe Reverser, the account must include it.
- Permission to create resources in the target resource group. The template briefly runs a *deployment script* (a 60-second wait that lets the Web App apply its settings before the package is deployed). Azure creates a temporary storage account and container instance for it and deletes them afterwards, so the resource providers `Microsoft.Storage` and `Microsoft.ContainerInstance` must be registered in the subscription (**Subscription → Settings → Resource providers**). They usually are.

---

## Deployment guide

### Phase 1 – Create the App Registration

1. Open the [Azure portal](https://portal.azure.com) and go to **Microsoft Entra ID → App registrations → New registration**.
2. Fill in:
   - **Name:** `Joe-Security-Outlook-Addin`
   - **Supported account types:** *Accounts in this organizational directory only*
   - **Redirect URI:** leave empty for now.
3. Click **Register**. On **Overview**, copy the **Application (client) ID** and the **Directory (tenant) ID**.
4. Go to **Certificates & secrets → New client secret**. Add a description and expiry, click **Add**, and copy the secret **Value** immediately (it is shown only once).
5. Go to **API permissions → Add a permission → Microsoft Graph → Delegated permissions** and add:
   - `openid`
   - `profile`
   - `User.Read`
   - `Mail.ReadWrite`. If you will never enable *Move reported email to folder*, `Mail.Read` is enough. In that case also change the `Mail.ReadWrite` scope in the manifest to `Mail.Read`.
6. Click **Grant admin consent for \<tenant\>** and confirm.

### Phase 2 – Deploy the Web App

1. Click **Deploy to Azure**:

   [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2F%7BJOE_GITHUB_ORG%7D%2F%7BJOE_REPOSITORY%7D%2Fmain%2Fdeploy%2Fazuredeploy.json)

   > **Placeholder:** replace `{JOE_GITHUB_ORG}/{JOE_REPOSITORY}` in this link and in the `PackageUrl` default of `deploy/azuredeploy.json` with the repository that hosts the add-in.

2. Fill in the parameters:

   | Parameter | Description |
   | --- | --- |
   | Subscription / Resource group / Region | Where the Web App is created. |
   | Web App Name | Name of the Web App. It forms the add-in domain. |
   | Sku | App Service tier. `B1` is fine for testing; `S1` or higher is recommended for production. |
   | Azure Tenant ID | Directory (tenant) ID from Phase 1. |
   | Azure Client ID | Application (client) ID from Phase 1. |
   | Azure Client Secret | Client secret value from Phase 1. |
   | Joe Sandbox Api Url | Joe Sandbox API endpoint. Default: Joe Sandbox Cloud (`https://jbxcloud.joesecurity.org/api`). On-premise: `http(s)://<server>/joesandbox/index.php/api`. |
   | Joe Sandbox Api Key | Joe Sandbox API key used for submissions. |
   | Accept Joe Sandbox Terms And Conditions | Must be `true` for Joe Sandbox Cloud ([Terms and Conditions](https://jbxcloud.joesecurity.org/resources/termsandconditions.pdf)). |
   | Analyze On Joe Reverser | `true` to also analyze reported emails with Joe Reverser. The account must include Joe Reverser. |
   | Move Reported Email To Folder | `true` to move the email to a folder after a successful submission. |
   | Reported Email Folder Name | Top-level mail folder for reported emails (default `Phishing Report`). It is created if it does not exist. |
   | Max Email Size MB | Largest email allowed for submission (default 25 MB). |
   | Deployment Timestamp | Leave as is. |
   | Package Url | Location of `source.zip`. Keep the default. |

3. Click **Review + create**, then **Create**. The deployment takes about 5 minutes.
4. When the deployment finishes, open **Outputs** and copy **WebAppURL** (for example `https://joe-report.azurewebsites.net`). Its host name is called **`<domain>`** below.
5. Open `https://<domain>/health`. It should return `"status": "ok"` and `"configured": true`.

### Phase 3 – Finish the App Registration

Open the App Registration from Phase 1.

1. **Authentication → Add a platform → Single-page application**, with redirect URI:
   ```
   https://<domain>/fallbackauthdialog.html
   ```
2. **Expose an API → Application ID URI → Add**. Set it to:
   ```
   api://<domain>/<client-id>
   ```
   and click **Save**.
3. **Add a scope**:
   - **Scope name:** `access_as_user`
   - **Who can consent:** *Admins only*
   - **Admin consent display name:** `Access Joe Security Outlook Add-in`
   - **Admin consent description:** `Allows the Outlook add-in to call its middle tier on behalf of the signed-in user.`
   - **State:** *Enabled*
4. Under **Authorized client applications**, click **Add a client application**, enter `ea5a67f6-b6f3-4338-b240-c655ddc3cc8e` (all Microsoft Office applications), tick the `access_as_user` scope and click **Add application**.

> If you use a custom domain for the Web App, add an application setting `APP_DOMAIN=<custom domain>` (App Service → **Settings → Environment variables**) and use that domain throughout this phase.

### Phase 4 – Get the manifest

The Web App serves a ready-to-use manifest, with the domain and client ID already filled in:

```
https://<domain>/manifest.xml
```

Download it and save it as `manifest.xml`.

**Manual alternative:** edit [app/manifest.xml](app/manifest.xml) and replace `{YOUR_DOMAIN}`, `{CLIENT_ID}` and `{APPLICATION_ID_URI}`.

**Optional:** change the display texts (`DisplayName`, `Description`, `GroupLabel`, `ReportButton.Label`, `ReportButton.Tooltip`) or `SupportUrl` to match your organization.

### Phase 5 – Deploy the add-in in the Microsoft 365 Admin Center

1. Open the [Microsoft 365 Admin Center](https://admin.microsoft.com) as a Global Administrator.
2. Go to **Settings → Integrated apps → Upload custom apps**.
3. Choose **Office Add-in**, select **Upload manifest file (.xml) from device**, and upload `manifest.xml`.
4. Assign users: **Just me** (recommended for a first test), **Specific users/groups**, or **Entire organization**.
5. Accept the requested permissions and finish the wizard.

It can take up to 24 hours (in rare cases 72) before the add-in appears for users. Signing out of Outlook and back in usually speeds this up.

### Phase 6 – Verify

1. Open an email in Outlook. **Report Phishing** appears in the ribbon (classic Outlook) or under the **Apps** button of the message action bar (Outlook on the web and new Outlook).
2. Click it, then click **Report**. The task pane shows **Submitted to Joe Sandbox.**
3. In Joe Sandbox, a new analysis appears with the tags `Outlook Add-in` and the reporter's address in tag form (for example `first_last_contoso_com`), and the comment `Reported via Outlook Add-in by <user>`.
4. If *Move reported email to folder* is enabled, the email moves to the configured folder.

**Post-deployment checklist**

- [ ] `/health` returns `"configured": true`
- [ ] SSO works without a sign-in prompt for an already signed-in user
- [ ] An email with attachments is submitted and analyzed
- [ ] An email above *Max Email Size MB* is rejected with a clear message
- [ ] Move-to-folder behaves as configured

---

## Configuration reference

All settings are App Service application settings (**Web App → Settings → Environment variables**). The ARM template creates them. After a change, click **Apply**; the app restarts automatically.

| Setting | Required | Default | Description |
| --- | --- | --- | --- |
| `AZURE_TENANT_ID` | yes | | Directory (tenant) ID |
| `AZURE_CLIENT_ID` | yes | | Application (client) ID |
| `AZURE_CLIENT_SECRET` | yes | | Client secret. Consider a [Key Vault reference](https://learn.microsoft.com/azure/app-service/app-service-key-vault-references). |
| `APP_DOMAIN` | no | `WEBSITE_HOSTNAME` | Public host name. Set it for a custom domain. |
| `AZURE_APP_ID_URI` | no | `api://<APP_DOMAIN>/<client id>` | Application ID URI, if you chose a different one |
| `JOE_API_URL` | yes | `https://jbxcloud.joesecurity.org/api` | Joe Sandbox API endpoint |
| `JOE_API_KEY` | yes | | Joe Sandbox API key. Consider a Key Vault reference. |
| `JOE_ACCEPT_TAC` | Cloud | `false` | Accept the Joe Sandbox Cloud Terms and Conditions |
| `JOE_ANALYZE_ON_REVERSER` | no | `false` | Also analyze with Joe Reverser (`analyze-on-reverser`) |
| `MOVE_REPORTED_EMAIL_TO_FOLDER` | no | `false` | Move the email after a successful submission |
| `REPORTED_EMAIL_FOLDER_NAME` | no | `Phishing Report` | Target top-level folder (created if missing) |
| `MAX_EMAIL_SIZE_MB` | no | `25` | Maximum email size for submission |

Every other analysis setting (analysis systems, internet access and so on) uses the defaults of your Joe Sandbox account.

---

## User guide

1. Open the suspicious email.
2. Click **Report Phishing**:
   - **Classic Outlook (Windows/Mac):** on the ribbon, in the *Joe Security* group.
   - **Outlook on the web / new Outlook:** in the message action bar, under the **Apps** icon. You can pin it for one-click access.
3. Check the subject shown in the task pane and click **Report**.
4. Keep the task pane open until **Submitted to Joe Sandbox.** appears. If your organization enabled it, the email is then moved to the *Phishing Report* folder.

The first time, a sign-in window may appear. Sign in with your work account.

**Messages you may see**

| Message | Meaning / what to do |
| --- | --- |
| Submitted to Joe Sandbox. | The email was submitted for analysis. |
| This email is larger than the maximum size allowed… | The email exceeds the configured size limit; contact your security team. |
| This email could not be found… | The email was moved or deleted while reporting; open it again and retry. |
| Your sign-in has expired / could not be verified… | Click **Try again**; sign in when prompted. |
| Sign-in was cancelled… / The sign-in window could not be opened… | Allow pop-ups for Outlook and click **Try again**. |
| Microsoft 365 is busy / temporarily unavailable… | Temporary Microsoft issue; try again later. |
| Joe Sandbox is currently unavailable / did not respond in time… | Temporary Joe Sandbox issue; try again later. |
| The Joe Sandbox submission quota has been reached… | The account quota is used up; try later or contact your administrator. |
| Joe Sandbox rejected the configured API key… / Terms and Conditions… / not permitted… | Configuration problem; contact your administrator. |

Temporary failures (HTTP 408, 429 and 5xx) are retried automatically with exponential back-off before an error is shown.

---

## Troubleshooting

**Logs.** Web App → **Monitoring → Log stream**. Each report is logged with a short hash of the message ID, the user's object ID and the Joe Sandbox submission ID. Email content is never logged.

| Symptom | Likely cause |
| --- | --- |
| `/health` shows `"configured": false` | A required setting is empty; the log lists the missing setting names. |
| "app registration is misconfigured" | The client secret is wrong or expired. Create a new one and update `AZURE_CLIENT_SECRET`. |
| "Permission to read this email has not been granted" | Admin consent is missing (Phase 1, step 6). |
| Sign-in dialog fails with a redirect URI error | The SPA redirect URI from Phase 3, step 1 is missing or has a typo. |
| "Your sign-in could not be verified" every time | The Application ID URI differs from `api://<domain>/<client-id>`. Set `AZURE_APP_ID_URI` or `APP_DOMAIN`. |
| "Terms and Conditions have not been accepted" | Set `JOE_ACCEPT_TAC=true` (Joe Sandbox Cloud). |
| "not permitted to make this submission" with Joe Reverser enabled | The Joe Sandbox account does not include Joe Reverser; set `JOE_ANALYZE_ON_REVERSER=false`. |
| Add-in does not appear | Wait for propagation; sign out of Outlook and back in. |
| Clicking the add-in does nothing (Outlook on the web) | A browser privacy extension or built-in blocker (for example Brave Shields or an ad blocker) blocks the add-in. Allow `outlook.office.com`, `outlook.cloud.microsoft` and `<domain>` and reload Outlook. |
| Template deployment fails at `waitForWebApp` | The `Microsoft.ContainerInstance` or `Microsoft.Storage` resource provider is not registered, or a policy blocks the temporary resources. See [Prerequisites](#prerequisites). |
| Deployment fails in the *Build* step (for example `libpython3.12.so.1.0: cannot open shared object file`) | The Web App is building on the server, which the package does not need. Set `SCM_DO_BUILD_DURING_DEPLOYMENT=false` and `ENABLE_ORYX_BUILD=false`, **Stop** and **Start** the Web App (this also clears a damaged build cache), and deploy again. |
| App does not start: `ModuleNotFoundError: No module named 'app'`, health check unhealthy | The startup command must be `sh startup.sh` (**Configuration → General settings → Startup Command**). The first log line of each start shows where `startup.sh` found the app (`ROOT=…`). If it says `<not found>`, the deployed zip is not `deploy/source.zip` as built (for example, it was re-zipped with an extra top-level folder). |

---

## Development

The project uses Python only: no Node.js and no build step for the task pane.

```
app/
  main.py            FastAPI app: /api/report, /health, /runtime-config.js, /manifest.xml, static files
  auth.py            SSO token validation and On-Behalf-Of exchange (MSAL)
  graph.py           Microsoft Graph: MIME download, move to folder
  joesandbox.py      Joe Sandbox submission (jbxapi) and error classification
  retry.py           Retry with exponential back-off (408, 429, 5xx)
  errors.py          Error codes and task pane messages
  config.py          Settings from environment variables
  worker.py          Gunicorn worker class used on App Service
  manifest.xml       Add-in manifest (template)
  static/            Task pane, fallback sign-in dialog, icons, vendored MSAL Browser
startup.sh           App Service startup command: finds the app and starts gunicorn
deploy/
  azuredeploy.json   ARM template ("Deploy to Azure")
  source.zip         Package deployed by the ARM template (built by scripts/build_package.py)
scripts/             build_package.py, make_icons.py
tests/               pytest suite
```

**Set up**

```bash
python3.12 -m venv .venv
.venv/bin/pip install -r requirements-dev.txt
cp .env.example .env        # fill in the values
.venv/bin/python -m pytest
```

**Run locally.** Office add-ins must be served over HTTPS. For example, with a locally trusted certificate (from [mkcert](https://github.com/FiloSottile/mkcert)) in `certs/`:

```bash
.venv/bin/uvicorn app.main:app --env-file .env --port 8443 \
    --ssl-keyfile certs/localhost-key.pem --ssl-certfile certs/localhost.pem
```

Set `APP_DOMAIN=localhost:8443`, then sideload `https://localhost:8443/manifest.xml` in Outlook (**Get Add-ins → My add-ins → Add a custom add-in**). The App Registration needs the matching redirect URI and Application ID URI for `localhost:8443`, or a separate development registration.

**Release a new package.** Run `.venv/bin/python scripts/build_package.py`, then commit `deploy/source.zip`. The package contains:

- `app/`: the middle tier and the task pane
- `.python_packages/lib/site-packages/`: all dependencies, as Linux x86_64 wheels for Python 3.12, at the versions installed in `.venv`
- `startup.sh`: the startup command
- `.deployment`: tells App Service not to build on the server

App Service runs the package as is, with no server-side build. Existing Web Apps can be updated with *Deployment Center → Publish files* or `az webapp deploy --src-path deploy/source.zip --type zip`. Upload the zip exactly as built; do not unzip and re-zip it.

**Before releasing to customers**

Replace these placeholders:

- `app/static/assets/icon-*.png`: placeholder icons. Replace them with Joe Security artwork at the same sizes (16, 32, 64, 80, 128 px).
- `--brand` colours in `app/static/taskpane.css`.
- `SupportUrl` in `app/manifest.xml`.
- `{JOE_GITHUB_ORG}/{JOE_REPOSITORY}` in this README and in `deploy/azuredeploy.json`.

Then run the end-to-end tests: Outlook on Windows and Mac, Outlook on the web (Edge, Chrome, Firefox, Safari) and new Outlook, covering emails with and without attachments, a large email, SSO and the fallback sign-in dialog, move-to-folder on and off, an invalid API key, and Joe Sandbox unavailable.

---

## Version history

| Version | Release date | Notes |
| --- | --- | --- |
| 1.0.0 | TBD | Initial release |
