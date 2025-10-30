# HR-ASSIST — Agentic AI MCP Server for HR Workflows

HR-ASSIST is a **production-style, agentic AI backend** that exposes HR operations (employee onboarding, leave, meetings, tickets, and email notifications) as **MCP tools** to an MCP-compatible client (e.g. **Claude Desktop**). The goal is to let an AI assistant **drive real HR workflows** end-to-end — “add employee → notify manager → raise tickets → schedule intro meeting” — without the human needing to click through multiple systems.

This repository contains the **MCP server** implementation, in-memory **HRMS services**, and the **tooling** that an MCP client can call.

> ✅ **Target audience**: someone new to the repo who wants to run the server locally and connect it to Claude Desktop.  
> ✅ **Outcome**: after following this README, you should be able to 1) start the MCP server, 2) see the tools in Claude, and 3) run the “onboard_new_employee” workflow.

---

## Table of Contents

1. [What This Project Is](#what-this-project-is)
2. [Architecture Overview](#architecture-overview)
   - [Mermaid Diagram](#mermaid-diagram)
   - [Data Flow](#data-flow)
3. [Repository Structure](#repository-structure)
4. [Prerequisites](#prerequisites)
5. [Installation & Setup](#installation--setup)
   - [Option A: using `uv` (recommended)](#option-a-using-uv-recommended)
   - [Option B: using `pip`](#option-b-using-pip)
6. [Claude Desktop MCP Configuration](#claude-desktop-mcp-configuration)
7. [Environment Variables](#environment-variables)
8. [Running the MCP Server](#running-the-mcp-server)
9. [Available Tools (MCP Endpoints)](#available-tools-mcp-endpoints)
10. [How Onboarding Works End-to-End](#how-onboarding-works-end-to-end)
11. [Extending the System](#extending-the-system)
12. [Troubleshooting](#troubleshooting)
13. [Notes & Limitations](#notes--limitations)
14. [License](#license)

---

## What This Project Is

**In one sentence:**  
This is an **MCP server** that exposes HR primitives — employees, leave, meetings, tickets, emails — so that an **agentic client** (like Claude Desktop) can **call them as tools** and orchestrate an HR workflow such as onboarding.

**Key ideas:**

- The **client (Claude Desktop)** is the “brain” / agent.
- This **repo is the “hands”** — it exposes concrete actions over MCP.
- HR data is **in-memory** (seeded via `utils.seed_services(...)`) to make the demo self-contained.
- Email is sent via an **SMTP sender** (`EmailSender`), which expects credentials from env vars.
- All HR operations are wrapped as **MCP tools** using `fastmcp`.

---

## Architecture Overview

At runtime, your setup looks like this:

1. **Claude Desktop** receives a user prompt (“Onboard Rohan under Sarah”)
2. Claude calls the **MCP server** (`server.py`) over STDIO
3. MCP server exposes tools: `add_employee`, `send_email`, `create_ticket`, `schedule_meeting`, …
4. The server delegates to **HRMS service layer** (`HRMS/…`)
5. Data is **seeded** on startup (sample employees, meetings, tickets)
6. Email is sent via an SMTP backend using env vars

### Mermaid Diagram

```mermaid
flowchart TD

    subgraph Client
        A[Claude Desktop<br/>(MCP client)]
    end

    subgraph MCPServer[This repo: hr-assist MCP server]
        B[server.py<br/>FastMCP("hr-assist")]
        C[utils.seed_services(...)<br/>→ in-memory data]
        subgraph HRMS[HRMS service layer]
            E1[EmployeeManager]
            E2[LeaveManager]
            E3[MeetingManager]
            E4[TicketManager]
            E5[Schemas (Pydantic)]
        end
        D[EmailSender<br/>(SMTP)]
    end

    A -- MCP Call (stdio) --> B
    B --> C
    B --> HRMS
    B --> D

    A -- prompt: onboard_new_employee --> B

    HRMS -->|creates/reads| C
    D -->|SMTP| F[(Email Provider)]

    style MCPServer fill:#f9f9f9,stroke:#999,stroke-width:1px
    style HRMS fill:#fff,stroke:#bbb,stroke-width:1px
```

---

### Data Flow

1. **Startup**
   - `server.py` imports HRMS managers from `HRMS/`
   - `utils.seed_services(...)` populates: employees, leave balances, meetings, tickets
   - `EmailSender` is initialized with `CB_EMAIL` + `CB_EMAIL_PWD`
   - `FastMCP("hr-assist")` registers tools

2. **Client call (example: onboarding)**
   - Claude shows tool `onboard_new_employee`
   - Claude sends structured args (`employee_name`, `manager_name`)
   - MCP server:
     1. creates employee via `EmployeeManager`
     2. sends welcome email
     3. creates tickets (laptop, ID card, etc.)
     4. schedules intro meeting

3. **Data is NOT persisted to disk** — it lives only in memory for this demo.

---

## Repository Structure

```text
└── rohan0497-hr-assist/
    ├── README.md                # (you can replace with this file)
    ├── main.py                  # simple entrypoint (demo)
    ├── pyproject.toml           # project metadata + deps
    ├── server.py                # ✅ main MCP server (run this)
    ├── utils.py                 # ✅ seeds in-memory HRMS data
    ├── .python-version          # uses Python 3.11
    └── HRMS/
        ├── __init__.py          # re-exports managers and schemas
        ├── employee_manager.py  # employees CRUD + search
        ├── leave_manager.py     # leave balance, history, apply
        ├── meeting_manager.py   # schedule / list / cancel
        ├── schemas.py           # Pydantic models used by server
        └── ticket_manager.py    # create / update / list tickets
```

**Important files to read first:**

- `server.py` → how tools are defined (`@mcp.tool()`, `@mcp.prompt()`)
- `utils.py` → how demo data is created
- `HRMS/schemas.py` → the data contracts

---

## Prerequisites

- **Python**: `3.11` (repo already has `.python-version` = 3.11)
- **Claude Desktop** (or another MCP-capable client)
- **SMTP credentials** (for the email tool to actually send mail)
  - environment vars:
    - `CB_EMAIL`
    - `CB_EMAIL_PWD`
- **OS**: works on Windows/macOS/Linux (example config below is Windows-style)

> 📦 The project uses **`mcp[cli] >= 1.19.0`** (declared in `pyproject.toml`)

---

## Installation & Setup

You can install this in **two** ways.

### Option A: using `uv` (recommended)

This is the style already shown in the original README snippet.

1. **Clone the repo**
   ```bash
   git clone <your-fork-or-path>/rohan0497-hr-assist.git
   cd rohan0497-hr-assist
   ```

2. **Create environment**
   ```bash
   uv init
   ```

3. **Install deps**
   ```bash
   uv add "mcp[cli]>=1.19.0"
   uv add python-dotenv
   ```

   > `python-dotenv` is used in `server.py` via `load_dotenv()`.

4. **Run the server (local test)**
   ```bash
   uv run server.py
   ```

   This will start the MCP server on **stdio** (i.e. it’s ready for Claude).

---

### Option B: using `pip`

If you don’t want `uv`:

1. **Clone**
   ```bash
   git clone <your-fork-or-path>/rohan0497-hr-assist.git
   cd rohan0497-hr-assist
   ```

2. **(optional) Create venv**
   ```bash
   python -m venv .venv
   source .venv/bin/activate     # on macOS/Linux
   # or
   .venv\Scripts\activate        # on Windows
   ```

3. **Install**
   ```bash
   pip install "mcp[cli]>=1.19.0" python-dotenv
   ```

4. **Run**
   ```bash
   python server.py
   ```

---

## Claude Desktop MCP Configuration

To make Claude Desktop **see** this MCP server, add the following to your `claude_desktop_config.json` (or whichever config file Claude is using):

```json
{
  "mcpServers": {
    "hr-assist": {
      "command": "C:\\Users\\username\\.local\\bin\\uv",
      "args": [
        "--directory",
        "C::\\code\\hr-assist",
        "run",
        "server.py"
      ],
      "env": {
        "CB_EMAIL": "YOUR_EMAIL",
        "CB_EMAIL_PWD": "YOUR_APP_PASSWORD"
      }
    }
  }
}
```

**Notes:**

- Change paths to match **your** machine.
- If you’re **not** using `uv`, set `"command": "python"` and make args `["server.py"]`.
- If you are on **macOS/Linux**, use POSIX paths.
- `env` is the **easiest** way to inject SMTP secrets.

---

## Environment Variables

`server.py` does this:

```python
from dotenv import load_dotenv
load_dotenv()
```

So you can use either:

1. a local `.env` file:

   ```env
   CB_EMAIL=your_email@example.com
   CB_EMAIL_PWD=your_app_password
   ```

   and then just run `python server.py`

**OR**

2. you can pass them via Claude’s MCP config (as shown above)

**Why needed?**

- `EmailSender(...)` is constructed like this:

  ```python
  emailer = EmailSender(
      smtp_server="smtp.gmail.com",
      port=587,
      username=os.getenv("CB_EMAIL"),
      password=os.getenv("CB_EMAIL_PWD"),
      use_tls=True
  )
  ```

- So if these are missing, `send_email(...)` tool will fail.

> ⚠️ You must also have an **`emails` module** available in your environment (`from emails import EmailSender`). If your environment does **not** have that package, either install it (if it’s a private/local module) or create a minimal `emails.py` with an `EmailSender` class that wraps `smtplib`.

---

## Running the MCP Server

From the repo root:

```bash
python server.py
# or
uv run server.py
```

- It will bind to **stdio** (this is what Claude expects)
- Keep this process **running** while you test in Claude
- Once Claude connects, you should see the tools listed

---

## Available Tools (MCP Endpoints)

These are defined in `server.py` using `@mcp.tool()`:

1. **`add_employee(emp_name: str, manager_id: str, email: str)`**
   - Uses `EmployeeManager`
   - Auto-generates `emp_id`
   - ✅ use this during onboarding

2. **`get_employee_details(name: str)`**
   - Fuzzy search
   - Returns employee dict

3. **`send_email(to_emails: List[str], subject: str, body: str, html: bool = False)`**
   - Uses SMTP credentials from env
   - Returns `"Email sent successfully."`

4. **`create_ticket(emp_id: str, item: str, reason: str)`**
   - Creates ticket like `T0001`
   - Used for “new laptop”, “ID card”, etc.

5. **`update_ticket_status(ticket_id: str, status: str)`**
   - Status can be `Open | In Progress | Closed | Rejected`

6. **`list_tickets(employee_id: str, status: str)`**
   - Filter by employee & status

7. **`schedule_meeting(employee_id: str, meeting_datetime: datetime, topic: str)`**
   - Stores meeting in-memory

8. **`get_meetings(employee_id: str)`**

9. **`cancel_meeting(employee_id: str, meeting_datetime: datetime, topic: str)`**

10. **`get_employee_leave_balance(emp_id: str)`**

11. **`apply_leave(emp_id: str, leave_dates: list)`**

12. **`get_leave_history(emp_id: str)`**

---

### Prompts (MCP)

There is also an **MCP prompt** defined:

```python
@mcp.prompt("onboard_new_employee")
def onboard_new_employee(employee_name: str, manager_name: str):
    ...
```

This is a **high-level workflow** expressed as instructions for the agent:

- add employee
- send welcome email
- notify manager
- raise tickets
- schedule meeting

**Claude will read this and call the tools above** in sequence.

---

## How Onboarding Works End-to-End

Here’s the exact flow, step-by-step:

1. **User action (in Claude):**
   - “Onboard Jane Doe under Sarah Johnson”
   - or selects prompt “onboard_new_employee”

2. **Claude → MCP:**
   - Calls `onboard_new_employee(employee_name="Jane Doe", manager_name="Sarah Johnson")`
   - Claude interprets the returned instructions

3. **Claude → MCP (tool calls):**
   1. `add_employee(emp_name="Jane Doe", manager_id="<resolved-id>", email="jane.doe@atliq.com")`
   2. `send_email(to=["jane.doe@atliq.com"], subject="Welcome to Atliq", ...)`
   3. `create_ticket(emp_id="E009", item="Laptop", reason="New hire setup")`
   4. `create_ticket(emp_id="E009", item="ID Card", reason="New hire setup")`
   5. `schedule_meeting(employee_id="E009", meeting_datetime=..., topic="Intro with manager")`

4. **Server side:**
   - Employee stored in `EmployeeManager.employees`
   - Tickets stored in `TicketManager.tickets`
   - Meetings stored in `MeetingManager.meetings`
   - Email sent via SMTP

5. **Result:**
   - New person added
   - HR artifacts created
   - Agent can query back for verification (e.g. `list_tickets`, `get_meetings`)

---

## Extending the System

You can add new HR operations in **three steps**:

1. **Add a method** in `HRMS/<manager>.py`  
   e.g. a “performance review” manager

2. **Expose it in `server.py`** as an MCP tool:
   ```python
   @mcp.tool()
   def create_performance_review(emp_id: str, quarter: str) -> str:
       return perf_manager.create_review(emp_id, quarter)
   ```

3. **(Optional) Add it to the onboarding prompt** if you want the agent to do it by default.

---

## Troubleshooting

**1. “ModuleNotFoundError: No module named 'emails'”**  
→ You need to provide an `emails` module or install the one the original author used. Minimal stub:

```python
# emails.py
import smtplib
from email.mime.text import MIMEText

class EmailSender:
    def __init__(self, smtp_server, port, username, password, use_tls=True):
        self.smtp_server = smtp_server
        self.port = port
        self.username = username
        self.password = password
        self.use_tls = use_tls

    def send_email(self, subject, body, to_emails, from_email=None, html=False):
        msg = MIMEText(body, "html" if html else "plain")
        msg["Subject"] = subject
        msg["From"] = from_email or self.username
        msg["To"] = ", ".join(to_emails)

        with smtplib.SMTP(self.smtp_server, self.port) as server:
            if self.use_tls:
                server.starttls()
            server.login(self.username, self.password)
            server.sendmail(msg["From"], to_emails, msg.as_string())
```

**2. Email not sending**  
- Check `CB_EMAIL` and `CB_EMAIL_PWD`
- If using Gmail: you may need an **app password**
- Check SMTP port `587`

**3. Claude doesn’t show the MCP**  
- Check path in `claude_desktop_config.json`
- Run `python server.py` manually — if it crashes, Claude can’t start it either
- Make sure the command is reachable from Claude (absolute path!)

**4. Meetings “conflict” error**  
- `MeetingManager.schedule_meeting(...)` raises if the same datetime exists — that’s by design.

---

## Notes & Limitations

- **In-memory only**: restarting the server will **lose data**.
- **No real HRMS / DB**: this is a demo agentic backend.
- **Email module is external**: you must provide it.
- **MCP transport**: currently `stdio` (what Claude Desktop expects).
- **Security**: creds are passed via env → for real prod usage, use a secret manager.

---

## License

This repository did not specify a license in the provided files.  
If you plan to use it commercially or publish it, **add a `LICENSE` file** (MIT / Apache-2.0 / etc.).

---

## Quick Start (TL;DR)

1. `git clone ...`
2. `cd rohan0497-hr-assist`
3. create `.env` with:
   ```env
   CB_EMAIL=you@example.com
   CB_EMAIL_PWD=app_password
   ```
4. `pip install "mcp[cli]>=1.19.0" python-dotenv`  
5. `python server.py`
6. add MCP entry in Claude
7. in Claude → “Onboard a new employee” ✅

You now have a working **agentic HR assistant**.
