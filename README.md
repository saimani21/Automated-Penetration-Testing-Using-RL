# Automated Penetration Testing Using Reinforcement Learning

## 1. Project Overview

This project implements an **RL-based agent** that autonomously discovers and exploits vulnerabilities by **navigating web applications and testing attack vectors**.

In this initial phase, the focus is on a **single, real vulnerable web application**:

- **Target:** DVWA (Damn Vulnerable Web Application) running in Docker
- **Vulnerability:** SQL Injection (SQLi)
- **Security Level:** Low

The agent uses **Reinforcement Learning (RL)** to learn which SQL injection payloads successfully exploit DVWA and how to do it in the **fewest possible requests** — mimicking an automated penetration tester.

Future phases will extend this to other vulnerabilities (e.g. XSS, Command Injection), but this README covers **Phase 1–5**, where **SQLi is fully implemented and working**.

---

## 2. Goals

- Build a **practical RL environment** around a real vulnerable web app (DVWA)
- Implement an **RL agent (DQN)** that:
  - Tries different SQLi payloads
  - Observes the HTTP responses
  - Receives rewards based on success/failure
  - Learns which payloads reliably exploit SQL injection
- Compare the trained agent's performance against a **random attacker**
- Provide a small "tool-style" script that uses the trained model to **automatically exploit SQLi** in DVWA

---

## 3. System Architecture (Phase 1–5)

### High-Level Concept

The long-term project vision is a **multi-vulnerability orchestrator**:

```
Automated Penetration Testing Using RL
    ├── SQL Injection Module (this phase ✅)
    ├── XSS Module (planned)
    ├── Command Injection Module (planned)
    └── ... (future modules)
```

Right now, only the SQL Injection module is implemented.

### SQLi Module Internals

```
+--------------------------+       +-----------------------------+
|  DVWAClient (HTTP API)   | <----> |  DVWA (Docker, SQLi page)  |
+--------------------------+       +-----------------------------+
              ^
              | used by
              v
+--------------------------+
|  DVWASQLiEnv (Gym Env)   |
+--------------------------+
              ^
              | used by
              v
+--------------------------+
|  DQN Agent (SB3 / PyTorch) |
+--------------------------+
```

**Components:**
- **DVWAClient:** handles login, CSRF tokens, setting security level, and sending SQLi requests
- **DVWASQLiEnv:** a custom gymnasium environment wrapping DVWA's SQLi page
- **DQN Agent:** trained using Stable-Baselines3 to interact with the environment and learn an effective exploitation strategy

---

## 4. Project Structure

```
rl-pentester/
│
├── env/
│   └── dvwa_sqli_env.py          # Gymnasium environment wrapping DVWA SQLi (hard-mode)
│
├── scripts/
│   ├── __init__.py
│   ├── dvwa_client.py            # DVWA HTTP client: login, set security, SQLi requests
│   ├── reward_utils.py           # SQLi payload library + success/error heuristics
│   ├── manual_dvwa_sqli.py       # Manual login + SQLi test (for verification)
│   ├── test_env_random.py        # Random-agent sanity check of the environment
│   └── use_trained_agent_once.py # Use trained model once to exploit SQLi
│
├── training/
│   ├── __init__.py
│   ├── train_dqn_sqli.py         # Train DQN agent on DVWASQLiEnv
│   └── eval_dqn_sqli.py          # Evaluate trained agent vs random policy
│
├── models/
│   └── dqn_dvwa_sqli_hard.zip    # Trained DQN model (created after training)
│
├── data/                         # (Optional) logs, CSVs, plots
│
├── venv/                         # Python virtual environment (not tracked)
│
└── README.md                     # This file
```

---

## 5. Setup (Phase 1 – Environment)

### 5.1 Requirements

- **OS:** Windows / Linux / macOS
- **Python:** 3.11+
- **Docker Desktop / Docker Engine**
- <img width="1427" height="807" alt="image" src="https://github.com/user-attachments/assets/1afc589f-fa77-4af1-a21c-066fe5c46de8" />

