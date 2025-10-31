

# Intelligent Lead Qualification & Personalized Outreach

![Automation: n8n](https://img.shields.io/badge/Automation-n8n-20B2AA?logo=n8n) ![Telegram Bot](https://img.shields.io/badge/Interface-Telegram-26A5E4?logo=telegram) ![Google Sheets](https://img.shields.io/badge/Data-Google%20Sheets-34A853?logo=googlesheets) ![OpenAI GPT-4.1-mini](https://img.shields.io/badge/AI-GPT--4.1--mini-6366F1) ![HIL](https://img.shields.io/badge/Human--in--the--Loop-Required-000000) ![Timezone IST](https://img.shields.io/badge/Timezone-IST%20\(UTC%2B05%3A30\)-FF9800)

---

## 📁 Project Overview

**Lead Assistant** automates the flow from lead intake → qualification → personalized outreach → logging. It connects your **Google Form → Google Sheet** with a Telegram interface and two AI-driven subworkflows:

* **Lead Qual Tool** — evaluates incoming leads (via OpenAI) and updates the sheet with `Rating` and `Reasoning`.
* **Personal Message** — generates customized outreach for qualified leads and writes it back to the sheet.

---

## 🎯 Project Outcomes (Client-facing)

This system gives you:

* **Automated, consistent qualification** of every lead.
* **Ready-to-send, personalized outreach** for qualified prospects.
* **Centralized auditing** — every qualification decision and outreach message is recorded in Google Sheets.
* **Seamless operator experience** — Telegram-based confirmations and status updates.

### Estimated Impact (conservative, project-dependent)

* **Lead follow-up speed:** **20–40% faster** (reduced manual triage & drafting).
* **Manual hours spent on triage & messaging:** **30% reduction** (automation of repetitive copy tasks).
* **Lead engagement (opens/replies):** **+15–25%** uplift (personalized, timely outreach).
* **Operational consistency:** 100% repeatable qualification logic and audit trail.

---

## 🔧 Client Pain Points Addressed

* Slow, inconsistent lead triage → solved by automated qualification.
* Lack of scale in personalized outreach → solved by AI-generated messages.
* Missed leads and no audit trail → solved by automatic logging and `Reasoning` column.

---

## 🌊 Mermaid Diagram (high-level flow)

```mermaid
flowchart LR
  A[Telegram Trigger] --> B[Lead Assistant AI Agent]
  B --> C{Select Tool}
  C --> D[Lead Qual Tool (Subworkflow)]
  C --> E[Personal Message (Subworkflow)]
  D --> F[Google Sheet: Update Rating & Reasoning]
  E --> G[Google Sheet: Update Outreach]
  F --> H[Telegram: Send Response]
  G --> H
```

---

# ⚙️ Node-by-node Configuration (Main + Subworkflows)

---

## 🔹 Main Workflow — **Lead Assistant**

![Workflow Diagram](https://github.com/SachinSavkare/Lead-Qualification-Assistant-n8n/blob/main/4.1%20Lead%20Assistant%20AI%20Agent.JPG)

**Step 1 — Telegram Trigger**

* **Node name:** `Telegram Trigger`
* **Purpose:** Start on incoming Telegram messages.
* **Inputs available:** `message.text`, `message.chat.id`, message metadata.
* **Note:** `message.chat.id` used as `Session ID` for simple memory.

**Step 2 — Lead Assistant AI Agent**

* **Node name:** `Lead Assistant AI Agent`
* **Purpose:** Central router — calls subworkflow tools as required.
* **Model:** `gpt-4.1-mini` (primary) — fallback enabled
* **Memory:** Simple Memory — Session ID = `{{ $('Telegram Trigger').item.json.message.chat.id }}`, context window = 5.
* **Input (user prompt):** `{{ $json.message.text }}`

**System Message** :

```
You are the Lead Assistant AI Agent. Your job is to handle incoming lead-related queries and connect with the right tools instantly when required. You do not process the detailed qualification logic or craft the message yourself — the two subworkflows do that.

Tools available:
- Lead Qual Tool
- Personal Message

When a query requires qualification, call Lead Qual Tool and return its response. When a qualified-lead requires outreach, call Personal Message and return its response. Use the Google Sheet "3. Lets Work Together (Response)" and identify rows by Timestamp. When the requested process completes, respond exactly with: done
```

**Step 3 — Response (Telegram Send Message)**

* **Node name:** `Response`
* **Purpose:** Sends AI/subworkflow reply back to Telegram chat (use `message.chat.id`).

---

## 🔹 Subworkflow 4.2 — **Lead Qual Tool**

![Workflow Diagram](https://github.com/SachinSavkare/Lead-Qualification-Assistant-n8n/blob/main/4.2%20Lead%20Qual%20Tool.JPG)

**Step 1 — When Executed by Another Workflow**

* **Trigger:** Accept all data

**Step 2 — Google Sheets: Get Response**

* **Document:** `3. Lets Work Together (Response)`
* **Sheet:** `Form Responses 1`
* **Purpose:** Load the relevant lead row(s). Use `Timestamp` as row identifier.

**Step 3 — OpenAI Agent: LeadVerdict-45 AI Agent**

* **Model:** `GPT-4.1-MINI`
* **Prompt (User):**

```
Here is lead information:
Name: {{ $json['Full Name'] }}
Company Size: {{ $json['Company Size'] }}
Decision Maker: {{ $json["Are you a decision-maker for your company,s"] }}

Your task is to qualify the incoming lead. Reply in JSON only:
{"qualified?":"string","reasoning":"string"}
```

* **Simplify Output:** Enabled
* **Output Content:** JSON string (to be parsed next)

**Step 4 — Edit Field (Parse)**

* **Mapping:** `reply = {{ JSON.parse($json.message.content) }}`

**Step 5 — Merge**

* **Inputs:** Sheet row + parsed reply
* **Combine by:** Position

**Step 6 — Google Sheets: Update Rating and Reasoning**

* **Match on:** `Timestamp`
* **Update:** `Rating` = `{{ $json.reply['qualified?'] }}`, `Reasoning` = `{{ $json.reply.reasoning }}`

**Step 7 — Edit Field (Response to Main Workflow)**

* **Return:** `response = "done"`

---

## 🔹 Subworkflow 4.3 — **Personal Message**

![Workflow Diagram](https://github.com/SachinSavkare/Lead-Qualification-Assistant-n8n/blob/main/4.3%20Personal%20Message.JPG)

**Step 1 — When Executed by Another Workflow**

* **Trigger:** Accept all data

**Step 2 — Google Sheets: Get Response**

* **Document:** `3. Lets Work Together (Response)`
* **Sheet:** `Form Responses 1`
* **Purpose:** Read lead data for composing a message.

**Step 3 — OpenAI Agent: Personalized Lead Messenger AI Agent**

* **Model:** `GPT-4.1-MINI`
* **Prompt (User):**

```
Here is a lead's information:
Name: {{ $json['Full Name'] }}
Job Title: {{ $json['Job Title'] }}
What they are looking to Automate: {{ $json['What Area of Business are you looking to Audit ?'] }}
Rating: {{ $json.Rating }}
Preferred Contact Methods(s): {{ $json['Preferred Contact Methods(s)'] }}
Best Time to Contact ?: {{ $json['Best Time to Contact ?'] }}

If Rating == "Qualified", generate a personalized outreach message.
Return JSON with:
{
  "preferred_contact_method": "string",
  "best_time_to_contact": "string",
  "outreach_message": "string"
}
Messages must be from: Sachin Savkare, Codebasics Automation Corp. No placeholders.
If not Qualified, return empty output.
```

**Step 4 — Edit Field (Content Field)**

* **Mapping:** `reply = {{ $json.message.content }}`

**Step 5 — Merge**

* **Inputs:** Sheet row + reply

**Step 6 — Google Sheets: Update Outreach**

* **Match on:** `Timestamp`
* **Update:** `Outreach` = `{{ $json.reply }}`

**Step 7 — Edit Field (Response to Main Workflow)**

* **Return:** `response = "done"`

---


## 📥 **Free Workflow Template**

* **Download:** [Lead Assistant Agent](https://github.com/SachinSavkare/Lead-Qualification-Assistant-n8n/blob/main/4.1%20Lead%20Assistant%20AI%20Agent.json)
* **Download:** [⚙️Lead Qual Tool](https://github.com/SachinSavkare/Lead-Qualification-Assistant-n8n/blob/main/4.2%20%E2%9A%99%EF%B8%8FLead%20Qual%20Tool.json)
* **Download:** [⚙️Personal Message](https://github.com/SachinSavkare/Lead-Qualification-Assistant-n8n/blob/main/4.3%20%E2%9A%99%EF%B8%8FPersonal%20Message.json)
---

## 📋 Author

**Sachin Savkare**
📧 `sachinsavkare08@gmail.com`

---
