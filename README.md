# Agentic AI Takes Over Commerce - a multi-party authorization problem

How do you securely **identify a human user**, **retrieve their pre-authorized third-party credentials**, and **confirm a financial transaction** across **two independent OAuth boundaries** — *in real time*, *transparently*, and inside an *external conversational AI interface* (such as *ChatGPT*, *Le Chat* or *Claude*)?

It's a **multi-party authorization problem** on how to Book and Pay a Uber (or Lyft) Ride Inside a 3rd party **conversational AI interface**? 

Each of these is a distinct protocol problem. None is automatically inherited from solving the others. **Auth0 is the architectural component that spans all three** — 
- as **authorization server**,
- **identity broker**,
- **credential vault**,
- and **confirmation orchestrator**

... making it the single most critical dependency in the entire stack.

### The Vision
Tomorrow, a user **opens ChatGPT**, types *"Book me an Uber to O'Hare"*, and the ride is confirmed — **without ever opening the Uber app**. 
- **ChatGPT becomes the interface**.
- **Uber** becomes a **backend service**.
- The AI agent orchestrates everything.

This is not a **UX convenience story**. It is a **multi-party authorization problem** with two distinct security boundaries that must be solved independently.

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

We need a mechanism that **bridges the ChatGPT session identity** to the **Uber account identity** without asking the user to **re-authenticate every time**. 

This is exactly what **Identity Federation** and specifically **Token Exchange** (RFC 8693) solves. This is where **Identity Jag** (**Id-Jag**) or equivalent **cross-app identity** patterns come in.

### What Auth0 Must Solve Across Both Boundaries

### ChatGPT SDK
The **ChatGPT SDK** lets developers bring their products directly into ChatGPT with **custom Ul components**, **API access**, and **user context** that can **persist** across chats. It's built on Model Context Protocol (**MCP**), which defines how ChatGPT communicates with our app through **tools**, **resources**, and **structured data**.

OpenAI chatgpt integrates with our **OAuth-protected MCP** server by performing **resource** and **authorization server discovery**, **dynamic client registration** (**DCR**), and a **PKCE-based authorization code flow** with **Auth0** to obtain a **JWT access token**, which our server verifies via **JWKS public keys** before allowing **SSE-based MCP tool execution** and seamless **token refresh** for ongoing secure access.

- The **OAuth flow** authenticates **OpenAI chatgpt** (**the client**) to our **MCP service** (**the resource provider**). 
- It does **NOT** **authenticate** or **identify the individual human** (OpenAI chatgpt's user) to us.
- We **won’t receive any user identity info** unless OpenAI chatgpt explictly passes it.
- OAuth by itself **does not identify a user**; it just **delegates authorization**.

In traditional web apps, we often combine **OAuth + OpenID Connect (OIDC)** to both **authenticate** and **authorize users**.
In the OpenAI chatgpt SDK integration, **only OAuth 2.1 is used** — **not OIDC.** So there’s **no user identity payload** (**no ID token**, **no claims** about the user).

# The Problem Statement
When ChatGPT connects to your MCP server, OAuth 2.0 authenticates the ChatGPT application — not the human behind it. The access token your MCP server receives proves that OpenAI's client is authorized to call your tools. It carries zero information about which specific human issued the command.
OpenAI's MCP integration uses OAuth 2.0 without OpenID Connect. No ID token is issued. No sub claim. No user profile. The human is invisible at the protocol level.
Your MCP server receives a legitimate, cryptographically valid token — and has no idea whose Uber account to charge.
# Boundary 1 — ChatGPT ↔ MCP Server

# Boundary 2 — MCP Server ↔ Uber API

# Bibliography

https://www.youtube.com/watch?v=ve7ulU01jI4
