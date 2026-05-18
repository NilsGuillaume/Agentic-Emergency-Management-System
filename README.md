# 🚨 Voice-Agent Emergency Call Management System

> **An AI-powered telephone hotline that triages medical emergencies in real time — built as an ICCS Final Project by Nils Neuner**

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Twilio](https://img.shields.io/badge/Twilio-Voice_API-F22F46?style=flat-square&logo=twilio&logoColor=white)](https://twilio.com)
[![Deepgram](https://img.shields.io/badge/Deepgram-Speech--to--Speech-101010?style=flat-square&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0xMiAyQzYuNDggMiAyIDYuNDggMiAxMnM0LjQ4IDEwIDEwIDEwIDEwLTQuNDggMTAtMTBTMTcuNTIgMiAxMiAyem0tMSAxNEg5VjhIMTF2OHptNCAwSDEzVjhIMTV2OHoiLz48L3N2Zz4=)](https://deepgram.com)
[![OpenAI](https://img.shields.io/badge/OpenAI-GPT-412991?style=flat-square&logo=openai&logoColor=white)](https://openai.com)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-336791?style=flat-square&logo=postgresql&logoColor=white)](https://postgresql.org)
[![Render](https://img.shields.io/badge/Deployed_on-Render-46E3B7?style=flat-square&logo=render&logoColor=black)](https://render.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)

---

## 📞 Try It Live

Call the deployed system right now:

```
+1 (608) 549-5615
```

> ⚠️ The caller's phone number is stored in the database. This can be disabled on request.

---

## 📖 Table of Contents

- [Motivation](#-motivation)
- [System Architecture](#-system-architecture)
- [How It Works](#-how-it-works)
- [Setup Guide](#-setup-guide)
  - [Option A — Use the Hosted Environment](#option-a--use-the-hosted-environment-recommended)
  - [Option B — Self-Hosted Setup](#option-b--self-hosted-setup)
- [Running the GUI](#-running-the-gui)
- [Project Structure](#-project-structure)
- [Results & Limitations](#-results--limitations)
- [AI Usage Statement](#-ai-usage-statement)
- [Source Code](#-source-code)

---

## 💡 Motivation

In large-scale humanitarian crises, emergency call services are often the first bottleneck — a shortage of human operators means incoming emergencies go unrecorded and ambulances are delayed. At the same time, digital reporting tools require literacy and connectivity that vulnerable populations may not have.

**This project offers a simpler alternative: just call a phone number.**

An AI voice agent answers, guides the caller through a structured conversation to collect their location and emergency description, and stores everything in a database. A background triage algorithm then assigns an **Emergency Severity Index (ESI)** to each case. Emergency managers review and act on cases through a graphical desktop interface.

> This is a **proof of concept**, not a production-ready system. It is an exploration of both the potential and the significant ethical challenges of using AI in high-stakes medical contexts.

---

## 🏗 System Architecture

```
┌─────────┐     ┌─────────────┐     ┌───────────────┐     ┌────────────┐
│  Caller │────▶│ Voice Agent │────▶│ Triage System │────▶│ Management │
└─────────┘     └──────┬──────┘     └───────┬───────┘     └─────┬──────┘
                       │                    │                    │
                       └──────────┬─────────┘                    │
                                  ▼                              │
                           ┌─────────────┐                       │
                           │  PostgreSQL  │◀──────────────────────┘
                           │  Database   │
                           └─────────────┘
```

The system has **four principal components**:

| Component | Description |
|---|---|
| 🎙️ **Voice Agent** | Answers calls via Twilio, uses Deepgram for speech-to-speech AI conversation |
| 🏥 **Triage System** | LLM-based ESI classification triggered after each call |
| 🗄️ **Database** | PostgreSQL storing all call sessions, locations, descriptions, and ESI results |
| 🖥️ **Management GUI** | Desktop interface for emergency personnel to review and respond to cases |

### Streaming Architecture

For each incoming call, three asynchronous coroutines run concurrently:

```
Twilio ──[twilio_ws]──▶ twilio_receiver ──[audio_queue]──▶ sts_sender ──[sts_ws]──▶ Deepgram
  ▲                                                                                      │
  └────────────────────── sts_receiver ◀──────────────────────────────────────────────────┘
```

- **`twilio_receiver`** — receives audio chunks and call metadata from Twilio
- **`sts_sender`** — forwards audio to Deepgram in real time
- **`sts_receiver`** — receives synthesized agent audio and function call results from Deepgram; routes audio back to the caller

When the agent records an emergency description, the call's `streamSid` is pushed to an `evaluation_queue`. A background `eval_queue_manager` picks this up and triggers the triage system asynchronously — without interrupting the live call.

---

## ⚙️ How It Works

### 1. Voice Agent (`main.py` + `agent_functions.py` + `config.json`)

- **`main.py`** manages WebSocket connections between Twilio and Deepgram
- **`agent_functions.py`** exposes two function-calling tools to the agent:
  - `location_verifier` — geocodes the caller's address using `geopandas` and writes coordinates to the DB
  - `note_emergency_description` — appends new information to the call's emergency description (supports incremental updates as the caller reveals more)
- **`config.json`** defines the agent's system prompt, voice, LLM settings, and available function calls

### 2. Triage System (`triage_system.py`)

Uses an OpenAI LLM to evaluate each emergency description against the ESI framework:

```
ESI 1  →  Immediate life-saving intervention required?
  └── No → ESI 2  →  High-risk situation?
              └── No → ESI 3
```

Each classification includes a **textual justification** with both supporting and opposing reasoning, so human operators can review the AI's logic. Results are written back to the database automatically.

### 3. Management GUI (`GUI.py`)

Built with `tkinter` and `customtkinter`:

- Live table of all incoming calls with colour-coded ESI levels (🔴 ESI 1, 🟠 ESI 2)
- View emergency description and triage justification per call
- Manually override ESI level or mark a case as assigned
- Deferred save — changes batch-commit to the database on "Save Progress"

---

## 🚀 Setup Guide

### Option A — Use the Hosted Environment *(Recommended)*

The system is already deployed on [Render.com](https://render.com). Contact the author to be added as an **editor** — you'll have immediate access to a fully configured environment including Twilio, Deepgram, and the database.

Existing API keys can also be shared for testing purposes on request.

---

### Option B — Self-Hosted Setup

#### 1. Twilio & Deepgram

Follow this tutorial **up to minute 17:34**:

👉 [https://www.youtube.com/watch?v=hDKBREokidU&t=3637s](https://www.youtube.com/watch?v=hDKBREokidU&t=3637s)

This covers Twilio phone number configuration and Deepgram voice agent setup.

#### 2. Environment Variables

Create a `.env` file in the project root:

```env
DEEPGRAM_API_KEY=your_deepgram_key
OPENAI_API_KEY=your_openai_key
DATABASE_URL=your_postgresql_connection_string
```

#### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

#### 4. Database Setup

Create a PostgreSQL database, set `DATABASE_URL` in your `.env`, then run:

```bash
python locals/setup_db.py
```

#### 5. Run the Server

**For local testing with [ngrok](https://ngrok.com):**

Replace the `main()` function (lines 266–286 of `main.py`) with:

```python
async def main():
    await websockets.serve(
        twilio_handler,
        host="localhost",
        port=5000
    )
    print("Started Server")
    await asyncio.Future()

if __name__ == "__main__":
    asyncio.run(main())
```

Then start the server from the `hosted/` folder:

```bash
cd hosted
python main.py
```

> Make sure all files in the `hosted/` directory are present before starting.

---

## 🖥️ Running the GUI

The management interface runs **locally** and connects to the remote (or local) database.

```bash
# Install local dependencies
pip install -r locals/requirements.txt

# Create a .env with your DATABASE_URL
echo "DATABASE_URL=your_connection_string" > .env

# Run the GUI (ensure the Graphics/ folder is present)
python locals/GUI.py
```

---

## 📁 Project Structure

```
.
├── hosted/                      # Server-side components (deployed on Render)
│   ├── main.py                  # WebSocket server & streaming coordinator
│   ├── agent_functions.py       # Voice agent function-calling tools
│   ├── config.json              # Agent configuration (prompt, voice, LLM, functions)
│   ├── triage_system.py         # ESI classification logic
│   ├── db.py                    # Database helper functions
│   └── Triage_System_Prompts/   # External prompt files for triage criteria
│
└── locals/                      # Local components
    ├── GUI.py                   # Management interface (tkinter + customtkinter)
    ├── setup_db.py              # Database schema initialisation
    ├── requirements.txt         # Local dependencies
    └── Graphics/                # Icons and logos for the GUI
```

---

## 📊 Results & Limitations

> The results of this project should be interpreted with caution. This is an **exploratory prototype**, not an endorsement for real-world deployment.

### ✅ What Works

- End-to-end voice call handling with real-time AI conversation
- Automatic structured data capture (location, description) from unstructured phone calls
- Asynchronous LLM-based triage running without interrupting live calls
- Functional management GUI with live DB integration and manual overrides

### ⚠️ Known Limitations

| Area | Limitation |
|---|---|
| **Reliability** | LLM outputs are probabilistic; hallucinations or misclassifications could have severe consequences |
| **Security** | Health data flows through third-party APIs (Deepgram, OpenAI) — incompatible with GDPR/medical standards without major changes |
| **Scalability** | Not optimised for simultaneous high call volumes; Python's GIL limits true parallelism |
| **Robustness** | Voice agent can be confused by unexpected inputs; prompt engineering is not production-hardened |
| **Localization** | No integration with country-specific emergency protocols or languages |
| **Validation** | No testing against certified medical triage standards; no professional oversight built in |

### 🔭 Outlook

A production-ready version would require:
- All AI processing hosted within certified healthcare infrastructure
- Extensive clinical validation and regulatory compliance (EU AI Act classifies this as **high-risk**)
- Distributed architecture with load balancing for concurrent calls
- Multilingual support and local emergency protocol integration

---

## 🤖 AI Usage Statement

Generative AI tools (specifically ChatGPT) were used within the limits permitted by the course lecturer, in the following supportive capacities:

- Refining system prompts for the triage system and voice agent
- Debugging assistance for understanding error messages
- Paraphrasing and reformulating written materials (README, report)

**No AI-generated code was directly copied or integrated.** All code was written and reviewed independently.

---

## 🔗 Source Code

The full source code is available at:

👉 [https://cloud.uni-konstanz.de/index.php/f/227298721](https://cloud.uni-konstanz.de/index.php/f/227298721)

---

<div align="center">

**ICCS Final Project · Nils Neuner · University of Konstanz**

*Built with Twilio · Deepgram · OpenAI · PostgreSQL · Python*

</div>
