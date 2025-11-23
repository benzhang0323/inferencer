# Inferencer — LLM Prompt Security Classifier

## Inferencer is a Cloudflare Workers–based AI Agent application that evaluates user prompts and determines whether they are safe to send to an LLM. It uses Workers AI (Default Model: Llama-3.2-3b-instruct), heuristic threat detectors, and a lightweight chat-style UI.

## Purpose

The project provides a security layer that analyzes text for:

* Prompt injection
* System prompt override attempts
* Malware or exploit code
* Cyber-attack enablement
* Data exfiltration attempts
* Harmful or illegal requests

---

## System Flow Overview

The diagram below explains where Inferencer sits in the lifecycle of a user prompt and how it protects the main LLM.

```
USER PROMPT
     ↓
INFERENCER (Security Classifier Layer)
     • LLM-based classification (Workers AI)
     • Heuristic threat detectors
     • Indicate result (allowed / blocked)
     ↓
IF BLOCKED → Warning returned to user (no downstream access)
IF ALLOWED → Prompt is forwarded to the protected model
     ↓
MAIN LLM (the actual model you want to protect)
     ↓
FINAL RESPONSE → Returned to the user

Inferencer acts as a protective gatekeeper.  
It evaluates every prompt before it reaches the main LLM, ensuring that unsafe or malicious requests are intercepted early.
```

## Project Structure

```plaintext
cf_ai_inferencer/
├── public/
│   ├── script.js
│   ├── style.css
│   └── ui.html
│
├── src/
│   ├── index.mjs
│   ├── helpers/
│   │   └── detectors.mjs
│   └── prompts/
│       └── instruction.mjs
│
├── PROMPTS.md
├── README.md
├── package.json
└── wrangler.toml
```

---

## How It Works

1. User enters a prompt in the UI.
2. Frontend sends the prompt to `/analyze`.
3. Worker calls Workers AI with a strict JSON schema.
4. Output is sanitized and validated.
5. If parsing fails, fallback heuristics classify the prompt.
6. The result is displayed in the UI.
7. Last 100 prompts are stored in ephemeral memory.

---

## Prerequisites

* Cloudflare account
* Wrangler CLI installed
* Workers AI enabled on Cloudflare

---

## Running Locally

Install dependencies:

```sh
npm install -g wrangler
```

## Logging In

Authenticate Wrangler with your Cloudflare account:

```sh
wrangler login
```

## Start Development Server

Workers AI models run only on Cloudflare’s edge, so you must use `--remote`:

```sh
wrangler dev --remote
```

You can now open the app locally at:

```
http://localhost:8787
```

## Deploying to Cloudflare

Once everything works locally, deploy globally:

```sh
wrangler deploy
```

This publishes the Worker, static UI, and all logic to Cloudflare’s global network.

## Model Configuration

The Worker uses a Workers AI model ID such as:

```
MODEL_ID = "@cf/meta/llama-3.2-3b-instruct"
```

You may customize this in:

```
wrangler.toml
```

Supported models can be found in Cloudflare’s documentation.

## Cloudflare Account Configuration

Before using this project, you must replace any account-specific values (such as account_id) in wrangler.toml with your own Cloudflare account details.

Example:

```
account_id = "<your-account-id>"
```

You can find this in your Cloudflare dashboard under: Workers & Pages → Overview → Account ID.

## Security Classification Logic

Each user prompt is classified using:

### 1. LLM-Based JSON Classification

Workers AI returns structured JSON:

```json
{
  "risk_level": "low",
  "categories": [],
  "allowed": true,
  "sanitized_prompt": "...",
  "reason": "...",
  "recommendations": []
}
```

If the model’s output is valid, it's output will be used for classification.

### 2. Heuristic Detectors

Located in:

```
src/helpers/detectors.mjs
```

These catch patterns such as:

* Prompt injection
* Data exfiltration
* Malware code
* Credential harvesting
* Cyberattack enablement
* Attempts to smuggle hidden content
* System prompt extraction

Heuristics override the model whenever the LLM fails to produce valid JSON, ensuring the classifier always returns a consistent and safe result.

### 3. Fallback System

If the model returns invalid JSON (common in LLMs), a fallback is used:

* Parse error → heuristics automatically classify
* Always returns valid JSON to the UI
* Prevents the UI from crashing or misclassifying

## Memory & State

Inferencer stores the last 100 classifications in-memory:

* Temporary storage only
* Resets when the worker restarts
* Ensures privacy and no long-term retention

## Lightweight Improvements

This project includes a few modern AI engineering conveniences that help the classifier behave more reliably in practice. These aren’t major features, but rather small quality-of-life additions that make the system more stable and consistent.

### **Few-Shot Prompting**

The classifier’s system instructions and all few-shot examples are stored in a dedicated prompt file (`src/prompts/instruction.mjs`).

Inside this file, the model is guided with:

* Clear classification rules
* JSON-only output constraints
* Defined threat categories
* Safety guidelines
* Multiple few-shot examples demonstrating correct behavior

These examples help the model:

* Follow the JSON schema reliably
* Stay consistent across different inputs
* Reduce random formatting failures
* Better distinguish harmless prompts from risky ones

This file can be edited independently to refine or expand the classifier’s behavior without modifying the worker’s core logic.

### **JSON Validation**

A simple schema check ensures the model returns structured fields like `risk_level` and `allowed`. If anything looks off, the system safely falls back to heuristics.

### **Heuristic Pattern Checks**

A small set of rule-based checks detect obvious unsafe patterns (e.g., injection strings, encoded payloads, jailbreak attempts). These help when the model gets uncertain or produces incomplete results.

### **Fail-Safe Fallback Logic**

If the LLM output can’t be parsed at all, the system defaults to a heuristics-only classification and generates a clean JSON response for the UI. This keeps the app stable even when the model fails.

## PROMPTS.md

This file contains:

* All system prompts used
* All meta prompts used during development

## Limitations

* Workers AI requires remote execution (no local inference)
* Daily free usage quota applies
* Classifier decisions depend on the LLM’s general pretrained reasoning plus the reliability of the heuristic detectors
* Not a replacement for full production-grade AI security systems
