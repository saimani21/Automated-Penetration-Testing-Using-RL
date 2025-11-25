# Automated Penetration Testing Using Reinforcement Learning

## 1. Project Overview

This project implements an **RL-based agent** that autonomously discovers and exploits vulnerabilities by **navigating web applications and testing attack vectors**.

In this initial phase, the focus is on a **single, real vulnerable web application**:

- Target: **DVWA (Damn Vulnerable Web Application)** running in Docker
- Vulnerability: **SQL Injection (SQLi)**
- Security Level: **Low**

The agent uses **Reinforcement Learning (RL)** to learn which SQL injection payloads successfully exploit DVWA and how to do it in the **fewest possible requests** — mimicking an automated penetration tester.

Future phases will extend this to other vulnerabilities (e.g. XSS, Command Injection), but this README covers **Phase 1–5**, where **SQLi is fully implemented and working**.

---

## 2. Goals

- Build a **practical RL environment** around a real vulnerable web app (DVWA).
- Implement an **RL agent (DQN)** that:
  - Tries different SQLi payloads,
  - Observes the HTTP responses,
  - Receives rewards based on success/failure,
  - Learns which payloads reliably exploit SQL injection.
- Compare the trained agent’s performance against a **random attacker**.
- Provide a small “tool-style” script that uses the trained model to **automatically exploit SQLi** in DVWA.

---

## 3. System Architecture (Phase 1–5)

### High-Level Concept

The long-term project vision is a **multi-vulnerability orchestrator**:





```text
Automated Penetration Testing Using RL
    ├── SQL Injection Module (this phase ✅)
    ├── XSS Module (planned)
    ├── Command Injection Module (planned)
    └── ... (future modules)





4. Project Structure
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



5. Setup (Phase 1 – Environment)
5.1 Requirements

OS: Windows / Linux / macOS

Python: 3.11+

Docker Desktop / Docker Engine

Git

Recommended IDE: VS Code

5.2 Python Virtual Environment
Install dependencies
