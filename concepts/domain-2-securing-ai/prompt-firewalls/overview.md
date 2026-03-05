# Prompt Firewalls

**Exam relevance:** Domain 2.0 — Securing AI Systems | Objectives 2.2 (Gateway Control), 2.6 (Compensating Control)

📺 [Watch the TikTok: Day 1](https://www.tiktok.com/@certgames.com/video/7613739701656145183) | ← [Back to Domain 2](../)

---

## What Is a Prompt Firewall?

A prompt firewall is a security layer that sits **between users and an AI model**, inspecting the natural-language content of prompts and responses for malicious intent, policy violations, and sensitive data — before they reach the model or the user.

Think of it like a bouncer at the door: checking IDs before anyone gets in, and checking bags on the way out.

**The key difference from a traditional WAF (Web Application Firewall):**

| Dimension | Traditional WAF | Prompt Firewall |
|-----------|----------------|----------------|
| Operating layer | Network (L3-L4) or HTTP (L7) | Semantic / language layer |
| Input type | Structured syntax (SQL, HTTP) | Unstructured natural language |
| Detection approach | Deterministic signature matching | Probabilistic ML classification |
| Control/data separation | Clear (parameterized queries) | None — instructions and data are both text |
| Attack surface | Requires technical skill | Anyone who can write can attack |

**The fundamental challenge:** In a SQL database, there's a clear boundary between code and data. Parameterized queries enforce that boundary — the definitive fix for SQL injection. No equivalent exists for LLMs. System instructions, user input, tool outputs, and retrieved documents all occupy the same context window as equal tokens.

---

## Where Prompt Firewalls Sit in Architecture

### The Standard Pipeline

```
User → API Gateway/Rate Limiting 
     → Input Firewall (PII redaction, injection detection, topic filtering)
     → System Prompt + Sanitized Input
     → LLM (with RLHF alignment)
     → Output Firewall (toxicity, data leak, code analysis)
     → Logging/Monitoring
     → User
```

### For Agentic Systems (AI with Tool Access)

```
User → Input Firewall 
     → Agent LLM 
     → Tool-Call Firewall (strips unnecessary data before tool executes)
     → Tool Execution 
     → Tool-Output Firewall (sanitizes output before re-entering agent context)
     → Agent LLM 
     → Output Firewall 
     → User
```

This matters for the exam: **indirect prompt injection** enters through tool outputs and retrieved documents, not through user prompts. Standard input firewalls miss it entirely.

---

## How Prompt Firewalls Work Technically

### Core Detection Mechanisms

**Input inspection (pre-model):** Validates user prompts before they reach the LLM.
- Regex-based PII detection (credit cards, SSNs, API keys)
- Keyword blocklists for known injection phrases
- ML classifier scoring (safe/unsafe probability)
- Secondary LLM analysis using models like Llama Guard

**Output inspection (post-model):** Evaluates generated responses before delivery.
- Data leak detection
- Toxic content filtering
- Code static analysis (e.g., LlamaFirewall's CodeShield: ~60ms light scan, ~300ms full scan, 90% resolved at light tier)
- Hallucination checks

**Classifier models:** Fine-tuned transformer models trained on known jailbreak attempts and benign prompts.
- Meta PromptGuard 2 (86M parameters, mDeBERTa architecture)
- Protect AI DeBERTa-v3 (F1: 0.9998 on evaluation sets)
- Lakera Guard (updated ~100,000 times daily from their Gandalf challenge game)

**Semantic similarity:** Embedding-based comparison against a vector database of known attacks. Incoming prompts are compared to stored attack embeddings via cosine similarity.

---

## Five Deployment Models

| Model | How It Works | Example | Tradeoff |
|-------|-------------|---------|---------|
| **Reverse proxy** | Transparent intermediary, zero code changes | Cloudflare Firewall for AI | Adds network hops |
| **API wrapper** | Single API call returns flagged/safe | Lakera Guard | Requires integration work |
| **Middleware/SDK** | Python library, developer configures | NVIDIA NeMo Guardrails | Most control, most complexity |
| **Edge network** | CDN-level, globally distributed | Akamai Firewall for AI | Lowest latency |
| **Sidecar container** | Kubernetes pod sharing with AI service | Dropbox/Lakera deployment | Works in cloud-native stacks |

---

## What Prompt Firewalls Stop

**Reliably detected:**
- Direct prompt injection (known patterns like "ignore all previous instructions")
- Known jailbreak templates (DAN, "grandma" exploit, character roleplay)
- PII in inputs and outputs (SSNs, credit cards, API keys, emails)
- Toxic and harmful output content
- System prompt extraction attempts
- Denial-of-wallet attacks (excessive token consumption)

**The catch:** Lakera's Gandalf challenge generates ~100,000 new attack data points daily from 1M+ players. The threat landscape evolves faster than any static blocklist.

---

## What Prompt Firewalls Cannot Stop

**Indirect prompt injection:** Malicious instructions embedded in external data — webpages, PDFs, emails, RAG documents. Input firewalls only see the user query, not the retrieved documents. A 2023 GitHub Copilot attack embedded instructions in source code markdown to exfiltrate conversation data via an image URL.

**Obfuscation techniques (Mindgard 2025 research — 100% evasion achieved):**
- Unicode homoglyphs (Cyrillic 'а' instead of Latin 'a')
- Zero-width characters inserted between letters
- Emoji Unicode tag smuggling — instructions embedded inside emoji tags, invisible to humans
- Payload splitting across multiple messages

**Multi-turn escalation (Crescendo):** Each individual turn looks clean. The threat only emerges across the full conversation — which most firewalls don't analyze holistically.

**GCG adversarial suffixes:** Gradient-optimized tokens that force affirmative responses. Achieved 88% attack success rate on black-box LLMs including GPT-4, and transfer from open-source to proprietary models.

---

## The False Positive Problem

Even highly accurate guards compound errors. If each of 5 guards has 90% accuracy, the probability that at least one incorrectly flags a benign input is:

**1 − 0.9⁵ ≈ 41%**

Meta experienced this firsthand: PromptGuard 1 attempted broad detection but generated excessive false positives. PromptGuard 2 deliberately narrowed its scope to reduce them. There is no free improvement — tightening security increases false positives; loosening it increases attack success.

---

## Latency Reality

| Approach | Added Latency |
|----------|--------------|
| Lightweight BERT classifier | ~10–50ms |
| Optimized modern system | 50–200ms |
| LLM-as-judge approach | 5–10× slower than classifier |
| Chain-of-thought reasoning | Up to 8.6 seconds |
| Comprehensive enterprise suite | 1–5 seconds |

Cloudflare mitigates this with **asynchronous parallel architecture** — detection modules run simultaneously, bounded by the slowest module, not their sum.

---

## Major Production Tools

| Tool | Type | Key Strength |
|------|------|-------------|
| **Cloudflare Firewall for AI** | Edge proxy | Zero code changes, model-agnostic |
| **Azure AI Prompt Shield** | API | Direct + indirect injection detection |
| **AWS Bedrock Guardrails** | Managed service | Formal logic verification (unique) |
| **Meta LlamaFirewall** | Open source | Composable: PromptGuard + AlignmentCheck + CodeShield |
| **NVIDIA NeMo Guardrails** | Open source framework | Programmable via Colang DSL, 5 rail types |
| **Protect AI LLM Guard** | Open source | 15 input + 20 output scanners, 2.5M+ downloads |
| **Lakera Guard** | API | Continuously updated threat intelligence |

---

## How It Appears on the SecAI+ Exam

Prompt firewalls appear **twice** in the official objectives, both in Domain 2.0 (40% of exam):

**Objective 2.2 — Gateway Control:**
Listed under "Gateway Controls" alongside rate limits, token limits, input quotas, modality limits, and endpoint access controls. Distinct from "Model Controls" (model guardrails, prompt templates). The exam tests that you understand **prompt firewalls are gateway-level** (external to the model).

**Objective 2.6 — Compensating Control:**
Listed **first** among compensating controls for attack scenarios. The exam will present an attack scenario and ask which control to recommend.

### Attack-to-Control Mapping (Memorize This)

| Attack | Primary Controls |
|--------|-----------------|
| Prompt injection / jailbreaking / input manipulation | **Prompt firewalls** |
| Sensitive information disclosure | Prompt firewalls + data integrity controls |
| Model DoS / unbounded token consumption | Prompt firewalls + rate limiting + token limits |
| Circumventing AI guardrails | Prompt firewalls (as external second layer) |
| Data poisoning / model theft | Access controls, encryption (not primarily firewalls) |

---

## The Critical Distinction: Firewalls vs. Guardrails

CompTIA explicitly separates these in Objective 2.2:

| | Prompt Firewalls | Model Guardrails |
|-|-----------------|-----------------|
| **Location** | Gateway (external to model) | Model-level (internal) |
| **Control type** | Gateway control | Model control |
| **Scope** | Input/output filtering | Training + inference behavior |
| **Updateable without retraining?** | Yes | No (training-time) / Yes (inference-time guardrail systems) |
| **Analogy** | Bouncer at the door | Rules baked into the employee's training |

**Exam tip:** If a question asks about controls that sit **between users and the model**, that's a prompt firewall. If it asks about controls **within the model's behavior**, that's guardrails.

---

## Quick Exam Summary

- Prompt firewalls = gateway-level security layer between users and AI models
- Inspect inputs AND outputs (bidirectional)
- Gateway control (Obj. 2.2) AND compensating control (Obj. 2.6)
- Stop: known injections, PII, toxic output, denial-of-wallet attacks
- Cannot reliably stop: indirect injection, obfuscation, multi-turn escalation, adversarial suffixes
- No guardrail achieves 100% detection — defense-in-depth is required
- Key distinction from model guardrails: external vs. internal, gateway vs. model control

---

← [AI Jailbreaking (What They Stop)](../ai-jailbreaking/overview.md) | [Model Guardrails →](../model-guardrails/overview.md)
