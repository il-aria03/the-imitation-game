# 🎮 The Imitation Game

> *"Can machines think?"* — Alan Turing, 1950

**The Imitation Game** is a web-based multiplayer game that brings the Turing Test to life as an interactive experience. Players take on the role of a **Judge** — trying to tell apart a human from an AI — or a **Participant** — trying to convince the judge they are human. Behind the scenes, a locally-hosted Large Language Model (LLM) generates questions, crafts answers, and plays as the silent opponent.

Built as a Bachelor's thesis project at **Sapienza University of Rome** (Computer and Automation Engineering, A.Y. 2024/2025), under the supervision of **Prof. Roberto Navigli**.

---

## 📑 Table of Contents

- [Overview](#overview)
- [Game Mechanics](#game-mechanics)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Database Design](#database-design)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Security](#security)
- [Future Developments](#future-developments)
- [Authors](#authors)

---

## Overview

The application revisits the classic Turing Test by adapting it into a gamified format that reduces the most common weaknesses of today's AI systems. Rather than long open-ended conversations — where AI tends to reveal its limitations — each session is kept short and focused: **three questions, three answers, one verdict**.

A random delay of up to 30 seconds is introduced before responses appear, simulating human typing behaviour and making it harder for the Judge to spot the machine.

This design philosophy mirrors real-world AI interaction patterns and gives the platform genuine research value, not just entertainment.

---

## Game Mechanics

### Roles

| Role | Description |
|---|---|
| **Participant** | Answers three questions posed by the Judge. The goal is to sound as human as possible. |
| **Judge** | Reads the three Q&A pairs and decides: human or machine? |

### Game Modes

**Classic Mode** (Judge only)
The Judge writes the questions and receives answers in real time, then delivers a verdict.

**Verdict Mode** (Judge only)
The Judge receives a complete session — three questions already answered — and must evaluate it. The session can come from a real Participant or be entirely AI-generated.

### Scoring — TuringCoin

The virtual currency *TuringCoin* rewards correct decisions and penalises mistakes, keeping the game balanced across all roles and modes.

| Event | Points |
|---|---|
| Judge guesses correctly | +3 |
| Judge guesses wrong | -2 |
| Participant is identified as human | +1 |
| Participant is identified as AI | 0 |

Score totals are protected at the database level: they can never drop below zero, enforced by SQL triggers.

---

## Architecture

The application follows an **n-tier RESTful architecture**, with clearly separated responsibilities across four independent services.

```
┌─────────────────────────────────────────────────────────┐
│                      CLIENT BROWSER                     │
└────────────────────────┬────────────────────────────────┘
                         │ HTTP (HTML/CSS/JS + Jinja2)
┌────────────────────────▼────────────────────────────────┐
│              FRONTEND SERVER  (FastAPI + Uvicorn)        │
│         Presentation layer — routing, templates          │
└────────────────────────┬────────────────────────────────┘
                         │ REST API (JSON)
┌────────────────────────▼────────────────────────────────┐
│              BACKEND SERVER  (FastAPI + Uvicorn)         │
│     Business logic — game flow, scoring, auth, AI        │
└──────────────┬──────────────────────────┬───────────────┘
               │ SQL                      │ HTTP
┌──────────────▼──────────┐  ┌───────────▼───────────────┐
│   MariaDB (RDBMS)        │  │  Ollama (Local LLM)        │
│  Users, Games, Stats     │  │  Question & answer gen.    │
└─────────────────────────┘  └───────────────────────────┘
```

All four services are orchestrated with **Docker Compose**, ensuring reproducible deployment in any environment with a single command.

---

## Tech Stack

### Backend
- **Python** — main programming language
- **FastAPI** — high-performance async REST API framework
- **Uvicorn** — ASGI server
- **asyncio** — concurrent task management (e.g. automatic session cleanup after 20 minutes of inactivity)
- **Pydantic** — strict data validation and serialisation across all endpoints
- **Passlib + bcrypt** — secure password hashing with automatic salting

### AI & Text Processing
- **Ollama** — runs the LLM locally, with no dependency on external cloud services
- **RapidFuzz** — fuzzy string matching to prevent duplicate or overly similar questions (similarity threshold: 85%)

### Database
- **MariaDB 11.7** — relational database with foreign key constraints, cascade policies, and custom SQL triggers

### Frontend
- **HTML5 / CSS / JavaScript** — standard web stack
- **Jinja2** — server-side templating engine
- Visual design inspired by **retro pixel art** from 1980s video games

### Infrastructure
- **Docker + Docker Compose** — full containerisation of all services

---

## Database Design

The schema is designed around five tables, each representing a core entity in the game.

```
Users ──< UserGames >── Games ──< Q_A
  │
  └──< Stats
```

| Table | Purpose |
|---|---|
| `Users` | Stores user credentials (username, email, hashed password) |
| `Stats` | Tracks cumulative statistics per user per role |
| `Games` | Records each game session and its status |
| `UserGames` | Associates users to games with their role, outcome, and points |
| `Q_A` | Stores question-answer pairs with flags for AI or human authorship |

### Key Constraints

- A game has exactly one Judge and one Participant (enforced via primary key on `game_id + player_role`).
- A user cannot hold two roles in the same game.
- Cascade deletion keeps the database clean when a user or game is removed.
- Two SQL `BEFORE` triggers prevent scores from becoming negative.

---

## Project Structure

```
.
├── backend/
│   └── src/
│       ├── backend.py          # FastAPI app initialisation
│       ├── config/             # Server config, constants, scoring values
│       ├── endpoints/
│       │   ├── auth/           # Registration and login
│       │   ├── game/           # Game lifecycle endpoints
│       │   └── user/           # Profile and statistics
│       ├── models/             # Pydantic models
│       └── utility/
│           ├── ai/             # Ollama integration
│           ├── db/             # Database connection management
│           ├── game/           # Game logic helpers
│           └── security/       # Password hashing and verification
│
├── frontend/
│   └── src/
│       ├── main.py             # FastAPI app initialisation
│       ├── config/             # Backend URL and frontend constants
│       ├── endpoints/          # HTTP calls to the backend
│       ├── models/             # Pydantic models (mirrored from backend)
│       ├── public/
│       │   ├── assets/
│       │   ├── css/
│       │   ├── js/
│       │   └── templates/      # Jinja2 HTML templates
│       └── utility/
│           ├── auth/
│           └── user/
│
├── database/
│   ├── mariadb_data/           # Docker volume (persistent data)
│   └── mariadb_init/
│       └── init.sql            # Schema definition and SQL triggers
│
├── Ollama/                     # Local LLM model storage
└── docker-compose.yaml
```

---

## Getting Started

### Prerequisites

- [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) installed on your machine.

### Installation

1. **Clone the repository**

```bash
git clone https://github.com/mattew-giambo/Turing-Test-Game.git
cd Turing-Test-Game
```

2. **Start all services**

```bash
docker compose up --build
```

Docker Compose will automatically start MariaDB first, wait for it to be healthy, then bring up Ollama, the backend, and finally the frontend — in the correct dependency order.

3. **Pull the LLM model** (first run only)

```bash
docker exec -it ollama_container ollama pull <model-name>
```

4. **Open the application**

Navigate to [http://localhost:8000](http://localhost:8000) in your browser.

### Service Ports

| Service | Port |
|---|---|
| Frontend | `8000` |
| Backend | `8003` |
| MariaDB | `3307` |
| Ollama | `11434` |

---

## Security

User passwords are never stored in plain text. The application uses **bcrypt** via the `Passlib` library, which automatically generates and applies a random salt before hashing. Verification at login consists of comparing the hash of the provided password against the stored hash — the original password is never recovered.

Input data at every endpoint is validated through **Pydantic models**, reducing the risk of malformed or malicious payloads reaching the database or the AI service.
