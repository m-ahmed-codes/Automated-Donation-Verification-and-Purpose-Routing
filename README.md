# Automated Donation Verification & Purpose Routing

> Built for **[Build with AI Hackathon 2026]** — Problem Statement 7

An n8n-powered workflow that automates the donation verification bottleneck for high-volume nonprofits like Al-Khidmat: a donor sends a bank transfer screenshot over WhatsApp, an LLM extracts the transaction details, the system cross-checks it against the bank statement, captures the donation's purpose conversationally, and sends back an instant confirmation — replacing what used to be a fully manual, per-screenshot human review process.

![Workflow Overview](./assets/workflow-screenshot.jpeg)

🎥 **Demo video:** [Watch here](./assets/demo.mp4) <!-- or paste a YouTube/Drive link -->

---

## Table of Contents
- [The Problem](#the-problem)
- [Our Solution](#our-solution)
- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Setup & Installation](#setup--installation)
- [Configuration](#configuration)
- [Running the Workflow](#running-the-workflow)
- [Sample Data](#sample-data)
- [Known Limitations](#known-limitations)
- [Future Improvements](#future-improvements)

---

## The Problem

Thousands of donations reach nonprofits like Al-Khidmat every month through direct bank transfers (IBFT, branch deposit, cheque) rather than card gateways. Because partner banks don't push real-time sender details, the organization only receives a generic credit notification — donors must separately send a payment screenshot over WhatsApp to get their donation attributed to them.

This creates a heavy manual bottleneck. Staff must, for every single screenshot:
- Read the transfer amount, time, and reference number off a (often low-quality) image
- Cross-check it against a periodically-sent bank statement PDF to confirm it's a real transaction
- Figure out what the donation is *for* (Gaza appeal, ration drive, orphan sponsorship, etc.) from a free-text WhatsApp message — or sometimes nothing at all
- Route it to the correct project ledger and confirm back to the donor

At scale — hundreds of messages a day during Ramzan or a relief appeal — this becomes one of the organization's most expensive manual workflows.

## Our Solution

This project automates the entire pipeline end-to-end using **n8n** for orchestration and **GPT-4 Vision** for the AI-heavy steps:

1. **Screenshot OCR + structured extraction** — pulls amount, date, time, sender, bank, and reference number from messy real-world screenshots (rotated, cropped, low-res) across multiple Pakistani banks, in English, Urdu, and Roman Urdu.
2. **Cross-verification against the bank statement** — deterministic matching logic flags a donation as `verified`, `pending` (valid screenshot, statement entry not posted yet), or `flagged` (mismatch, needs human review).
3. **Conversational purpose capture** — donor replies in free text ("Gaza ke liye", "orphan sponsorship"); an LLM maps it to the correct active campaign, falling back to a menu if confidence is low.
4. **Auto-confirmation & receipt** — donor gets an instant WhatsApp confirmation with a donation reference and project assignment.
5. **Coordinator visibility** — every donation, its extracted fields, and its status are logged centrally so staff can review flagged/pending cases without re-reading every screenshot manually.

---

## How It Works

```
Donor sends screenshot on WhatsApp
        │
        ▼
n8n Webhook receives the message
        │
        ▼
Image downloaded from WhatsApp Media API
        │
        ▼
GPT-4 Vision extracts: bank, amount, date, time, sender, reference number
        │
        ▼
Extraction cross-checked against bank statement (Code node logic)
        │
   ┌────┼────────────┐
   ▼    ▼             ▼
Verified  Pending   Flagged
   │       │            │
   ▼       ▼            ▼
Ask donor "what's this for?" / Notify donor it's under review
        │
        ▼
LLM classifies purpose → active project (with menu fallback if unsure)
        │
        ▼
Donation logged + confirmation sent back on WhatsApp
```

## Architecture

| Layer | Responsibility |
|---|---|
| **WhatsApp Cloud API** | Donor-facing chat interface — receives screenshots and free-text replies |
| **n8n** | Orchestration: webhooks, routing, state management, API calls |
| **GPT-4 Vision (OpenAI API)** | Screenshot → structured JSON extraction |
| **GPT (text)** | Free-text purpose → project category classification |
| **Google Sheets** | Bank statement data, donations log, active projects list (acts as lightweight DB) |


---

## Tech Stack
- **Orchestration:** [n8n](https://n8n.io/) (self-hosted / n8n cloud)
- **AI/LLM:** OpenAI GPT-4 Vision (extraction) + GPT-4 (purpose classification)
- **Messaging:** WhatsApp Cloud API (Meta)
- **Data store:** Google Sheets (bank statement, donations log, active projects)


---

## Repository Structure

```
.
├── workflow.json                  # n8n workflow export — import this directly into n8n
├── README.md
├── assets/
│   ├── workflow-screenshot.png    # Screenshot of the full n8n canvas
│   └── demo.mp4                   # Short demo recording
├── sample-data/
│   ├── sample-screenshots/        # Anonymised test screenshots (multiple banks)
│   ├── bank-statement-sample.pdf  # Synthetic bank statement used for matching
│   └── active-projects.csv        # List of campaigns the bot routes donations to

```

---

## Setup & Installation

### Prerequisites
- An [n8n](https://n8n.io/) instance — either:
  - **n8n Cloud** (easiest, no install needed), or
  - **Self-hosted**, via:
    ```bash
    npx n8n
    ```
- An [OpenAI API key](https://platform.openai.com/api-keys) with GPT-4 Vision access
- A [Meta Developer account](https://developers.facebook.com/) with a WhatsApp Business app set up (Cloud API) and a test phone number
- A Google account (for Google Sheets as the data store)
-

### 1. Clone the repository
```bash
git clone https://github.com/HaziqAhmed07/Automated-Donation-Verification-and-Purpose-Routing.git
cd Automated-Donation-Verification-and-Purpose-Routing
```

### 2. Import the workflow into n8n
1. Open your n8n instance
2. Click **Workflows → Import from File**
3. Select `workflow.json` from this repo
4. The full node graph will load onto the canvas

### 3. Set up credentials in n8n
You'll need to add the following credentials inside n8n (**Settings → Credentials**):

| Credential | Used for |
|---|---|
| OpenAI API Key | GPT-4 Vision extraction + purpose classification nodes |
| WhatsApp Cloud API (Meta access token) | Receiving/sending WhatsApp messages |
| Google Sheets OAuth2 | Reading/writing the bank statement, donations log, and projects sheets |

### 4. Set up the Google Sheets data store
Create a Google Sheet with **3 tabs**, matching the structure below (or copy the sample sheet linked in `sample-data/`):

**Tab 1 — `BankStatement`**
| date | time | amount | reference | sender_name | matched |
|---|---|---|---|---|---|

**Tab 2 — `DonationsLog`**
| donation_id | donor_phone | extracted_amount | extracted_date | extracted_time | extracted_bank | extracted_ref | status | purpose | project_assigned |
|---|---|---|---|---|---|---|---|---|---|



Populate `BankStatement` using the sample PDF in `sample-data/`, and `ActiveProjects` using `active-projects.csv`.

### 5. Connect WhatsApp Cloud API to n8n
1. In your Meta Developer App, set the webhook URL to your n8n webhook's **production URL**
   (find this in n8n on the Webhook node — it looks like `https://your-instance.app.n8n.cloud/webhook/...`)
2. Subscribe the webhook to the `messages` field
3. If self-hosting locally, run `ngrok http 5678` and use the generated HTTPS URL instead

---

## Configuration

Open these nodes in the imported workflow and update the placeholders:

| Node | What to change |
|---|---|
| `Gemini Vision Extract` / `GPT-4 Vision Extract` | Confirm your OpenAI credential is selected |
| `Verify Against Statement` (Code node) | Adjust the time-window tolerance (default: 30 min) if needed |
| `Gemini Classify Purpose` / `GPT Classify Purpose` | Update the project list in the prompt if your active campaigns differ |
| `Send WhatsApp Reply` (all instances) | Confirm your WhatsApp phone number ID is correct |

---

## Running the Workflow

1. Activate the workflow in n8n (toggle **Active** in the top right)
2. From a WhatsApp number, send a transaction screenshot to your test business number
3. The bot will reply with a verification status, then ask for the donation's purpose
4. Reply with the purpose in plain text (e.g. *"Gaza ke liye"*)
5. You'll receive a confirmation message with a donation reference number
6. Check the `DonationsLog` sheet — the row should now be fully populated with `status: completed`

**For judges/testers without WhatsApp access:** use the standalone web upload form (see `docs/` for the link/instructions) — it sends the same payload shape into the same n8n workflow, so the underlying extraction and verification logic is identical to the WhatsApp path.

---

## Sample Data

The `sample-data/` folder includes:
- A handful of anonymised real-world bank transfer screenshots across multiple Pakistani banks (HBL, Meezan, UBL, JazzCash, EasyPaisa, Allied Bank, Bank Alfalah)
- A synthetic bank statement PDF with seeded transactions — some matching the screenshots exactly, some mismatched, some "near-matches" intended to trigger the `flagged` status
- A sample list of active donation campaigns/projects

Use these to test the workflow end-to-end without needing real donor data.

---

## Known Limitations

- Fake-screenshot detection is limited to amount/timestamp/reference cross-checking against the statement — it does **not** perform pixel-level image forgery detection. Suspicious mismatches are flagged for human review rather than auto-rejected.
- Extraction accuracy depends on screenshot quality; very low-resolution or heavily cropped images may return partial data and get flagged automatically rather than mis-verified.
- Bank statement matching currently assumes a single statement file is loaded per session; real-time bank API integration is out of scope for this demo.

## Future Improvements

- Real-time bank statement feed integration (where available) instead of periodic PDF uploads
- A proper coordinator dashboard with manual override and audit trail
- Formal PDF tax receipt generation and delivery
- Multi-currency / overseas donor support

---



Built for **[Build With Ai Hackathone -2026]**, [June 2026].
