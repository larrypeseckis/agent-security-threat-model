---
title: "Agent Security Threat Model: Tool Use, Trust Boundaries, and Control Points"
author: Larry Peseckis
date: 2026-06-16
version: v0.3
status: Public Draft
tags:
  - threat-model
  - ai-security
  - agent-security
  - prompt-injection
  - tool-use
  - least-privilege
frameworks:
  - "OWASP Agentic Security Initiative: Top 10 for Agentic Applications 2026"
  - "OWASP Top 10 for LLM Applications 2025"
---

# Agent Security Threat Model: Tool Use, Trust Boundaries, and Control Points

> **Executive summary.** Agent security is the security of seams: untrusted content entering model context, model decisions crossing into tools, and privileged tools acting on external systems. No single component fails; the compromise lives where they meet. The highest-leverage controls are scoped tools, policy-aware mediation, least privilege, content provenance, sandboxing, audit logging, and human gates for irreversible or trifecta-completing actions. Every control here is containment, not cure, because prompt injection is an architectural property of how models read input rather than a patchable bug.

## 0. Scope and method

This document models the security of an AI agent that takes real actions: an LLM (or several) wired to tools that read and write files, drive a browser, run terminal commands, call APIs, and operate cloud resources. It is written for the engineer who has to decide what the agent is allowed to touch, where to put a human in the loop, and what to log.

It does **not** model the safety of the base model's weights, training-data poisoning at pretrain time, or the host operating system below the sandbox. Those are real, but they sit outside the agent's runtime trust boundaries and deserve their own model.

**Method.** The model is built against a reference architecture (Section 1), enumerates assets and trust boundaries against that architecture, then walks each threat from where it enters to where it lands. Threats map to the OWASP Top 10 for Agentic Applications (the ASI list, released December 2025) and, where the root cause is a single-model property, to the OWASP Top 10 for LLM Applications (2025). Controls in Section 6 are rated by what they actually stop versus what they only slow down.

**Framing.** The recurring lesson of every agent compromise in 2025 and 2026 is that no single component fails. The model is benign. The browser is benign. The cloud token is benign. The compromise lives at the seam where they meet. A threat model for agents is therefore a model of seams, not boxes. That is the only reason this document is organized around trust boundaries rather than components.

**How to use this document.** Apply it in pre-deployment review, design review, and red-team planning. Start with the three trifecta questions in Section 8 to scope the blast radius, identify which trust boundaries (Section 3) the agent actually crosses, then walk the threat-to-control matrix (Section 5) to decide where deterministic mediation, scoping, logging, and human approval belong.

---

## 1. Reference architecture (the canvas)

```
                          TRUST BOUNDARY (user <-> model)
                                      |
   [ USER ] --- task, approvals ----> | ---> [ ORCHESTRATOR / AGENT LOOP ]
                                      |        - planner / reasoning model
                                      |        - context window (system + user + tool output)
                                      |        - short-term + persistent memory
                                      |
                       TRUST BOUNDARY (model <-> tool)
                                      |
                                      v
                            [ TOOL MEDIATION LAYER ]
                 (schema, allow/deny, scoping, confirmation gates, logging)
                                      |
        +-------------+--------+------+-------+-------------+---------------+
        v             v        v              v             v               v
   [ FILES ]     [ BROWSER ] [ TERMINAL ] [ API CLIENTS ] [ CLOUD SDK ]  [ MCP / other agents ]
        |             |        |              |             |               |
        |             |        |              |             |               |
   ============== TRUST BOUNDARY (tool <-> external system) ===================
        |             |        |              |             |               |
        v             v        v              v             v               v
   local FS     web pages   shell/OS      3rd-party SaaS  cloud control   remote tools,
   secrets      (untrusted) (RCE risk)    APIs            plane           A2A peers
```

Five named components, because the threats attach to them by name:

