# MSAL.js Login Hint Sample — Seamless Logout with Microsoft Entra ID

This sample demonstrates how to use the **`login_hint`** optional claim from the ID token to enable **seamless sign-out** in a Single-Page Application (SPA) using **MSAL.js v2 (msal-browser)**. When `login_hint` is passed as `logoutHint`, the user is signed out **without being prompted** to select which account to log out of — delivering a smooth, frictionless experience.

---

## Table of Contents

1. [Why `login_hint` Matters](#why-login_hint-matters)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Install and Configure Live Server in VS Code](#step-1--install-and-configure-live-server-in-vs-code)
4. [Step 2 — Register an Application in Microsoft Entra ID](#step-2--register-an-application-in-microsoft-entra-id)
5. [Step 3 — Add a SPA Redirect URI in the App Registration](#step-3--add-a-spa-redirect-uri-in-the-app-registration)
6. [Step 4 — Enable the `login_hint` Optional Claim](#step-4--enable-the-login_hint-optional-claim)
7. [Step 5 — Update Client ID and Tenant ID in the Code](#step-5--update-client-id-and-tenant-id-in-the-code)
8. [Step 6 — Run the Sample](#step-6--run-the-sample)
9. [Code Walkthrough](#code-walkthrough)
10. [Summary of Steps](#summary-of-steps)
11. [Microsoft Learn References](#microsoft-learn-references)

---

## Why `login_hint` Matters

When a user signs out of a web application that uses Microsoft Entra ID, the default behaviour is to display an **"Pick an account"** prompt — asking the user which account they want to sign out of. This is disruptive, especially in enterprise scenarios where the user has only one session active.

The **`login_hint`** optional claim solves this:

| Without `login_hint` | With `login_hint` |
|---|---|
| User clicks "Sign Out" | User clicks "Sign Out" |
| Microsoft shows "Pick an account" screen | **No prompt — user is signed out immediately** |
| User selects the account manually | Redirect happens automatically |
| User is signed out | User is signed out |

By emitting the `login_hint` claim in the ID token and passing it as `logoutHint` in the MSAL logout request, the application tells Microsoft Entra ID **exactly** which account to sign out — eliminating the account picker entirely.

> **Key takeaway for app owners:** Enabling `login_hint` as an optional claim in the app registration and using it during logout is a simple configuration change that significantly improves the end-user sign-out experience.

---

## Prerequisites

- **Visual Studio Code** installed — [Download VS Code](https://code.visualstudio.com/)
- **Live Server** extension for VS Code (explained below)
- Access to the **Microsoft Entra admin center** (https://entra.microsoft.com) with permissions to register applications
- A Microsoft Entra ID tenant (any tier — Free, P1, or P2)

---

## Step 1 — Install and Configure Live Server in VS Code

**Live Server** is a VS Code extension that launches a local development web server with live-reload capability. This sample requires a local HTTPS/HTTP server because MSAL.js needs a proper origin (not `file://`).

### Install Live Server

1. Open **Visual Studio Code**.
2. Go to the **Extensions** view by pressing `Ctrl+Shift+X`.
3. Search for **"Live Server"** by Ritwick Dey.
4. Click **Install**.

### Configure Live Server Port (Optional)

By default, Live Server runs on port **5500** (`http://127.0.0.1:5500`). This sample's code uses `https://localhost:5500` as the redirect URI. If you want to use a different port:

1. Open VS Code **Settings** (`Ctrl+,`).
2. Search for `liveServer.settings.port`.
3. Change the port number (e.g., `3000`).
4. **Important:** If you change the port, you must also update the redirect URI in both the code **and** the Entra ID app registration.

### Launch the Sample with Live Server

1. Open the project folder in VS Code (`File > Open Folder` → select the folder containing `index.html`).
2. Right-click on `index.html` in the Explorer panel.
3. Select **"Open with Live Server"**.
4. Your browser will open at `http://127.0.0.1:5500/index.html`.

> **Note:** The `redirectUri` in the code uses `window.location.origin` as a fallback, so the sample will work with whatever origin Live Server serves.

---

## Step 2 — Register an Application in Microsoft Entra ID

If you don't already have an app registration, create one:

1. Sign in to the **Microsoft Entra admin center** → https://entra.microsoft.com.
2. Navigate to **Identity** → **Applications** → **App registrations**.
3. Click **+ New registration**.
4. Fill in the details:
   - **Name:** e.g., `Login Hint Sample SPA`
   - **Supported account types:** Choose the appropriate option (e.g., *Accounts in this organizational directory only* for single-tenant).
   - **Redirect URI:** Skip for now — we'll add this in the next step.
5. Click **Register**.
6. On the **Overview** page, note down:
   - **Application (client) ID** — you'll need this for the code.
   - **Directory (tenant) ID** — you'll need this for the authority URL.

---

## Step 3 — Add a SPA Redirect URI in the App Registration

The application must have a **Single-page application (SPA)** redirect URI that matches the URL where Live Server serves the page.

1. In the app registration, go to **Authentication** (left menu).
2. Under **Platform configurations**, click **+ Add a platform**.
3. Select **Single-page application**.
4. In the **Redirect URIs** field, enter:
   ```
   http://localhost:5500
   ```
   > If you also want HTTPS or a different port, add those URIs as well (e.g., `https://localhost:5500`).
5. Click **Configure**.

### Important Notes

- The redirect URI **must exactly match** the origin from which the application runs. A mismatch will cause a `redirect_uri_mismatch` error.
- Do **not** add the URI under "Web" platform — it must be under **Single-page application**. Using the wrong platform type will cause MSAL.js to fail with a CORS error.
- You can add multiple redirect URIs (e.g., `http://localhost:5500`, `http://127.0.0.1:5500`) if needed.

---

## Step 4 — Enable the `login_hint` Optional Claim

This is the **critical step** that makes seamless logout possible. By default, the `login_hint` claim is **not included** in the ID token. You must enable it explicitly.

1. In the app registration, go to **Token configuration** (left menu).
2. Click **+ Add optional claim**.
3. Select **ID** as the token type.
4. In the claims list, check **`login_hint`**.
5. Click **Add**.
6. If prompted to add the required Microsoft Graph permissions, check the box and click **Add**.

> **Without this step**, `account.idTokenClaims.login_hint` will be `undefined`, and the sample will fall back to normal logout **with** the account picker prompt.

---

## Step 5 — Update Client ID and Tenant ID in the Code

Open `index.html` and locate the MSAL configuration block (around line 96):

```javascript
const msalConfig = {
    auth: {
        clientId: "xxxxx",           // <-- Replace with YOUR Application (client) ID
        authority: "https://login.microsoftonline.com/xxxxxx", // <-- Replace the GUID with YOUR Directory (tenant) ID
        redirectUri: "https://localhost:5500" | window.location.origin,
    },
    ...
};
```

### What to Change

| Property | What to Replace | Where to Find It |
|---|---|---|
| `clientId` | Replace `2f5dade9-2b9b-483c-bdce-de11dcad239e` with your **Application (client) ID** | Entra admin center → App registrations → your app → Overview |
| `authority` | Replace `e95f1b23-abaf-45ee-821d-b7ab251ab3bf` with your **Directory (tenant) ID** | Entra admin center → App registrations → your app → Overview |
| `redirectUri` | Update `https://localhost:5500` to match your Live Server URL (if different) | Your Live Server port configuration |
| `postLogoutRedirectUri` | Update `https://localhost:5500/index.html` in the `signOut()` function (~line 152) to match your URL | Same as above |

> **Tip:** If you use `"common"` or `"organizations"` as the authority instead of a specific tenant ID, the app will accept sign-ins from any Microsoft Entra tenant or personal Microsoft accounts respectively.

---

## Step 6 — Run the Sample

1. Open the folder in VS Code.
2. Right-click `index.html` → **Open with Live Server**.
3. Click **Sign In** — a popup window opens for Microsoft authentication.
4. Sign in with your Microsoft account.
5. After successful sign-in, the page displays:
   - Your **display name**
   - Your **username**
   - The **`login_hint` claim** value (if configured in Step 4)
   - A table of all **ID token claims**
6. Click **Sign Out** — if `login_hint` is present, you will be signed out **immediately without any prompt**. If it is not present, Microsoft will show the account picker.

---

## Code Walkthrough

### HTML Structure (Lines 1–90)

The page contains two card sections:

- **Login Card** — Contains the "Sign In" and "Sign Out" buttons. The Sign Out button is hidden until the user is authenticated.
- **User Info Card** — Hidden by default. After sign-in, it displays the user's name, username, `login_hint` claim value, and a table of all ID token claims.

### MSAL.js Library (Line 93)

```html
<script type="text/javascript" src="https://alcdn.msauth.net/browser/2.35.0/js/msal-browser.min.js"></script>
```

The MSAL.js v2 library (`msal-browser`) is loaded from the Microsoft CDN. This library handles all OAuth 2.0 / OpenID Connect flows for SPAs.

### MSAL Configuration (Lines 96–108)

```javascript
const msalConfig = {
    auth: {
        clientId: "...",       // Your app's Application (client) ID
        authority: "https://login.microsoftonline.com/{tenant-id}",
        redirectUri: "https://localhost:5500" | window.location.origin,
    },
    cache: {
        cacheLocation: "sessionStorage",  // Tokens stored in session storage
        storeAuthStateInCookie: false,     // No cookie fallback
    },
};
```

- **`clientId`** — Identifies your application to Microsoft Entra ID.
- **`authority`** — Specifies the Entra ID tenant endpoint for authentication.
- **`redirectUri`** — Where Entra ID sends the user back after sign-in.
- **`cacheLocation`** — Set to `sessionStorage` so tokens are cleared when the browser tab closes.

### Login Request Scopes (Lines 110–112)

```javascript
const loginRequest = {
    scopes: ["openid", "profile", "User.Read"],
};
```

- **`openid`** — Requests an ID token (required for authentication).
- **`profile`** — Requests the user's profile information (name, etc.).
- **`User.Read`** — Requests permission to read the signed-in user's Microsoft Graph profile.

### MSAL Instance & Redirect Handling (Lines 115–122)

```javascript
const msalInstance = new msal.PublicClientApplication(msalConfig);

msalInstance.handleRedirectPromise()
    .then(handleResponse)
    .catch(function (error) { ... });
```

- Creates the MSAL public client instance.
- `handleRedirectPromise()` processes any authentication response if the page was loaded via a redirect (this is required even when using popups).

### `handleResponse()` — Process Auth Response (Lines 127–137)

```javascript
function handleResponse(response) {
    if (response) {
        updateUI(response.account);
    } else {
        const accounts = msalInstance.getAllAccounts();
        if (accounts.length > 0) {
            updateUI(accounts[0]);
        }
    }
}
```

- If a fresh authentication response exists, the UI is updated with the account info.
- Otherwise, it checks if the user already has a cached session (e.g., after a page refresh).

### `signIn()` — Popup Login (Lines 142–150)

```javascript
function signIn() {
    msalInstance.loginPopup(loginRequest)
        .then(function (response) {
            updateUI(response.account);
        })
        .catch(function (error) { ... });
}
```

Opens a popup window for the Microsoft sign-in experience. On success, the account information is passed to `updateUI()`.

### `signOut()` — Seamless Logout with `login_hint` (Lines 156–179)

**This is the core of the sample.**

```javascript
function signOut() {
    const account = msalInstance.getAllAccounts()[0];
    if (!account) return;

    // Extract the login_hint claim from the ID token claims
    const loginHint = account.idTokenClaims && account.idTokenClaims.login_hint;

    const logoutRequest = {
        account: account,
        postLogoutRedirectUri: "https://localhost:5500/index.html" || window.location.origin,
    };

    // If login_hint claim is available, use it as logoutHint
    if (loginHint) {
        logoutRequest.logoutHint = loginHint;
    }

    msalInstance.logoutPopup(logoutRequest)
        .then(function () { resetUI(); })
        .catch(function (error) { ... });
}
```

**Step-by-step breakdown:**

1. **Get the current account** from MSAL's cache.
2. **Extract `login_hint`** from `account.idTokenClaims.login_hint` — this claim is an opaque string that identifies the user's session with Microsoft Entra ID.
3. **Build the logout request** with the account and a post-logout redirect URI.
4. **Set `logoutHint`** — If the `login_hint` claim is available, it is assigned to `logoutRequest.logoutHint`. This is the property that tells Microsoft Entra ID which account to sign out **without** showing the account picker.
5. **Call `logoutPopup()`** — Opens a popup that signs the user out of Microsoft Entra ID. Because `logoutHint` is provided, the popup closes automatically without user interaction.

> **Without `logoutHint`:** The user sees a "Pick an account" page and must manually select their account.
>
> **With `logoutHint`:** The logout flow completes silently — the popup opens, signs the user out, and closes automatically.

### `updateUI()` — Display User Info (Lines 184–204)

Populates the user interface with the signed-in user's name, username, `login_hint` value, and renders all ID token claims in a table.

### `resetUI()` — Clear After Sign-Out (Lines 209–213)

Restores the page to its initial state by showing the Sign In button and hiding the user info card.

---

## Summary of Steps

| # | Step | Where |
|---|---|---|
| 1 | Install the **Live Server** extension in VS Code | VS Code Extensions panel (`Ctrl+Shift+X`) |
| 2 | **Register an application** in Microsoft Entra ID | Entra admin center → App registrations |
| 3 | Add a **SPA redirect URI** (`http://localhost:5500`) | App registration → Authentication → Single-page application |
| 4 | Enable the **`login_hint` optional claim** on the ID token | App registration → Token configuration → Add optional claim |
| 5 | Update **`clientId`** and **`authority`** (tenant ID) in `index.html` | `index.html` — `msalConfig` object (~line 99–100) |
| 6 | Update **`postLogoutRedirectUri`** if your port differs | `index.html` — `signOut()` function (~line 166) |
| 7 | **Open with Live Server** and test sign-in / sign-out | Right-click `index.html` → Open with Live Server |

---

## Microsoft Learn References

- [Optional claims — `login_hint`](https://learn.microsoft.com/en-us/entra/identity-platform/optional-claims-reference) — Full reference of optional claims including `login_hint`.
- [Configure optional claims in your app](https://learn.microsoft.com/en-us/entra/identity-platform/optional-claims) — How to add optional claims in the Entra admin center.
- [MSAL.js — Single sign-out](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-spa-sign-in?tabs=javascript2#sign-out-with-a-popup-window) — MSAL.js sign-out with popup documentation.
- [MSAL.js — `logoutHint` / `login_hint`](https://learn.microsoft.com/en-us/entra/identity-platform/msal-js-pass-custom-state-authentication-request) — Passing hints to the logout request.
- [Register a SPA in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-spa-app-registration) — Step-by-step SPA registration guide.
- [MSAL.js v2 (msal-browser) overview](https://learn.microsoft.com/en-us/entra/identity-platform/msal-overview) — Overview of the Microsoft Authentication Library.
- [Quickstart: Sign in users in a SPA](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-single-page-app-javascript-sign-in) — End-to-end quickstart for JavaScript SPAs.

---

## License

This sample is provided as-is for demonstration purposes.