- **Git**
- **Recommended IDE:** VS Code

### 5.2 Python Virtual Environment

From the project root (`rl-pentester/`):


Install dependencies:



## 6. Setup (Phase 2 – DVWA + Manual SQLi)

### 6.1 Run DVWA in Docker

In any folder (e.g. `ml1/`):

```bash
git clone https://github.com/digininja/DVWA.git
cd DVWA
docker compose up -d
```

Check containers:

```bash
docker ps
```
<img width="1919" height="495" alt="image" src="https://github.com/user-attachments/assets/32543a0a-6239-4163-a962-a04d4ebeaa3e" />

You should see something like:

```
CONTAINER ID   IMAGE                           PORTS
...            ghcr.io/digininja/dvwa:latest   127.0.0.1:4280->80/tcp
```

→ DVWA is at `http://127.0.0.1:4280`

### 6.2 DVWA Initial Setup (in Browser)
<img width="1865" height="782" alt="image" src="https://github.com/user-attachments/assets/7765cc5a-2dcc-49b7-84cc-a9974f9c5dc3" />


1. Open `http://127.0.0.1:4280` in your browser
2. Click **Create / Reset Database**
3. **Login:**
   - Username: `admin`
   - Password: `password`
4. Go to **DVWA Security** → set **Security Level = Low** → Submit

### 6.3 Verify Manual SQLi Automation

**Script:** `scripts/manual_dvwa_sqli.py`

This script:
- Logs in using requests + CSRF token handling
- Sends:
  - a benign payload: `"1"`
  - a SQLi payload: `"' OR '1'='1"`
- Uses a heuristic to detect SQLi success: multiple occurrences of "First name" in the HTML

Example output:

```
[+] Logged in to DVWA
[*] Benign] length=4651 count('First name')=1 success=False
[*] SQLi  ] length=4934 count('First name')=5 success=True
[+] SQL Injection appears to be successful based on heuristic!
```

If you see this, your HTTP automation + success heuristic are working correctly.

---

## 7. Phase 3 – RL Environment Design (DVWASQLiEnv)

### 7.1 DVWAClient (`scripts/dvwa_client.py`)

Handles:
- `login()` — with CSRF token parsing
- `set_security_level("low")` — sets DVWA to Low via UI
- `sqli_request(payload)` — sends a GET request to DVWA SQLi page with `id=payload`

### 7.2 Reward & Payload Design (`scripts/reward_utils.py`)

We define a **hard-mode payload library**:
- **Total payloads:** 12 (stored in `SQLI_PAYLOADS`)
- **Only 2 are consistently effective SQLi payloads:**
  - `"' OR '1'='1"`
  - `"' OR '1'='1' -- "`
- **Others are:**
  - benign IDs (`"1"`, `"2"`, …)
  - invalid junk (`"abc"`, `"xyz"`)
  - conditions that return zero or single rows

**Success heuristic:**

```python
def is_sqli_success(html: str) -> bool:
    return html.count("First name") > 1
```

**Error heuristic:**

```python
def has_error_indicator(html: str) -> bool:
    # checks for common SQL error patterns in the HTML response
```

### 7.3 Gym Environment (`env/dvwa_sqli_env.py`)

Implements the `gymnasium.Env` API:

**Action space:**
```python
spaces.Discrete(len(SQLI_PAYLOADS))
```
→ agent chooses a payload index

**Observation space:** 4D vector:
- Normalized response length
- Success flag (0/1)
- Error flag (0/1)
- Normalized last action index

**Reward function (hard mode):**
- `+10` if SQLi success (multiple rows)
- `-0.5` each step (discourages brute-force)
- `-1` if SQL error indicator present
- `-5` additional penalty if the episode ends without success

**Episodes:**
- `max_steps = 3` → the agent gets at most 3 attempts per episode

