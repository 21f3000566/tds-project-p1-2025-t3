# tds-project-p1-2025-t3

# 🚀 Automated Task Receiver & Deployment Service

This repository hosts a **FastAPI-based backend** that automates the following pipeline:

- ✅ Receives AI task submissions from an evaluation server  
- 🤖 Generates or updates code using **Google Gemini**  
- 📂 Saves generated code and attachments locally  
- 🧬 Initializes or updates a **GitHub repository**  
- 🌐 Publishes a live site automatically via **GitHub Pages**  
- 📢 Notifies the evaluation server with deployment results

It’s fully automated, concurrent, and designed to run on platforms like **Hugging Face Spaces**, **Render**, or **Vercel**.

---

## 📦 Features

- 🔐 **Secure Webhook** endpoint with secret verification  
- 🤖 Uses **Gemini API** to generate HTML/CSS/JS web apps  
- 📁 Automatic GitHub repo creation or cloning  
- 🌍 One-click deployment to **GitHub Pages**  
- 🧪 Notifies a remote evaluation server of success/failure  
- 📑 Built-in logging and health/status endpoints  
- 🔄 Supports **round-based updates** (initial generation + surgical updates)  
- 📎 Handles **attachments** (e.g., images) and integrates them into HTML

---

## 🧰 Tech Stack

| Component        | Description |
|------------------|-------------|
| [FastAPI](https://fastapi.tiangolo.com/) | REST API framework |
| [httpx](https://www.python-httpx.org/)   | Async HTTP client |
| [GitPython](https://gitpython.readthedocs.io/) | Git operations |
| [Pydantic](https://docs.pydantic.dev/)   | Request validation |
| [Gemini API](https://ai.google.dev/)     | LLM for code generation |
| GitHub API       | Repo & Pages automation |

---

## ⚙️ Environment Variables

Create a `.env` file in the root directory with the following:

```env
# Google Gemini API key
GEMINI_API_KEY=your_gemini_api_key

# GitHub credentials
GITHUB_TOKEN=your_github_personal_access_token
GITHUB_USERNAME=your_github_username

# Secret for /ready endpoint authentication
STUDENT_SECRET=super_secret_value

# Optional: GitHub Pages base URL (defaults to https://<GITHUB_USERNAME>.github.io)
GITHUB_PAGES_BASE=https://<your-username>.github.io

# Optional: service settings
LOG_FILE_PATH=logs/app.log
MAX_CONCURRENT_TASKS=2
KEEP_ALIVE_INTERVAL_SECONDS=30
