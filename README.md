# PerfectOAuth2

Swift 6 OAuth2 client library with providers for Google, GitHub, Facebook, Slack, LinkedIn, and Salesforce. No external dependencies — Foundation and URLSession only.

**Resurrection status:** Complete. Original `Perfect-Authentication` (Swift 3, PerfectCURL, PerfectHTTP) rewritten for Swift 6 strict concurrency.

## Package

```swift
// Package.swift
.package(path: "../Perfect-Authentication"),

// target dependency
.product(name: "PerfectOAuth2", package: "Perfect-Authentication"),
```

```swift
import PerfectOAuth2
```

## Configuration

Set config before your server starts. All properties are `nonisolated(unsafe) static var` to satisfy Swift 6 global state rules.

```swift
GoogleConfig.appid    = "your-client-id"
GoogleConfig.secret   = "your-client-secret"
GoogleConfig.endpointAfterAuth = "https://yourapp.com/auth/response/google"
GoogleConfig.redirectAfterAuth = "https://yourapp.com/"

// Google only: restrict to a G Suite / Workspace domain
GoogleConfig.restrictedDomain = "yourcompany.com"

GitHubConfig.appid    = "your-client-id"
GitHubConfig.secret   = "your-client-secret"
GitHubConfig.endpointAfterAuth = "https://yourapp.com/auth/response/github"
GitHubConfig.redirectAfterAuth = "https://yourapp.com/"

FacebookConfig.appid   = "your-app-id"
FacebookConfig.secret  = "your-app-secret"
FacebookConfig.endpointAfterAuth = "https://yourapp.com/auth/response/facebook"
FacebookConfig.redirectAfterAuth = "https://yourapp.com/"

SlackConfig.appid   = "your-client-id"
SlackConfig.secret  = "your-client-secret"
SlackConfig.endpointAfterAuth = "https://yourapp.com/auth/response/slack"
SlackConfig.redirectAfterAuth = "https://yourapp.com/"

LinkedinConfig.appid   = "your-client-id"
LinkedinConfig.secret  = "your-client-secret"
LinkedinConfig.endpointAfterAuth = "https://yourapp.com/auth/response/linkedin"
LinkedinConfig.redirectAfterAuth = "https://yourapp.com/"

SalesForceConfig.appid   = "your-client-id"
SalesForceConfig.secret  = "your-client-secret"
SalesForceConfig.endpointAfterAuth = "https://yourapp.com/auth/response/salesforce"
SalesForceConfig.redirectAfterAuth = "https://yourapp.com/"
```

## Usage

The library is framework-agnostic. You provide two routes — one to redirect the user to the provider, one to handle the callback. Both are plain async functions; session management is your responsibility.

### Step 1: Redirect to provider

```swift
// PerfectNIO example — route: GET /to/google
let csrf = session.data["csrf"] as? String ?? ""
let loginURL = Google.loginURL(state: csrf, sessionToken: session.token)
// redirect the user to loginURL
```

Or use the instance API for custom scopes:

```swift
let g = Google(clientID: GoogleConfig.appid, clientSecret: GoogleConfig.secret)
let url = g.loginURL(state: csrf, sessionToken: session.token, scopes: ["profile", "email"])
```

### Step 2: Handle the callback

```swift
// Route: GET /auth/response/google?code=...&state=...
let code  = request.queryParam("code") ?? ""
let state = request.queryParam("state") ?? ""
let csrf  = session.data["csrf"] as? String ?? ""

let profile = try await Google.processAuthResponse(
    code: code,
    state: state,
    sessionCSRF: csrf,
    sessionToken: session.token
)

// Store in session
session.userid           = profile.userid
session.data["loginType"]  = profile.loginType
session.data["accessToken"]  = profile.accessToken
session.data["firstName"]    = profile.firstName ?? ""
session.data["lastName"]     = profile.lastName ?? ""
session.data["picture"]      = profile.picture ?? ""

// redirect to redirectAfterAuth
```

### OAuthUserProfile

`processAuthResponse` returns an `OAuthUserProfile`:

```swift
public struct OAuthUserProfile: Sendable {
    public let userid: String
    public let firstName: String?
    public let lastName: String?
    public let picture: String?
    public let accessToken: String
    public let refreshToken: String?
    public let loginType: String   // "google" | "github" | "facebook" | "slack" | "linkedin" | "salesforce"
}
```

### Manual token exchange (advanced)

If you need access to the raw token (e.g. to call provider APIs beyond the profile):

```swift
let provider = GitHub(clientID: GitHubConfig.appid, clientSecret: GitHubConfig.secret)
let token = try await provider.exchange(code: code, state: state, sessionToken: session.token)
let userdata = await provider.getUserData(token.accessToken)
```

## Providers

| Provider   | Default scopes                     | Notes |
|------------|------------------------------------|-------|
| Google     | `profile`                          | Add `restrictedDomain` to enforce G Suite domain |
| GitHub     | `user`                             | Uses `Authorization: Bearer` header (not deprecated query param) |
| Facebook   | _(none)_                           | Graph API v2.8 |
| Slack      | `identity.basic identity.avatar`   | |
| LinkedIn   | `openid profile`                   | v2 userinfo endpoint (OpenID Connect) |
| Salesforce | `id`                               | Token exchange returns `idURL`; `getUserData` fetches from that URL |

## Error handling

```swift
do {
    let profile = try await Google.processAuthResponse(...)
} catch let e as OAuth2Error {
    // e.code: OAuth2ErrorCode (.invalidGrant, .accessDenied, .unsupportedResponseType, ...)
    // e.description: human-readable message
} catch is InvalidAPIResponse {
    // provider returned unexpected JSON
}
```

`processAuthResponse` throws `OAuth2Error(code: .unsupportedResponseType)` if `state != sessionCSRF`.

## Future work

- **LocalAuthentication target** — the original package included a username/password auth system (account schema, email verification, SMTP). It was not resurrected because it depends on `Perfect-SMTP` and `Perfect-Mustache`, neither of which are resurrected yet. The source lives in `Sources/LocalAuthentication/` if that work is ever picked up.

- **Token refresh** — `OAuth2Token.refreshToken` is captured from the provider response but there is no `refresh(token:)` method on the base class or providers. Implement when long-lived sessions are needed.

- **LinkedIn email scope** — add `email` to the default scopes and extract `email` from the `/v2/userinfo` response if email is needed.

- **Salesforce sandbox** — `SalesForce` hardcodes `login.salesforce.com`. Add `SalesForceConfig.domain` to support sandbox (`test.salesforce.com`) or My Domain instances.

- **Facebook Graph API version** — currently pinned to `v2.8`. Facebook's minimum supported version changes over time; bump to a current version (v21+) when updating.

- **NIO session middleware integration** — the OAuth callback pattern (extract code/state, call processAuthResponse, write to session, redirect) is repetitive. A `PerfectNIOOAuth2` target in Perfect-NIO could provide pre-wired route handlers that accept a session driver and config.
