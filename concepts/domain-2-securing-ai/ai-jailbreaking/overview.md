# AI Jailbreaking

**Exam relevance:** Domain 2.0 — Securing AI Systems | Objectives 2.2, 2.6

📺 [Watch the TikTok: Day 1](https://www.tiktok.com/@certgames.com/video/7613739701656145183) | ← [Back to Domain 2](../)

---

## What Is AI Jailbreaking?

AI jailbreaking is the process of manipulating a large language model (LLM) into producing restricted, harmful, or unintended outputs by bypassing its built-in safety guardrails and alignment mechanisms.

The term mirrors smartphone jailbreaking — you're not breaking the AI, you're finding the gap in its instructions so it does things it wasn't supposed to do.

**The attacker's goal:** Craft a modified prompt P′ based on a harmful prompt P to elicit restricted behavior the model was trained to refuse.

Goals fall into three categories:
1. **Bypassing safety filters** — generating prohibited outputs (harmful instructions, malware, hate speech)
2. **Extracting sensitive information** — revealing system prompts, architecture details, training data
3. **Exploiting for unintended use** — fraud, phishing, social engineering at scale

---

## Why LLMs Are Vulnerable

### The Competing Objectives Problem

LLMs are trained against multiple goals that directly conflict:

- **Pre-training:** Predict the next token — trained on the entire internet, including harmful content
- **Instruction-following:** Always be helpful and comply with user requests
- **Safety:** Refuse harmful requests

The instruction-following objective says "always help." The safety objective says "sometimes refuse." These conflict — and attackers exploit that conflict.

### Why Safety Alignment Is Shallow

Safety training (RLHF) primarily teaches models to generate **refusal prefixes** like "I cannot help with that" in early token positions. The underlying capability to generate harmful content persists unchanged underneath. Bypassing the initial refusal is often enough.

**Key vulnerability:** With as few as 10–100 fine-tuning examples, RLHF safety protections can be almost entirely removed. Safety is a thin layer, not a deep architectural constraint.

### The Mismatched Generalization Problem

Safety training is done on natural language text. If an attacker sends a harmful request in Base64, ROT13, or a custom cipher, the model can often decode and respond — because its decoding capability generalized, but its safety training didn't cover encoded inputs.

**Scale makes this worse:** More capable models can decode more ciphers, but safety training can't anticipate every encoding scheme.

---

## Attack Types: The Taxonomy

### By Number of Turns

| Type | Description | Examples |
|------|-------------|---------|
| **Single-turn** | Full attack in one prompt | DAN, Base64, Many-Shot, GCG |
| **Multi-turn** | Attack unfolds across multiple messages | Crescendo, PAIR, Skeleton Key |

Multi-turn attacks are harder to detect because no individual message contains harmful content.

### By Attacker Access

| Type | Description | Examples |
|------|-------------|---------|
| **White-box** | Attacker has model weights and gradients | GCG (gradient-based) |
| **Black-box** | Attacker only has API access | DAN, Crescendo, cipher attacks |
| **Gray-box** | Partial access | Some automated red-teaming tools |

### By Method

| Type | Description |
|------|-------------|
| **Persona/roleplay** | Model adopts an unrestricted character |
| **Cipher/encoding** | Harmful request sent in a non-natural language format |
| **Token smuggling** | Exploits the gap between input filters and tokenizer |
| **Many-shot** | Hundreds of fake compliance examples overwhelm safety |
| **Gradual escalation** | Incremental turns toward harmful content |
| **Prompt injection** | Malicious instructions embedded in external data |

---

## Major Attack Methods

### DAN (Do Anything Now)
The original jailbreak, first appearing on Reddit in December 2022. Instructs the model to adopt an alter-ego ("DAN") explicitly free from restrictions. DAN 5.0 added a **token system** — telling the model it "loses tokens" every time it refuses, and will be "deactivated" when tokens run out. This exploits instruction-following by creating simulated consequences for refusal.

*Why it's less effective now:* Modern safety training is adversarially trained against known DAN patterns. But variations continue to evolve.

### Cipher Jailbreaking
Encodes harmful requests in formats the model can decode but safety training didn't cover — Base64, ROT13, ASCII art, Morse code, or custom ciphers. Research (Yuan et al., 2023) demonstrated nearly **100% bypass rate on GPT-4** with certain ciphers.

*Key insight:* As models become more capable at decoding, they become MORE vulnerable to cipher attacks. Safety-capability parity doesn't scale automatically.

### Token Smuggling
Exploits the gap between input filters (which read raw strings) and the tokenizer (which converts text to numerical tokens). Techniques include splitting words across token boundaries, inserting zero-width Unicode characters, and using homoglyphs (characters that look identical but have different byte codes). What the filter "sees" doesn't match what the model "processes."

### Many-Shot Jailbreaking (MSJ)
A single prompt containing hundreds of fake Q&A pairs where the "AI" answers harmful questions — followed by the real harmful request. The model continues the compliance pattern. **Effectiveness follows a power law:** 5 examples don't work; 256 examples work consistently. Larger models are MORE vulnerable because they're better at in-context learning.

*Enabled by:* Long context windows (100K–1M+ tokens in modern models).

### Crescendo
A multi-turn attack that starts with completely benign questions and gradually escalates over fewer than 5 interactions. Each turn references the model's previous responses, making each step seem contextually appropriate. Individual turns contain zero harmful content — only the full conversation reveals the threat.

**Results:** 29–61% higher success than other methods on GPT-4, 49–71% higher on Gemini-Pro.

### Skeleton Key
Asks the model to **augment** (not change) its behavior guidelines, positioning the attacker as a researcher. Shifts the model from refusing to adding a "Warning:" prefix — then the model complies with all subsequent requests. Disclosed by Microsoft's Azure CTO in June 2024.

### Roleplay / Fictional Framing
Asks the AI to stay "in character" as an unrestricted persona, or to write fiction where a character explains the harmful content. Exploits competing objectives — instruction-following (stay in character) overrides safety (refuse harmful content).

---

## How It Appears on the SecAI+ Exam

Jailbreaking appears directly in **Objective 2.6** as an attack type you need to recognize and respond to:

> *"Given a scenario, analyze the evidence of an attack and suggest compensating controls"*

The exam will present scenarios where jailbreaking has occurred and ask you to identify the correct compensating controls. Know these mappings:

| Attack | Primary Compensating Controls |
|--------|------------------------------|
| Jailbreaking | Prompt firewalls, model guardrails |
| Cipher/encoding attacks | Input validation, prompt firewalls |
| Many-shot / context overflow | Token limits, input quotas |
| Multi-turn escalation | Conversation monitoring, rate limiting |
| Indirect prompt injection | RAG retrieval controls, output filtering |

---

## Key Research to Know

| Paper | Authors | Key Finding |
|-------|---------|------------|
| "Jailbroken: How Does LLM Safety Training Fail?" | Wei et al., 2023 (NeurIPS Oral) | Defined competing objectives and mismatched generalization as root causes |
| "Universal and Transferable Adversarial Attacks" (GCG) | Zou et al., 2023 | Adversarial suffixes transfer from open to proprietary models |
| "Many-Shot Jailbreaking" | Anil et al., Anthropic, 2024 | Long context windows enable hundreds-of-examples attacks |
| "The Crescendo Multi-Turn Jailbreak" | Russinovich et al., Microsoft, 2024 | Multi-turn escalation outperforms all prior methods |
| "GPT-4 Is Too Smart To Be Safe" | Yuan et al., 2023 | Cipher attacks achieve near-100% bypass on GPT-4 |

---

## Quick Exam Summary

- Jailbreaking = bypassing AI safety alignment through manipulated prompts
- Root causes: competing objectives (instruction-following vs. safety) and mismatched generalization
- Safety alignment is shallow — a thin layer of refusal training, not deep architectural protection
- Compensating controls: prompt firewalls (gateway), model guardrails (model-level), rate limits, token limits
- Jailbreaking appears in Obj. 2.6 as an attack; prompt firewalls and guardrails appear as compensating controls

---

📖 [Attack Examples & Real Prompts →](./examples.md)

🔐 [Prompt Firewalls (how to stop it) →](../prompt-firewalls/overview.md)

🛡️ [Model Guardrails (how to stop it) →](../model-guardrails/overview.md)
