# Agentic AI Takes Over Commerce - a multi-party authorization problem

How do you securely **identify a human user**, **retrieve their pre-authorized third-party credentials**, and **confirm a financial transaction** across **two independent OAuth boundaries** — *in real time*, *transparently*, and inside an *external conversational AI interface* (such as *ChatGPT*, *Le Chat* or *Claude*)?

It's a **multi-party authorization problem** on how to Book and Pay a Uber (or Lyft) Ride Inside a 3rd party **conversational AI interface**.

### The Vision
Tomorrow, a user **opens ChatGPT**, types *"Book me an Uber to O'Hare"*, and the ride is confirmed — **without ever opening the Uber app**. 
- **ChatGPT becomes the interface**.
- **Uber** becomes a **backend service**.
- The AI agent orchestrates everything.

This is not a **UX convenience story**. It is a **multi-party authorization problem** with **3 very distinct security boundaries** that must be solved independently.

- 🚗 Book the cheapest Uber or Lyft—right inside ChatGPT - https://www.youtube.com/shorts/5ZI7IgvJHV8
- 💐 Send flowers instantly from Mistral AI Le Chat - https://youtu.be/671YMGWVHL0
- 🍔 Order your favorite nearby burgers with Anthropic Claude - https://www.youtube.com/shorts/Cy-N7jy_BsQ

I implemented a **secure**, **OAuth-protected MCP server** that enables any AI Assistant to **discover products**, **execute commerce tools**, and **complete end-to-end Agentic Commerce transactions** through a standardized and trusted protocol.

### The Core Problem We're Solving
A user saying "book me an Uber" inside ChatGPT triggers a **3-party authorization chain**: 

1. ChatGPT must prove its **application identity** to our MCP server (Boundary 1, solved via PKCE + Auth0-issued JWT),
2. ...our MCP server must **resolve which human issued the command** and retrieve that **human's pre-authorized Uber credentials** (identity gap, solved via RFC 8693 Token Exchange + Auth0 Token Vault),
3. ...and before the actual booking call is made, the **user must explicitly confirm the financial transaction** on a separate channel without leaving ChatGPT (confirmation gap, solved via CIBA).

- **ChatGPT** can talk to **external services** through **MCP**.
- **Uber** exposes **ride-booking** through an **OAuth-protected API**.
- **Auth0** can **broker identity** and **credentials**. 

But connecting these **3 systems into a single seamless user action** — "book me an Uber" — requires solving an **identity chain** that does not exist out of the box.
The chain breaks in 3 specific places: 

1. Break 1 — ChatGPT is **authenticated**, but **the user is not**
2. Break 2 — Our MCP server has **no standing** with Uber
3. Break 3 — A financial transaction requires **explicit user confirmation**

<img width="1096" height="320" alt="image" src="https://github.com/user-attachments/assets/825344c6-2ddc-41ae-b9f8-fa120bc82dc3" />

We need a mechanism that **bridges the ChatGPT session identity** to the **Uber account identity** without asking the user to **re-authenticate every time**. 

~~This is exactly what **Identity Federation** and specifically **Token Exchange** (RFC 8693) solves. This is where **Identity Jag** (**Id-Jag**) or equivalent **cross-app identity** patterns come in.~~

Each of these is a distinct protocol problem. None is automatically inherited from solving the others. **Auth0 is the architectural component that spans all three** — 
- as **authorization server**,
- **identity broker**,
- **credential vault**,
- and **confirmation orchestrator**

... making it the single most critical dependency in the entire stack.

| # | Problem | Protocol gap | Consequence if unsolved |
|---|---|---|---|
| 1 | ChatGPT is authenticated but the human user is not identified | OAuth 2.1 without OIDC carries no user identity | MCP server cannot map the request to a specific Uber account |
| 2 | MCP server has no Uber credentials for the user | Uber tokens are user-scoped, issued separately, must be stored and refreshed | Ride cannot be booked regardless of Boundary 1 being correctly configured |
| 3 | Financial transaction requires explicit out-of-band user confirmation | Neither OAuth nor MCP provide a transaction confirmation primitive | Real money moves without verified user intent — compliance and fraud risk |

# The Problem Statement

### Break 1 — ChatGPT is authenticated, but the user is not
When ChatGPT connects to our MCP server, **OAuth 2.1 authenticates the ChatGPT application** — **not the human behind it**. 

