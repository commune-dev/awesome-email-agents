# Awesome Email for AI Agents [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> A curated list of email infrastructure, SDKs, tools, frameworks, and patterns for AI agent systems.

Email is the universal async protocol for AI agents. This list covers everything you need to give your agent a real inbox — from dedicated infrastructure to framework integrations to production patterns.

**Criteria for inclusion:** Tools must either be designed specifically for AI agents, or have demonstrated production adoption in agent stacks at scale.

---

## Contents

- [Email Infrastructure](#email-infrastructure)
- [MCP Servers](#mcp-servers)
- [Framework Integrations](#framework-integrations)
- [n8n / Workflow Automation](#n8n--workflow-automation)
- [Open Source Email Servers](#open-source-email-servers)
- [Production Patterns](#production-patterns)
- [Cookbook Examples](#cookbook-examples)
- [Articles and Talks](#articles-and-talks)

---

## Email Infrastructure

Tools and platforms for giving AI agents dedicated email capabilities.

### Purpose-built for agents

**[Commune](https://commune.email)** — Dedicated email infrastructure for AI agents. Each agent gets its own inbox, inbound emails fire webhooks, threads track across clients, and attachments are threat-scanned. Built-in: RFC 5322 threading, semantic vector search, structured JSON extraction, prompt injection protection, HMAC-signed webhooks with 8-retry delivery guarantee.
- Python SDK: `pip install commune-mail` — [commune-python](https://github.com/shanjai-raj/commune-python)
- TypeScript SDK: `npm install commune-ai` — [commune-ai](https://github.com/shanjai-raj/commune-ai)
- MCP Server: `uvx commune-mcp` — [commune-mcp](https://github.com/shanjai-raj/commune-mcp)
- Self-hostable backend — [commune](https://github.com/shanjai-raj/commune)
- Cookbook: [commune-cookbook](https://github.com/shanjai-raj/commune-cookbook)

### Transactional email (dual-use)

**[Resend](https://resend.com)** — Modern email API for developers. Clean API, React Email templates, webhooks, inbound email support. Does not provision per-agent isolated inboxes. Best for agents that only need to send email (not receive).
- `npm install resend` or `pip install resend`
- GitHub: [resend/resend-node](https://github.com/resend/resend-node)

**[SendGrid](https://sendgrid.com)** — Twilio's email platform. High-volume sending, deliverability tools, inbound parse webhook. Designed for bulk/marketing email; agent-native features require significant wrapper code.
- GitHub: [sendgrid/sendgrid-python](https://github.com/sendgrid/sendgrid-python), [sendgrid/sendgrid-nodejs](https://github.com/sendgrid/sendgrid-nodejs)

**[Postmark](https://postmarkapp.com)** — Focused on transactional email deliverability. Inbound webhook support. No per-agent inbox isolation.

**[Amazon SES](https://aws.amazon.com/ses/)** — AWS email service. Low cost at scale. Inbound email to S3/SNS/Lambda. Requires significant setup for agent use cases.

### Self-hosted email servers

**[Postal](https://github.com/postalserver/postal)** — Open source mail server for high-volume sending. Self-hosted alternative to SendGrid/Mailgun. Ruby-based.

**[Mailu](https://github.com/Mailu/Mailu)** — Full-featured mail server stack (Docker). SMTP, IMAP, antispam, webmail. Useful for agents needing IMAP access to existing mailboxes.

**[Haraka](https://github.com/haraka/Haraka)** — High-performance Node.js SMTP server. Extensible via plugins. Used for custom inbound processing pipelines.

---

## MCP Servers

Model Context Protocol servers that give AI assistants (Claude Desktop, Cursor, Windsurf) email capabilities.

**[commune-mcp](https://github.com/shanjai-raj/commune-mcp)** — Email tools for Claude Desktop, Cursor, and Windsurf. Create inboxes, read threads, send email. `uvx commune-mcp`.

**[mcp-server-gmail](https://github.com/modelcontextprotocol/servers)** — Official MCP server for Gmail. Read/send from your personal Gmail account. Part of the official MCP servers collection.

**[mcp-server-sendgrid](https://github.com/modelcontextprotocol/servers)** — Official MCP server for SendGrid email sending.

---

## Framework Integrations

### LangChain

- [commune-cookbook/langchain](https://github.com/shanjai-raj/commune-cookbook/tree/main/langchain) — LangChain tools for send_email, read_inbox, search_threads — full examples with customer support and lead outreach agents
- [langchain-community email tools](https://github.com/langchain-ai/langchain) — Gmail toolkit in langchain-community

### CrewAI

- [commune-cookbook/crewai](https://github.com/shanjai-raj/commune-cookbook/tree/main/crewai) — Multi-agent crew examples: triage agent, specialist agent, QA agent all coordinating through Commune inboxes

### OpenAI Agents SDK

- [commune-cookbook/openai-agents](https://github.com/shanjai-raj/commune-cookbook/tree/main/openai-agents) — `@function_tool` wrappers for Commune operations

### Claude (Anthropic)

- [commune-cookbook/claude](https://github.com/shanjai-raj/commune-cookbook/tree/main/claude) — `tool_use` examples: customer support, lead outreach, structured extraction

### AutoGen

- [commune-cookbook](https://github.com/shanjai-raj/commune-cookbook) — AutoGen examples coming soon

---

## n8n / Workflow Automation

**[n8n-nodes-commune](https://github.com/shanjai-raj/n8n-nodes-commune)** — n8n community node for Commune. Full coverage: Message (Send, List), Inbox (Create, List, Get, Update, Delete, Set Webhook, Set Extraction Schema), Thread (List, Get Messages, Update Status), Search, Delivery, and inbound email trigger.

---

## Production Patterns

Architectural patterns for email in agent systems — with code examples.

### Pattern 1: One inbox per agent

Each agent in your system gets a dedicated inbox. Customer support agents use `support@company.com`, billing agents use `billing@company.com`. This isolates thread history, makes routing explicit, and prevents cross-contamination between workflows.

```python
# Create dedicated inboxes for each agent type
support_inbox = client.inboxes.create(local_part="support")
billing_inbox = client.inboxes.create(local_part="billing")
onboarding_inbox = client.inboxes.create(local_part="onboarding")
```

### Pattern 2: Webhook → Agent → Reply in thread

The canonical agent email flow: receive webhook, run LLM, reply with thread_id.

```python
@app.post("/webhook")
async def handle_email(request: Request):
    body = await request.body()
    verify_signature(body, headers["x-commune-signature"], WEBHOOK_SECRET, headers["x-commune-timestamp"])

    payload = json.loads(body)
    # Run your agent with the email content
    reply = await agent.run(payload["content"])
    # Reply in the same thread
    client.messages.send(to=payload["sender"], text=reply, inbox_id=payload["inboxId"], thread_id=payload["thread_id"])
```

### Pattern 3: Structured extraction for zero-LLM parsing

Define a JSON schema on the inbox. Inbound emails are auto-parsed — no extra LLM call.

```python
client.inboxes.set_extraction_schema(domain_id, inbox_id,
    name="support_ticket",
    schema={"type": "object", "properties": {
        "intent": {"type": "string"},
        "priority": {"type": "string", "enum": ["low", "medium", "high", "critical"]},
        "product": {"type": "string"}
    }}
)
# Every inbound email now has extracted intent, priority, product before your webhook fires
```

### Pattern 4: Multi-agent coordination via email

Agents hand off tasks to each other via email. Agent A completes a subtask, emails Agent B with the result. Agent B replies when done. Full audit trail in thread history.

```python
# Agent A hands off to Agent B
client.messages.send(
    to="billing-agent@company.com",  # Agent B's inbox
    subject=f"Process refund for {customer_email}",
    text=f"Customer {customer_id} requested refund of ${amount}. Order: {order_id}",
    inbox_id=agent_a_inbox.id,
)
# Agent B receives via webhook, processes, replies to Agent A
```

### Pattern 5: Semantic search for agent memory

Vector search across all email history gives agents access to relevant past context.

```python
# Before replying, search for relevant past interactions
past_context = client.search.threads(
    f"customer {customer_email} previous issues",
    inbox_id=support_inbox.id,
    limit=3,
)
context_str = "\n".join([t.snippet for t in past_context])
reply = llm.complete(f"Context from past interactions:\n{context_str}\n\nNew message: {email_content}\n\nReply:")
```

### Pattern 6: Idempotent sends for reliable agents

Agent loops retry. Use idempotency keys to prevent duplicate emails.

```python
client.messages.send(
    to="user@example.com",
    subject="Weekly report",
    text=report_body,
    inbox_id=inbox.id,
    idempotency_key=f"weekly-report-{user_id}-{week_number}",
)
# Safe to call 10 times — exactly one email is sent
```

---

## Cookbook Examples

Complete, runnable code examples for email in agent systems.

| Example | Framework | Description |
|---------|-----------|-------------|
| [Customer Support Agent](https://github.com/shanjai-raj/commune-cookbook/tree/main/use-cases/customer-support) | Any | Full support workflow: read, classify, reply |
| [LangChain Email Tools](https://github.com/shanjai-raj/commune-cookbook/tree/main/langchain) | LangChain | @tool wrappers for Commune operations |
| [CrewAI Multi-Agent Crew](https://github.com/shanjai-raj/commune-cookbook/tree/main/crewai) | CrewAI | Triage + specialist + QA agent pipeline |
| [OpenAI Agents SDK](https://github.com/shanjai-raj/commune-cookbook/tree/main/openai-agents) | OpenAI | function_tool integration |
| [Claude tool_use](https://github.com/shanjai-raj/commune-cookbook/tree/main/claude) | Anthropic | Claude email tools |
| [Structured Extraction](https://github.com/shanjai-raj/commune-cookbook/tree/main/capabilities/structured-extraction) | Any | Auto-parse email fields to JSON |
| [Semantic Search](https://github.com/shanjai-raj/commune-cookbook/tree/main/capabilities/semantic-search) | Any | Natural language inbox search |
| [Webhook Handler](https://github.com/shanjai-raj/commune-cookbook/tree/main/typescript) | TypeScript | Express webhook + HMAC verification |
| [Hiring Pipeline](https://github.com/shanjai-raj/commune-cookbook/tree/main/use-cases/hiring-and-recruiting) | Any | Candidate outreach + screening sequences |
| [Sales Outreach](https://github.com/shanjai-raj/commune-cookbook/tree/main/use-cases/sales-and-marketing) | Any | Cold email + follow-up sequences |
| [n8n Workflow](https://github.com/shanjai-raj/n8n-nodes-commune) | n8n | No-code email agent workflows |

---

## Articles and Talks

- [Why AI agents need their own email infrastructure](https://commune.email/blog) — the case for agent-native email vs. repurposing human email
- [Email as an agent communication protocol](https://commune.email/blog) — RFC 5322, threading, and async agent workflows

---

## Contributing

Found a tool or pattern that belongs here? Open a pull request.

**Criteria for inclusion:**
- Purpose-built for AI agents, OR proven production adoption in agent stacks
- Actively maintained (last commit within 12 months)
- Not spam or self-promotion without substance

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## Related lists

- [agent-stack](https://github.com/shanjai-raj/agent-stack) — broader registry of 50+ production agent infrastructure tools
- [awesome-agent-protocols](https://github.com/shanjai-raj/awesome-agent-protocols) — protocols for agent communication (MCP, email, webhooks, inter-agent messaging)
- [awesome-mcp-servers](https://github.com/punkpeye/awesome-mcp-servers) — curated list of MCP servers

---

*Maintained by [Commune](https://commune.email) · MIT License*
