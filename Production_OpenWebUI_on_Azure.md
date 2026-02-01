# Production OpenWebUI on Azure: From Zero to Enterprise SSO in 48 Hours

---

## The Problem Nobody Warns You About

I was working on deploying a production-ready AI chat platform for a enterprise environment. The requirements were straightforward on paper: self-hosted LLM interface, corporate single sign-on, persistent user data, and the ability to scale. OpenWebUI checked every box. Azure Container Apps was the obvious hosting choice. I had the architecture in my head within an hour.

Within 24 hours, I was staring at a broken application that would log me in, then silently forget I existed. Every tutorial I found glossed over the exact thing that was destroying my deployment. It took me the better part of two days to understand why, and the fix was embarrassingly simple once I knew what was actually happening.

This article is about what went wrong, why it went wrong, and the architectural decisions that matter when you move beyond a local demo. If you need hands-on implementation support for your own deployment, reach out — this is exactly the kind of work I do through Beyontomoro.

---

## Architecture Overview

Before diving into the details, here is the full picture of what the final deployment looks like. Every component below has a reason for being there, and I will explain each one as we go.

```
┌─────────────────────────────────────────────────────────┐
│                   Azure Subscription                     │
│                                                         │
│  ┌──────────┐     ┌─────────────┐    ┌───────────────┐  │
│  │  Entra   │───▶│  OpenWebUI  │───▶│  Azure AI     │  │
│  │  ID      │     │  Container  │    │  Foundry      │  │
│  │ (SSO)    │     │  App        │    │  (GPT-4o,     │  │
│  └──────────┘     └──────┬──────┘    │   Claude,     │  │
│                          │           │   Llama)      │  │
│                          │           └───────────────┘  │
│                          ▼                              │
│                   ┌──────────────┐                      │
│                   │  PostgreSQL  │                      │
│                   │  Flexible    │                      │
│                   │  Server      │                      │
│                   └──────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

The flow is simple: users authenticate via Microsoft Entra ID, land on OpenWebUI running inside a Container App, which talks to PostgreSQL for all persistent data and routes LLM requests to models deployed on Azure AI Foundry. Each of these connections has gotchas. I will walk through the ones that actually cost me time.

---

## Step 1: Deploying OpenWebUI on Container Apps

Container Apps is the right choice here. It handles scaling, SSL, and container lifecycle for you. No VM to manage, no OS to patch.

### Why Not a VM?

A VM would work, but you are paying for idle compute, managing updates manually, and losing the serverless scaling that Container Apps gives you for free. For a chat application with unpredictable usage patterns, Container Apps is the pragmatic choice.

### Deployment Configuration

OpenWebUI ships as a Docker image on GitHub Container Registry. Deploying it to Container Apps is straightforward, pull the image, set the target port, configure ingress as external. The Azure CLI makes this a single command, and the Container Apps portal does it in a few clicks. Either way, you are up in minutes.

The settings that actually matter at this stage are around compute and scaling. The compute allocation needs to be sized carefully for this workload, too little and you hit throttling under concurrent users, too much and you are wasting money. I tuned mine based on the expected concurrent load and adjusted from there. For scaling, the replica configuration is where things get interesting, and where most deployments quietly break. I will come back to why shortly.

At this point, OpenWebUI will deploy and load. It will even let you create an admin account and start a conversation. Everything will appear to work. It does not.

---

## Step 2: The Replica Trap — Why SQLite Silently Destroys Your Deployment

This is the part I wish someone had told me before I spent hours debugging 401 errors at midnight.

### What Actually Happens

By default, OpenWebUI stores all data, i.e.,  user accounts, sessions, chat history in a SQLite database file sitting inside the container. When you run a single container, this works fine. The moment you scale to two or more replicas, it breaks completely.

Here is why: each replica is an independent container with its own filesystem. That means each replica has its own separate SQLite database. They share nothing.

When you log in, your session is created on whichever replica the load balancer routes you to, say Replica 1. Your admin account exists only in Replica 1's SQLite file. The next request might land on Replica 2, which has never seen your account. You get a 401 error. You refresh. It works. You refresh again. It fails. The load balancer is randomly distributing your requests across two replicas that have no shared state.

```
User Request ──▶ Load Balancer ──▶ Replica 1 (has your session) ✓
User Request ──▶ Load Balancer ──▶ Replica 2 (no session)       ✗ 401
User Request ──▶ Load Balancer ──▶ Replica 1 (has your session) ✓
User Request ──▶ Load Balancer ──▶ Replica 2 (no session)       ✗ 401
```

### The Temporary Fix That Confirms the Problem

If you force both min and max replicas to 1, the app works perfectly. Everything persists, logins stick, no errors. This confirms the diagnosis immediately. But it is not a production solution as you have zero redundancy, and any container restart wipes your data.

### An Important Misconception

Setting min replicas to 2 does not prevent restarts. It only guarantees that two instances are running. Azure can still restart individual containers for maintenance, health check failures, or platform updates. Min replicas is about availability, not stability of a single instance. I learned this the hard way.

---

## Step 3: PostgreSQL — The Actual Fix

The solution is to move all persistent data out of the container and into a shared database that every replica can access. Azure Database for PostgreSQL Flexible Server is the right tool for this.

### Why PostgreSQL Matters Here

This is not just about persistence. PostgreSQL is what makes multi-replica deployments possible. All replicas connect to the same external database, so it does not matter which replica handles your request, the session and user data are always there. It also survives container restarts, which SQLite inside a container does not.

### Setting Up PostgreSQL

The setup itself is not complicated. Create a Flexible Server instance in the same region as your Container App. The SKU tier matters here. You do not need anything powerful for OpenWebUI's query patterns, but picking the wrong tier either wastes money or creates performance issues under load. I sized mine based on the number of concurrent users and the chat history volume we expected at rollout. Region alignment matters too. Cross-region database calls add latency, and in some configurations, cost.

For a production enterprise environment, you will want private endpoints or VNet integration rather than public access. Getting the networking right in this context is non-trivial. There are firewall rules, IP detection quirks in the Azure Portal, and subnet configurations that need to align precisely. I hit several dead ends here before the connectivity worked cleanly.

### Connecting OpenWebUI to PostgreSQL

OpenWebUI switches from SQLite to PostgreSQL automatically when it detects the right environment variable on the Container App. A single variable is all it takes, no code changes, no migration scripts. But the connection string has to be constructed exactly right, with the correct format, authentication parameters, and SSL configuration. Get any part of it wrong and OpenWebUI silently falls back to SQLite, which brings you right back to the replica trap.

There is one thing that tripped me up more than once during this process. The way environment variables are entered into Container Apps can introduce invisible characters that break the connection string without any obvious error message. I spent more time on this than I would like to admit before I figured out what was going wrong. Once PostgreSQL is connected properly, scale back to multiple replicas. The 401 errors disappear entirely.

---

## Step 4: Enterprise SSO with Microsoft Entra

For a corporate deployment, you do not want users creating their own accounts. You want them to log in with their existing corporate credentials. Microsoft Entra handles this via OAuth2 and OIDC, and OpenWebUI supports it natively, but the configuration requires some care.

### App Registration

You need an app registration in Entra for OpenWebUI. The platform type and redirect URI have to match exactly what OpenWebUI expects on its end, if there is any mismatch in the callback path, the OAuth flow fails silently and you are left wondering why the sign-in button does nothing. I got this wrong on my first attempt and it took a while to diagnose.

### Environment Variables

OpenWebUI picks up SSO configuration through a set of environment variables on the Container App. There are several of them, and each one maps to a specific value from your Entra app registration, i.e., the client ID, the client secret, the tenant-specific provider URL, and the scope configuration. Getting the provider URL format right is particularly easy to get wrong. Once these are correctly in place, the default login screen is replaced with a corporate sign-in button.

### Web vs SPA App Registration — A Subtle but Critical Distinction

This one caught me out. If you later need to integrate OneDrive or SharePoint access, reading files from your organisation's storage, that requires a *separate* app registration of type SPA (Single Page Application), not Web. The two serve different purposes and cannot be combined into one registration cleanly. The token flows are fundamentally different between the two types, and trying to use a single Web registration for both authentication and file access will fail in ways that are not immediately obvious. Wiring both registrations together in the same deployment requires careful coordination. Plan for this if file access is on your roadmap.

---

## Step 5: Connecting Azure AI Foundry Models

OpenWebUI is model-agnostic. It speaks the OpenAI-compatible API format, and Azure AI Foundry exposes its models through that interface. Connecting GPT-5 and Llama is straightforward. Add a connection in admin settings, paste your endpoint and API key, and they work.

Claude is a different story entirely.

### The Claude Compatibility Problem

Here is what happens if you try the same approach with Claude: you add it via Admin Settings → Connections, paste your Azure AI Foundry endpoint and API key, and the connection test passes. The model appears in the dropdown. You select it and send a message. OpenWebUI returns "unknown model." Everything looked correct, but nothing worked.

The root cause is that OpenWebUI's Connections interface assumes all models speak the OpenAI API format. GPT-5 and Llama on Azure AI Foundry do. Claude does not. Anthropic uses a fundamentally different API structure with different endpoint paths, different authentication headers, different request and response formats. The connection test only checks reachability, not API compatibility. So it passes even though actual inference will fail.

This is a well-known limitation. The OpenWebUI community has reported it, and the official position is that non-OpenAI-compatible models are out of scope for the Connections interface.

### The Fix: Anthropic Manifold Pipe

The solution is to bypass Connections entirely and use a Pipe function instead. OpenWebUI's Pipe system lets you integrate any model provider, regardless of whether it speaks the OpenAI format. The community has built an Anthropic Manifold Pipe specifically for this.

The tricky part is not finding the Pipe. It is configuring it to work against Azure AI Foundry rather than Anthropic's direct API. The Pipe is built to talk to Anthropic's endpoints out of the box. Pointing it at Azure requires changing how it authenticates and which endpoint path it hits. Azure exposes Claude via the native Anthropic API format, but the URL structure and header requirements are different enough that the default configuration does not work. Getting this right took me a few attempts. Once configured correctly, Claude appears in your model selector and works like any other model.

This is the correct approach for any non-OpenAI-compatible model in OpenWebUI. The Connections interface is for OpenAI-format APIs only. Everything else needs a Pipe and getting the Pipe configuration right against Azure endpoints is where the real expertise sits.

---

## What I Would Do Differently

Looking back, here is what I would change if I were starting this deployment from scratch:

**Set up PostgreSQL first, before anything else.** Do not touch OpenWebUI until the database is ready. The SQLite default is a trap designed for local development, not production. Every hour I spent debugging 401 errors was an hour wasted on a problem that did not need to exist.

**Start with one replica, then scale.** Validate the full stack: authentication, database, model connections on a single replica first. Once everything works, scale to multiple replicas. Debugging a broken multi-replica deployment is significantly harder than scaling a working single-replica deployment.

**Use the same region for everything.** I had a second deployment in a different region while my primary was in Australia East. It added complexity with no benefit during initial setup. Pick one region, get it working, then consider multi-region later if your organisation requires it.

**Do not waste time trying to connect Claude via Connections.** It will never work through that interface. Go straight to the Anthropic Manifold Pipe from the start. The same lesson applies to any model provider that does not use the OpenAI API format. Check first, then decide whether you need a Pipe or a direct connection.

---

## Conclusion

Deploying OpenWebUI on Azure Container Apps is genuinely fast, the infrastructure side takes only few hours if you know what you are doing. The gap between "it runs" and "it works in production" is almost entirely about persistent storage, authentication, and model compatibility. These are not difficult problems once you understand them. But they are invisible until you hit them, and the debugging feels disproportionate to the actual fix.

The replica trap is not a bug. It is a predictable consequence of running stateful applications on ephemeral compute without shared storage. Once you understand why it happens, avoiding it is trivial. But if you hit it without knowing what to look for, it will cost you a full day.

The architecture described here is what I am running in production for a large enterprise environment. It is not over-engineered. It is not minimal either. It is the right amount of infrastructure for a self-hosted AI chat platform that needs to be reliable, secure, and maintainable along with the right amount of experience to build it correctly the first time.

If you are working on a similar deployment and want to skip the painful parts, feel free to reach out. This is exactly the kind of engagement Beyontomoro takes on.

---

*Next in this series: Building Autonomous AI Agents on Azure with MCP Servers and Custom Tools.*