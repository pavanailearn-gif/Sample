# Context & Task: Upgrade ASP.NET Web Forms Application for WorkflowGen v10.5 & Microsoft Identity v2.0

## System Architecture & Constraints
- **Parent Platform:** WorkflowGen v10.5 (upgraded from v8.0.10) running on .NET Framework 4.8 / IIS.
- **Child Application:** Custom ASP.NET Web Forms application hosted on the same server, embedded inside the WorkflowGen portal via an iframe / custom portal menu (`portal.xml`).
- **Identity & Authentication:** Shared Microsoft Entra ID (Microsoft Identity Platform v2.0) via OpenID Connect (OIDC) and Bearer tokens.
- **Assembly Reference:** Uses `WorkflowGen.My.dll` (v10.5). Legacy assemblies (`WorkflowGen.My.Web.UI.WebControls.dll`, `Advantys.Security.dll`) have been removed.

## Current Problem / Regression
Following the upgrade from WorkflowGen v8 to v10.5, the custom ASP.NET Web Forms application fails to authenticate properly when framed inside the WorkflowGen portal. Specifically:
1. Modern browser security policies (`SameSite=Lax/Strict`, CSP, and `X-Frame-Options`) drop OIDC correlation/nonce cookies and `.ASPXAUTH` tokens inside the portal iframe context.
2. The code-behind needs to correctly inherit from `WorkflowGen.My.WorkflowPage` to extract the shared Microsoft Identity v2.0 Bearer access token managed by WorkflowGen v10.5 rather than relying on dropped session cookies.
3. Cryptographic and header settings between the main WorkflowGen app and this sub-app need strict alignment to prevent silent authentication redirects and iframe loading blocks.

---

## Instructions for Copilot

Please review the provided `.aspx` pages, `CodeBehind` (`.aspx.cs`), and `web.config` files in this repository and apply the following refactoring rules:

### 1. `web.config` Refactoring
- **Frame-Ancestors & Framing Headers:** Ensure `<system.webServer>/<httpProtocol>/<customHeaders>` removes restrictive `X-Frame-Options` and configures `SAMEORIGIN` alongside `Content-Security-Policy: frame-ancestors 'self' https://<YOUR_WORKFLOWGEN_DOMAIN>;`.
- **Cookie Security:** Configure `<system.web>/<httpCookies>` to set `sameSite="None"` and `requireSSL="true"` so that Microsoft Entra ID correlation/session cookies are preserved inside the portal iframe.
- **Authentication Mode:** Set `<authentication mode="None" />` inside `<system.web>` to allow the OIDC middleware and WorkflowGen token handoff to control identity without interfering with legacy Forms authentication.
- **MachineKey Cryptography:** Synchronize the `<machineKey>` element to use `validation="SHA256"` and `decryption="AES"` matching WorkflowGen v10.5 standards.

### 2. Code-Behind (`.aspx.cs`) Modernization
- Ensure code-behind classes inherit from `WorkflowGen.My.WorkflowPage`.
- Update the page execution lifecycle (`Page_Load`) to retrieve the authenticated user's active Microsoft Entra ID access token via `CurrentUserAccessToken()`.
- Validate that token verification, parameter passing (`dataIN.xml` / `dataOUT.xml`), and process completion via `SubmitToWorkflow()` comply with WorkflowGen v10.5 APIs.
- Eliminate obsolete `WorkflowGen.My.Web.UI.WebControls` namespace references.

---

## Action Plan Requested
1. Inspect my current `web.config` and propose the exact XML diff required to fix the framing and `SameSite` cookie blocks.
2. Refactor my code-behind (`.aspx.cs`) to cleanly handle Microsoft Identity v2.0 token extraction using `WorkflowGen.My.WorkflowPage`.
3. Highlight any broken or legacy API methods that are incompatible with WorkflowGen v10.5.