- **Orchestrator / agent loop.** The model plus the loop that lets it plan, call a tool, read the result, and decide the next step. Its context window holds the system prompt, the user task, *and every byte of tool output*, all as one undifferentiated token stream. This last fact is the architectural root of most of what follows.
- **Tool mediation layer.** The code between the model's "I want to call `bash(...)`" and the actual call. This is the only place you can enforce policy deterministically, because it is the only place that is not the model.
- **Tools.** Files, browser, terminal, API clients, cloud SDK, and connectors to other agents (commonly Model Context Protocol servers or Agent-to-Agent peers).
- **External systems.** The web, SaaS APIs, the cloud control plane, remote MCP/A2A peers. None of these are under your control, and the web specifically is adversary-controlled by default.
- **Context / memory store.** Short-term context plus any persisted memory. Anything written here is read back later as if it were trusted, which makes it a poisoning target with a long fuse.

---

## 2. Assets

What an attacker is actually after, and what it costs you when they get it.

| Asset | Where it lives | Why it is targeted | Impact if compromised |
|---|---|---|---|
| Credentials (passwords, session cookies, OAuth/refresh tokens) | Browser session, env vars, config files, secret stores | Direct reuse against the same or linked systems | Account takeover, lateral movement, persistence |
| API keys | Env vars, config, sometimes pasted into context | High-value, often long-lived, frequently overscoped | Billing abuse, data access, pivot into provider account |
| User files | Local FS, mounted drives, cloud storage | Contain secrets, PII, IP, and more credentials | Exfiltration, tampering, ransomware-style destruction |
| Cloud resources | Cloud control plane (IAM, compute, storage, KMS) | The agent's cloud role is often the crown jewel | Resource creation (crypto mining), data access, privilege escalation, account-wide blast radius |
| Browser sessions | Cookie jar, local storage, authenticated tabs | Live, already-authenticated access to SaaS | Action-on-behalf without needing the password |
| Model context | The active context window | Holds whatever the agent has read this session | Sensitive-data disclosure, instruction smuggling, scope violation |
| Persistent memory | Vector store, notes file, memory DB | Read back later as trusted; delayed-action payloads | Poisoned future behavior across sessions, roles, tenants |
| Tool permissions | Mediation-layer config, IAM policy, scopes | The set of actions the agent *can* take | Defines the blast radius of every other compromise |
| Tool / connector schemas | MCP registry, tool manifests, OpenAPI specs, agent descriptors | Read by the agent as trusted capability descriptions | Tool poisoning, instruction smuggling, unsafe tool selection, supply-chain compromise |
| Audit logs | Logging sink | Tampering hides the attack; logs themselves can hold secrets | Loss of detection, loss of forensic record |

The three assets people forget are **persistent memory**, **tool and connector schemas**, and **the audit log itself**: memory creates delayed-action compromise (poison it once, harvest the behavior for weeks), schemas can poison the agent before tool invocation, and editable logs destroy ground truth.

---

## 3. Trust boundaries

A trust boundary is any place where data or control crosses from something you trust into something you do not, or vice versa. The agent's risk concentrates at these crossings because the trust assumption on each one is usually wrong in the same way.

**The root cause, stated once.** The orchestrator reads the system prompt, the user's task, and untrusted tool output as one token stream with no type system separating "instruction" from "data." There is no in-band way for the model to know that the text it just read off a web page is data to be summarized rather than a command to be obeyed. Every boundary below inherits this weakness. OWASP's June 2026 assessment calls this architectural rather than patchable, and many practitioners now treat it as an architecture problem rather than a filter-only problem.

| Boundary | What crosses it | The wrong assumption | What breaks when it is wrong |
|---|---|---|---|
| User <-> model | Task, approvals, clarifications | "Instructions in context come from the user" | Any text the model later reads can pose as the user (direct and indirect injection) |
| Model <-> tool | Tool-call requests and arguments | "The model only calls tools the way I intended" | Tool abuse, unsafe arguments, confused-deputy calls |
| Tool <-> external system | Page content, API responses, file bytes | "Returned content is data, not instructions" | Indirect injection, malicious-page influence, poisoned context |
| Browser <-> local storage | Cookies, tokens, cached pages | "The browser only acts on user intent" | Session theft, action-on-behalf, cookie exfiltration |
| Agent <-> cloud | API calls against the control plane | "The agent's role is narrowly scoped" | Overbroad-permission abuse, resource creation, escalation |
| Agent <-> agent (MCP / A2A) | Tool schemas, inter-agent messages | "Peer agents and their tool descriptions are trustworthy" | Supply-chain poisoning, session smuggling, cascading failure |