**Sanity check:** `scripts/test_env_random.py`  
Runs random actions and prints step info so you can verify environment behaviour.

---

## 8. Phase 4 – Training the DQN Agent

**Script:** `training/train_dqn_sqli.py`

- Uses `DVWASQLiEnv`, wrapped with `Monitor` and `DummyVecEnv`
- **RL algorithm:** DQN (Deep Q-Network) from Stable-Baselines3

**Key hyperparameters:**
```python
learning_rate = 1e-3
buffer_size = 20_000
batch_size = 64
gamma = 0.99
exploration_fraction = 0.4
exploration_final_eps = 0.05
total_timesteps = 50_000
```

**Run training:**

```bash
python .\training\train_dqn_sqli.py
```

This creates:
```
models/dqn_dvwa_sqli_hard.zip
```

---
<img width="896" height="312" alt="image" src="https://github.com/user-attachments/assets/7f9811bc-06f9-4d95-b0b2-166e5cfdfb69" />


## 9. Phase 5 – Evaluation & Practical Use

### 9.1 Evaluation: Trained vs Random (`training/eval_dqn_sqli.py`)

This script:
- Evaluates the trained DQN agent over multiple episodes
- Evaluates a random policy under the same conditions
- Logs for each episode:
  - success (True/False)
  - number of steps
  - total reward

**Typical pattern on hard-mode:**

**Random policy example:**
- Success rate: ~40–50%
- Avg steps: ~2–3
- Avg reward: ≈ small positive or near zero (many failures, some successes)
- <img width="1425" height="240" alt="image" src="https://github.com/user-attachments/assets/d7e2310a-5923-405e-9e20-6f12c9355bf1" />


**Trained agent example:**
- Success rate: much higher (often close to 90–100%)
- Avg steps: close to 1
- Avg reward: significantly higher (because it avoids penalties and succeeds quickly)

**Run:**

```bash
python .\training\eval_dqn_sqli.py
```

Use these metrics in your project report to show clear learning and improvement over a random attacker.

### 9.2 Using the Trained Model as a Tool (`scripts/use_trained_agent_once.py`)

This script demonstrates how the trained RL agent can be used as an automated penetration testing helper:

1. Instantiates `DVWASQLiEnv`
2. Loads `models/dqn_dvwa_sqli_hard.zip`
3. Runs one deterministic episode
4. Prints:
   - chosen action index
   - exact payload string
   - whether SQLi succeeded
   - reward

**Example:**

```
[DVWAClient] Logged in
[DVWAClient] Security level set to low
[RUN] Using trained agent to exploit DVWA SQLi...
Step 1: action=11, payload="' OR '1'='1' -- ", success=True, reward=9.50

[RESULT] Agent successfully triggered SQL injection!
```
<img width="1363" height="250" alt="image" src="https://github.com/user-attachments/assets/ee1eb648-98ba-44c0-ae0e-b82974919d67" />


This shows that:
- The agent has learned which payloads work
- It can autonomously exploit the SQLi on DVWA with minimal requests

---

## 10. Limitations & Future Work

### Current Limitations (Phase 1–5)

- Only **SQL Injection** (Low security level) is implemented
- Only **DVWA** is used as a target (lab app)
- Success is detected via **HTML pattern matching** (count of "First name")
- No modelling of:
  - WAF evasion
  - stealth / IDS evasion
  - multi-step navigation (e.g. deeper workflows or chained attacks)

### Planned Future Enhancements

- Add RL modules for:
  - **XSS** (Cross-Site Scripting)
  - **Command Injection**
  - **File Upload / RFI / LFI** vulnerabilities (where applicable)
- Implement a true orchestrator script that:
  - logs in once
  - runs each module (SQLi, XSS, etc.)
  - prints a combined vulnerability report
- Try additional RL algorithms:
  - PPO, A2C, etc.
- Experiment with:
  - richer state representations (e.g. basic text features of responses)
  - more realistic penalty schemes

---

