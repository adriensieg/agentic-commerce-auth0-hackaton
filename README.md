# Agentic AI Takes Over Commerce - a multi-party authorization problem

A multi-party authorization problem on how to Book and Pay a Uber (or Lyft) Ride Inside ChatGPT thanks to Auth0.

### The Vision
Tomorrow, a user **opens ChatGPT**, types *"Book me an Uber to O'Hare"*, and the ride is confirmed — **without ever opening the Uber app**. 
- **ChatGPT becomes the interface**.
- **Uber** becomes a **backend service**.
- The AI agent orchestrates everything.

This is not a **UX convenience story**. It is a **multi-party authorization problem** with two distinct security boundaries that must be solved independently.

### The Core Problem We're Solving
When a user books an Uber inside ChatGPT, **3 identity questions** must be **answered simultaneously**:

- **Who is ChatGPT (the application)?** — solved by Boundary 1 / Auth0
- **Who is the human user?** — not solved by OAuth alone
- **Does that human have an Uber account and has Uber authorized this?** — solved by Boundary 2 / Token Vault

The gap is question 2. We need a mechanism that **bridges the ChatGPT session identity** to the **Uber account identity** without asking the user to **re-authenticate every time**. 

This is exactly what **Identity Federation** and specifically **Token Exchange** (RFC 8693) solves. This is where **Identity Jag** (**Id-Jag**) or equivalent **cross-app identity** patterns come in.

### ChatGPT SDK
The **ChatGPT SDK** lets developers bring their products directly into ChatGPT with **custom Ul components**, **API access**, and **user context** that can **persist** across chats. It's built on Model Context Protocol (**MCP**), which defines how ChatGPT communicates with our app through **tools**, **resources**, and **structured data**.

OpenAI chatgpt integrates with our **OAuth-protected MCP** server by performing **resource** and **authorization server discovery**, **dynamic client registration** (**DCR**), and a **PKCE-based authorization code flow** with **Auth0** to obtain a **JWT access token**, which our server verifies via **JWKS public keys** before allowing **SSE-based MCP tool execution** and seamless **token refresh** for ongoing secure access.

- The **OAuth flow** authenticates **OpenAI chatgpt** (**the client**) to our **MCP service** (**the resource provider**). 
- It does **NOT** **authenticate** or **identify the individual human** (OpenAI chatgpt's user) to us.
- We **won’t receive any user identity info** unless OpenAI chatgpt explictly passes it.
- OAuth by itself **does not identify a user**; it just **delegates authorization**.

In traditional web apps, we often combine **OAuth + OpenID Connect (OIDC)** to both **authenticate** and **authorize users**.
In the OpenAI chatgpt SDK integration, **only OAuth 2.1 is used** — **not OIDC.** So there’s **no user identity payload** (**no ID token**, **no claims** about the user).

# Boundary 1 — ChatGPT ↔ MCP Server

# Boundary 2 — MCP Server ↔ Uber API

# Bibliography

https://www.youtube.com/watch?v=ve7ulU01jI4