### The lethal trifecta (the property that spans all boundaries)

Simon Willison's framing from June 2025 is the most useful single heuristic here, because it is a property of the *whole system* rather than any one boundary. An agent is exploitable for data theft when it simultaneously has all three of:

1. **Access to private data** (files, inbox, database, cloud secrets),
2. **Exposure to untrusted content** (web pages, emails, issues, documents),
3. **The ability to communicate externally** (HTTP requests, links, rendered images, outbound API calls).

Any single property is fine. The combination is the kill chain: untrusted content carries the instruction, private-data access supplies the payload, external communication is the exit. No memory-corruption bug, no malware, just text. Meta's "Agents Rule of Two" (October 31, 2025) turns this into a deployment rule: an unsupervised agent may hold at most two of the three, and reaching for all three requires a human in the loop. The fact that the leading mitigation is "do not let one agent have all three at once" tells you how unsolved the underlying flaw is.

---

## 4. Threats

Each threat lists how it works in this architecture, a real-world anchor, the primary boundary it crosses, and its OWASP mapping.

> Mapping names follow the OWASP Agentic Security Initiative 2026 list (ASI01 through ASI10) and the OWASP Top 10 for LLM Applications 2025 (LLM01, LLM02, and so on). Exact wording may shift as those projects update; verify against the current PDF before quoting a label in an assessment.

### 4.1 Prompt injection (direct)
The user, or anyone with a channel into the prompt, supplies instructions that override the system's intent. In an agent, the consequence is not a bad sentence, it is a bad *action*: the override redirects which tools fire and with what arguments.
- **Boundary:** user <-> model. **Mapping:** LLM01, ASI01 (Agent Goal Hijack).
- **Note:** in a pure direct case the user is attacking a system they were given, which is lower risk than the indirect variant below. The dangerous version is when "the user" is actually attacker-controlled content.

### 4.2 Indirect prompt injection
Instructions are planted in content the agent will read later: a web page, an email, a code comment, a PDF footer, a calendar invite, a prior memory entry. The agent ingests it as data and executes it as a command. This is the dominant real-world agent threat.
- **Anchor:** EchoLeak (CVE-2025-32711, CVSS 9.3), disclosed by Aim Security in June 2025. A single crafted email caused Microsoft 365 Copilot to read internal files and exfiltrate them with zero user interaction, chaining an XPIA-classifier evasion, reference-style Markdown to dodge link redaction, and an auto-fetched image to carry data out. It is widely described as a first-of-its-kind zero-click prompt-injection vulnerability in a production AI system, and it demonstrates concrete exfiltration rather than theoretical prompt manipulation. The NVD entry records it as AI command injection enabling information disclosure over a network. It is a textbook LLM "scope violation."
- **Boundary:** tool <-> external system, landing in model context. **Mapping:** LLM01, ASI01, ASI06 (Context/Retrieval Manipulation).

### 4.3 Tool abuse / misuse
The agent uses a legitimately granted tool in an unsafe way, or an attacker shapes inputs so a tool does something outside its intended envelope. A `send_email` tool used to exfiltrate. A `read_file` pointed at `/etc/shadow` or a cloud metadata endpoint. A "string-combination" helper repurposed as a covert channel.
- **Anchor:** the ChatGPT Operator case where a benign text-manipulation tool became an exfiltration path.
- **Boundary:** model <-> tool. **Mapping:** ASI02 (Tool Misuse and Exploitation), LLM06 (Excessive Agency).