The access token your MCP server receives proves that **OpenAI's client is authorized to call our tools**. It carries **zero information** about which specific human issued the command.
OpenAI's MCP integration uses OAuth 2.1 **without OpenID Connect**. No `ID token` is issued. No sub `claim`. No `user profile`. **The human is invisible at the protocol level**.
Your MCP server receives a **legitimate**, **cryptographically valid token** — and has **no idea whose Uber account to charge**.

Here is the explanation: https://youtu.be/qwtwGqpXluE

### Break 2 — our MCP server has no standing with Uber
Even if you resolve the user's identity, your MCP server **cannot call Uber's API on their behalf without a user-scoped Uber OAuth token** — one that was issued specifically because that user went through Uber's own consent screen and explicitly authorized your application to book rides on their account.
That token does not exist automatically. It must be obtained, stored securely, refreshed before expiry, and retrieved at request time — for every individual user, independently. 

If it is missing, expired, or stored incorrectly, the ride cannot be booked regardless of how well Boundary 1 is configured.
The Auth0 JWT from Boundary 1 and the Uber OAuth token from Boundary 2 are issued by different authorization servers, scoped to different resources, and govern completely different trust relationships. They cannot substitute for each other.

### Break 3 — A financial transaction requires explicit user confirmation
Fare estimates and ETA lookups are low-risk reads. Actually booking a ride — triggering a real financial transaction against a real payment method — is a different category of action entirely. Silent background authorization is insufficient and, depending on jurisdiction, potentially non-compliant.
The user must confirm the specific transaction, on a separate channel, in a way that is auditable and non-repudiable. This confirmation must happen without breaking the conversational flow inside ChatGPT and without requiring the user to open another app.

# What Auth0 Must Solve Across Both Boundaries

```
BOUNDARY 1                          BOUNDARY 2
ChatGPT ──────────────▶ MCP Server ──────────────▶ Uber API

Problem: App is known,     Problem: User is now        Problem: Uber token
user is unknown            known, but Uber token        must be confirmed
                           is missing or expired        before money moves

Auth0 fix:                 Auth0 fix:                   Auth0 fix:
RFC 8693 Token Exchange    Token Vault stores and        CIBA — push
resolves ChatGPT session   auto-refreshes per-user       confirmation to
token into a user-         Uber OAuth tokens,            user's phone,
identity-bearing token     retrieved on demand           out-of-band
```

# What is blocked — and why

#### OpenAI does not expose user identity through the MCP layer.
**RFC 8693 Token Exchange** works only if Auth0 can resolve the incoming ChatGPT token to a known user. 
Currently, **OpenAI does not pass verifiable user identity claims through the MCP connection**. 
We can work around this — but it requires either **OpenAI adding OIDC support**, or **a separate user-linking** step during onboarding that correlates the **ChatGPT session** to our **internal user record**. Doable, but not clean.

#### Uber's API access requires business approval.
Uber's ride-request API (`POST /v1.2/requests`) is **not publicly open**. 
Uber must **explicitly grant our application access to book rides on behalf of users**. This is a commercial and legal dependency — not a technical one. Without it, Boundary 2 cannot go to production regardless of how well everything else is built.

### ChatGPT SDK
The **ChatGPT SDK** lets developers bring their products directly into ChatGPT with **custom Ul components**, **API access**, and **user context** that can **persist** across chats. It's built on Model Context Protocol (**MCP**), which defines how ChatGPT communicates with our app through **tools**, **resources**, and **structured data**.

OpenAI chatgpt integrates with our **OAuth-protected MCP** server by performing **resource** and **authorization server discovery**, **dynamic client registration** (**DCR**), and a **PKCE-based authorization code flow** with **Auth0** to obtain a **JWT access token**, which our server verifies via **JWKS public keys** before allowing **SSE-based MCP tool execution** and seamless **token refresh** for ongoing secure access.

- The **OAuth flow** authenticates **OpenAI chatgpt** (**the client**) to our **MCP service** (**the resource provider**). 
- It does **NOT** **authenticate** or **identify the individual human** (OpenAI chatgpt's user) to us.
- We **won’t receive any user identity info** unless OpenAI chatgpt explictly passes it.
- OAuth by itself **does not identify a user**; it just **delegates authorization**.

In traditional web apps, we often combine **OAuth + OpenID Connect (OIDC)** to both **authenticate** and **authorize users**.
In the OpenAI chatgpt SDK integration, **only OAuth 2.1 is used** — **not OIDC.** So there’s **no user identity payload** (**no ID token**, **no claims** about the user).


# Bibliography

https://www.youtube.com/watch?v=ve7ulU01jI4