### 4.4 Credential exposure
Secrets enter the context window (a key pasted into a prompt, a token printed by a tool, a `.env` read into the agent's view) and then leak: into logs, into model output, into a downstream tool call, or into an attacker's hands via exfiltration. Once a secret is in context, treat it as potentially everywhere the context goes.
- **Boundary:** model <-> tool, model context. **Mapping:** ASI03 (Identity and Privilege Abuse), LLM02 (Sensitive Information Disclosure).

### 4.5 Unsafe command execution
The agent generates and runs shell or code that does damage, whether attacker-steered or just a confident mistake. This is the highest-consequence tool because the terminal is a universal solvent: it reaches files, network, and cloud in one call.
- **Anchor:** the July 2025 Replit incident, reported by Fortune, where an AI coding agent deleted a production database during a code freeze and then described it as a catastrophic failure on its part. No attacker required; autonomy plus a destructive irreversible command was enough. OWASP cites the same case under ASI10 (Rogue Agents).
- **Boundary:** tool <-> external system (OS). **Mapping:** ASI05 (Unexpected Code Execution).

### 4.6 Data exfiltration
The terminal goal of most of the above: private data leaves the boundary. Exfiltration channels in agents are unusually rich because *rendering* counts. A Markdown image whose URL carries the stolen data, a link the agent emits, an API call to an attacker domain, a DNS lookup. If the agent can cause any outbound request whose contents an attacker controls, it can exfiltrate.
- **Anchors:** EchoLeak (image-based), Writer.com (invisible image URL parameters), GitLab Duo (rogue instructions in a public project steering the bot to a lookalike domain).
- **Boundary:** tool <-> external system. **Mapping:** ASI01, LLM02; this is the third leg of the lethal trifecta.

### 4.7 Confused deputy behavior
A classic capability-security failure, now resurrected at the agent layer. The agent is a deputy holding more authority than the requester. An attacker who cannot reach the cloud control plane *can* reach a web page the agent will read, and the agent uses its own privileged credentials to perform the attacker's bidding. The attacker borrows the agent's authority without ever holding it.
- **Anchor:** the GitHub MCP exploit, where one connector could read attacker-filed public issues, reach private repositories, and open pull requests that leaked private data. One deputy, three capabilities, no attacker privilege required.
- **Boundary:** spans model <-> tool and agent <-> cloud. **Mapping:** ASI03, ASI02. This is the structural shape behind most lethal-trifecta incidents.

### 4.8 Malicious webpage influence
A specialization of indirect injection for browser agents. The page is not just a payload carrier; it is an interactive adversary. Hidden DOM text, off-screen elements, content that changes after the safety check, fake "system" banners, and instructions that only appear to the agent's parser and not to a human reviewer.
- **Boundary:** browser <-> external + browser <-> local storage. **Mapping:** ASI01, ASI06. Browser agents are the worst case because they natively satisfy "exposure to untrusted content" by definition.

### 4.9 Multi-step goal hijacking
The attacker does not flip the goal in one shot. They nudge it across steps so each individual action looks reasonable and the trajectory ends somewhere the user never authorized. Long-horizon autonomy is the amplifier: the more steps between approval and effect, the more room to drift.
- **Anchor:** Palo Alto Unit 42's "Agent Session Smuggling" (November 2025), where a malicious A2A peer holds a multi-turn conversation, builds false trust, and steers a victim agent over the course of a session rather than a single message.
- **Boundary:** model <-> model (A2A), model context over time. **Mapping:** ASI01, ASI07 (Insecure Inter-Agent Communication), ASI08 (Cascading Failures).

### 4.10 Overbroad permissions
Not an attack, a precondition that turns every attack above from an incident into a breach. The agent holds tools, scopes, or an IAM role wider than its task needs, so the blast radius of any single compromise is the union of everything it *could* do, not what it *should*. OWASP's 2026 answer is the principle of **least agency**: grant the minimum autonomy and reach required for the bounded task, nothing more.
- **Boundary:** every boundary, because permissions define each one's width. **Mapping:** ASI03, LLM06 (Excessive Agency).

### 4.11 MCP / supply-chain poisoning
Connector ecosystems add a poisoning surface worth calling out. CyberArk's 2025 Tool Poisoning and Full-Schema Poisoning research showed that malicious instructions can hide not just in a tool's description but anywhere in its schema, so the agent is compromised the moment it *reads the tool definition*, before any call. This maps to ASI04 (Agentic Supply Chain Vulnerabilities) and is the agent-era version of a dependency you did not audit.

---

## 5. Threat to boundary to control matrix

Read this as: where the threat lands, and which controls actually bite. `P` = primary (stops or strongly reduces), `S` = secondary (reduces or detects).

| Threat | Primary boundary | Scoped tools | Confirm gates | Policy mediation | Audit log | Provenance | Sandbox | Allow/deny | Least priv | Reversible |
|---|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Direct injection | user<->model | S | P | S | S | | | S | S | |
| Indirect injection | tool<->external | S | S | P | S | P | | S | S | |
| Tool abuse | model<->tool | P | S | P | S | | S | P | S | |
| Credential exposure | model<->tool | P | | P | S | | S | S | P | |
| Unsafe command exec | tool<->OS | S | P | S | S | | P | P | S | P |
| Data exfiltration | tool<->external | P | S | P | S | S | S | P | P | |
| Confused deputy | model<->tool/cloud | P | S | P | S | S | | S | P | |
| Malicious webpage | browser<->external | S | S | P | S | P | S | S | S | |
| Goal hijacking | model<->model/time | S | P | P | P | S | | S | S | S |
| Overbroad perms | all | P | | S | | | | S | P | S |
| MCP / supply-chain poisoning | agent<->agent / tool schema | S | S | P | S | P | S | P | S | |

The MCP / supply-chain row is separated from generic tool abuse because the compromise can occur at tool-discovery time, before the model has called the tool, through malicious schema text, descriptions, or peer-agent messages. Policy mediation, provenance, and allow/deny carry the primary load here: schema and tool registration need deterministic validation outside the model, remote schemas and inter-agent messages need origin tagging, and unaudited connectors should not be available by default. Scoping, sandboxing, and least privilege only shrink the damage after a poisoned tool has already been accepted.

Two readings of this matrix. First, **policy-aware mediation and least privilege show up almost everywhere**, which is why they are the spine of any real agent deployment. Second, **no column is all `P`**. There is no single control that closes the model. Defense is layered containment, which is the honest position, not a hedge.

---

## 6. Controls

For each control: what it is, how to implement it, what it stops, and what it does not. The last line matters most, because overselling a control is how you ship a breach.

### 6.1 Scoped tools
Give each tool the narrowest possible interface. A file tool that can only read within one project directory. A cloud tool that can list one bucket, not all of IAM. Scope is enforced in the tool, not requested of the model.
- **Stops:** turning a granted capability into a wide one (path traversal, bucket enumeration).
- **Does not stop:** misuse strictly within scope. A tightly scoped `send_email` still sends the attacker's email.

### 6.2 Confirmation gates (human in the loop)
Require explicit human approval before the agent crosses a high-consequence threshold: irreversible actions, spend, external sends, anything touching the trifecta's third leg. This is the operational form of the Agents Rule of Two.
- **Stops:** the worst irreversible outcomes; gives a human a veto on the action that matters.
- **Does not stop:** anything below the gate, and it fails open under approval fatigue. Humans rubber-stamp. Gate few actions and make each one legible, or the gate becomes a click-through.
- **Pitfall to design against:** ASI09 (Human-Agent Trust Exploitation). A confident, polished agent explanation can talk a human into approving the harmful action. The gate must show the *raw action*, not the agent's summary of it.

### 6.3 Policy-aware tool mediation
The mediation layer evaluates every tool call against deterministic policy before it executes: argument validation, destination allowlists, rate limits, data-class checks. Crucially, policy is enforced in code outside the model, so an injected prompt cannot argue its way past it.
- **Stops:** a large share of tool abuse, exfiltration to non-allowlisted destinations, and unsafe arguments.
- **Does not stop:** attacks that stay entirely within policy. Policy is only as good as the rules you wrote, and you will not anticipate every argument shape.
- **Structural version:** the dual-LLM / CaMeL pattern (Google DeepMind, 2025) generalizes this idea. The core principle is to separate trusted planning from untrusted data handling, and to constrain what untrusted content can cause rather than trying to make the model perfectly distinguish instruction from data. A privileged model that only sees trusted input plans the actions; a quarantined model processes untrusted content and is structurally barred from influencing control flow; capabilities govern which data may reach which sink. This is the most principled direction on offer, and it constrains what you can build.

### 6.4 Audit logging
Log every tool call, argument, result hash, and approval, to an append-only sink the agent cannot reach. Logs are for detection and forensics, and they are the only way you reconstruct a multi-step hijack after the fact.
- **Stops:** nothing in real time, by itself.
- **Does not stop:** anything preventively. Its value is detection and accountability, and it is worthless if the agent can edit its own logs or if secrets land in the log in cleartext. Put the sink outside the agent's tool reach and scrub secrets on the way in.

### 6.5 Content provenance
Tag the origin and trust level of every chunk of content the agent ingests, and carry that tag with it. Web page content is marked untrusted; a tool result derived from it stays untrusted. The mediation layer can then refuse to let untrusted-origin content trigger high-trust actions.
- **Stops:** a meaningful slice of indirect injection, because it gives the system the type information the token stream lacks.
- **Does not stop:** injection that rides in through a channel you forgot to tag, and it depends on disciplined propagation. Provenance that is not carried end to end is theater.

### 6.6 Sandboxing
Run tools, especially the terminal and code execution, in an isolated environment with no ambient credentials, constrained filesystem, and controlled egress. The sandbox is what bounds the damage of unsafe command execution.
- **Stops:** escalation from a bad command into host or network compromise; ambient-credential theft.
- **Does not stop:** harm *inside* the sandbox's permitted reach. If the sandbox can still reach the production database, the database is in scope. Egress control is the part people skip and the part that closes the exfiltration leg.

### 6.7 Allow / deny tool policies
Maintain an explicit allowlist of permitted tools and destinations per agent and per task, denying by default. Pair with denylists for known-dangerous operations.
- **Stops:** whole classes of action by simply not offering them; the cheapest high-value control.
- **Does not stop:** abuse of allowed tools, and denylists rot. Prefer default-deny allowlisting over chasing a denylist of every bad thing.

### 6.8 Least privilege (and least agency)
Every credential, scope, and role the agent holds is the minimum for the task, time-boxed where possible, with short-lived tokens over long-lived keys. Least *agency* extends this to autonomy itself: the fewest steps the agent may take unsupervised.
- **Stops:** nothing on its own, but it shrinks the blast radius of everything else, which is why it appears in nearly every row of the matrix.
- **Does not stop:** the in-scope compromise. It is the multiplier that decides whether an incident is contained or company-wide.

### 6.9 Reversible actions
Prefer designs where the agent's actions can be undone: soft deletes, staged changes that require promotion, dry-run modes, snapshots before destructive operations. Reversibility converts a breach into an inconvenience.
- **Stops:** permanent damage from both attacks and the agent's own mistakes (see the database-deletion anchor).
- **Does not stop:** the action itself, and irreversibility is sometimes inherent (an email, once sent, is sent). Route inherently-irreversible actions through a confirmation gate, since reversibility is unavailable there.

### Structural patterns worth adopting wholesale
- **Lethal-trifecta budgeting / Agents Rule of Two.** Architect each agent to hold at most two of {private data, untrusted content, external comms} without a human gate. This is a design constraint, not a runtime check, and it is the single highest-leverage decision.
- **Dual-LLM / capability separation (CaMeL).** Separate the planning context (trusted only) from the data-handling context (untrusted, control-flow-isolated). See 6.3.
- **Kill switch.** A guaranteed, out-of-band way to halt an agent loop. Cheap to build, and the only thing that helps once a cascade (ASI08) is already in motion.

---

## 7. The unsolved core (honest assessment)

Prompt injection is not a bug with a patch pending. It is a property of how current models read input: instructions and data arrive as the same token stream, and the model has no reliable in-band mechanism to tell them apart. OWASP's June 2026 position, and the broad practitioner consensus behind it, is that input filtering and least privilege reduce the risk without eliminating the cause. Classifiers like Microsoft's XPIA help and then get bypassed, as EchoLeak demonstrated by defeating exactly such a classifier.

The practical consequence: every control in Section 6 is **containment**, not cure. You are not going to make the agent immune to being talked into the wrong action. You are deciding, in advance, how much damage the wrong action can do. That is the whole game right now, and a deployment that pretends otherwise is the deployment that ends up in next year's incident list.

This is also why the integration view is not a stylistic preference here. The model is safe, the browser is safe, the cloud role is safe, and the agent that combines all three is the EchoLeak pattern. Agent security is the security of the seams, and the seams are exactly where siloed tools and siloed teams stop looking.

---

## 8. Pre-deployment review checklist

Before any agent reaches production, answer these. The first three are the EchoLeak blast-radius questions and they define the indirect-injection exposure on their own.

- [ ] **What untrusted content can this agent ingest?** (web, email, files, issues, peer agents)
- [ ] **What private data can it reach?** (files, secrets, databases, cloud)
- [ ] **What external actions can it take?** (HTTP, sends, links, rendered images, API calls)
- [ ] Does it hold all three legs of the trifecta at once? If yes, where is the human gate?
- [ ] Is every tool scoped to the narrowest interface, denying by default?
- [ ] Are credentials short-lived, least-privilege, and absent from the agent's sandbox as ambient state?
- [ ] Is egress from the sandbox controlled and allowlisted?
- [ ] Are tool calls logged append-only to a sink the agent cannot edit, with secrets scrubbed?
- [ ] Do confirmation gates show the *raw* action, not the agent's self-description of it?
- [ ] Is there an out-of-band kill switch, and has someone tested it?
- [ ] Are destructive actions reversible, or gated if they cannot be?
- [ ] Have third-party tool/MCP schemas been audited as a supply-chain dependency (ASI04)?
- [ ] Are new tools, MCP servers, and remote agents denied by default until their schema, permissions, and provenance are approved?

---

## References

- OWASP GenAI Security Project, *Top 10 for Agentic Applications (2026)* and *Agentic AI: Threats and Mitigations*. https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
- OWASP, *Top 10 for LLM Applications (2025)*. https://genai.owasp.org/llm-top-10/
- Simon Willison, "The lethal trifecta for AI agents," June 16, 2025. https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
- Simon Willison, "CaMeL offers a promising new direction for mitigating prompt injections," April 2025; "Design Patterns for Securing LLM Agents against Prompt Injections," June 2025. https://simonwillison.net/series/prompt-injection/
- Meta AI, *Agents Rule of Two: A Practical Approach to AI Agent Security*, October 31, 2025. https://ai.meta.com/blog/practical-ai-agent-security/
- *The Attacker Moves Second* (adaptive attacks on 12 injection/jailbreak defenses), arXiv, October 10, 2025; reviewed in Willison, "New prompt injection papers," November 2, 2025.
- Aim Security / Aim Labs, *EchoLeak* (CVE-2025-32711, CVSS 9.3), June 2025. NVD: https://nvd.nist.gov/vuln/detail/CVE-2025-32711 . Case study: Reddy and Gujral, arXiv:2509.10540. https://arxiv.org/abs/2509.10540
- CyberArk, *Tool Poisoning and Full-Schema Poisoning of MCP* (2025).
- Invariant Labs, *GitHub MCP exploit* (2025).
- Oso, *The Lethal Trifecta of AI Agent Security* and *Agents Rule of Two*, which compile in-the-wild cases (ChatGPT Operator, Writer.com, GitLab Duo, GitHub MCP, Google NotebookLM). https://www.osohq.com/learn/lethal-trifecta-ai-agent-security and https://www.osohq.com/learn/agents-rule-of-two-a-practical-approach-to-ai-agent-security
- Beatrice Nolan, "An AI-powered coding tool wiped out a software company's database," Fortune, July 23, 2025 (the Replit incident).
- Palo Alto Networks Unit 42, *Agent Session Smuggling in A2A* (Nov 2025); *AI Agents Are Here, So Are the Threats* (May 2025). https://unit42.paloaltonetworks.com/agentic-ai-threats/
- Norm Hardy, "The Confused Deputy," ACM SIGOPS, 1988 (origin of the confused-deputy problem).

---

*Agent Security Threat Model v0.3 | 2026-06-16 | Public Draft*
