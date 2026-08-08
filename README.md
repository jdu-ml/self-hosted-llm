# Self-Hosted LLM
A self-hosted LLM stack running entirely on local hardware, with no API keys, no cloud inference, and no data leaving your machine. Built with Podman, Ollama, and Open WebUI.

## Why
Most "run a local LLM" tutorials focus on a single container interacting with a natively-installed host app. This project goes further: it deploys two independent container services (a model server and a chat UI) connected over a dedicated, isolated network using container-name DNS for seamless communication. In production, applications are rarely a single container. They are typically made up of multiple independent services that communicate with each other, which is the pattern this project follows.

This project's debugging process is documented in detail in [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md). Self-hosted infrastructure work depends on systematic diagnosis with potential causes methodically validated and eliminated throughout the process.


## Stack
| Component | Role | Why |
|---|---|---|
| **Podman** | Container engine | Runs rootless by default. Containers run without root/administrator privileges on the host, limiting the blast radius if a container is ever compromised |
| **Ollama** | Model server | Serves models to any client (e.g., chat UI, terminal, custom scripts) |
| **Open WebUI** | Browser-based chat interface | Provides a familiar ChatGPT-style UI |

Both services run as **separate containers on a shared Podman network** (`llm-net`), communicating via container-name DNS (`http://ollama:11434`) rather than through the host machine.


## Architecture
```
┌───────────────────────────────────────────┐
│         Podman Network (llm-net)          │
│                                           │
│   ┌──────────┐         ┌──────────────┐   │
│   │  Ollama  │◄────────┤  Open WebUI  │   │
│   │  :11434  │         │    :8080     │   │
│   └──────────┘         └──────────────┘   │
│                                           │
└─────────────────────┬─────────────────────┘
                      │
               localhost:3500
                      │
                 ┌────▼────┐
                 │ Browser │
                 └─────────┘
```


## Getting Started
> **Note:** This project was built and documented on Windows 11 with WSL2. Commands and troubleshooting steps are Windows/WSL2-specific. Podman setup on macOS or Linux will differ.

**Prerequisites:** Podman + Podman Desktop + podman-compose (`pipx install podman-compose` recommended)

```powershell
# Clone the repo and cd into it
cd path\to\self-hosted-llm

# Start the Podman VM (Windows/WSL2 only)
podman machine start

# Create the shared network and volumes (first-time setup only)
podman network create llm-net
podman volume create ollama-data
podman volume create open-webui

# Start both containers
podman-compose up -d

# Pull a model inside the Ollama container (llama3.2 used as an example, may take a few minutes to download)
podman exec -it ollama ollama pull llama3.2
```

Navigate to `http://127.0.0.1:3500` in a web browser. When launching Open WebUI for the first time, create a local account. The first account created becomes the administrator account. After signing in, select a model and start chatting.

**Troubleshooting (Windows):** If `http://127.0.0.1:3500` does not load, this may be a Windows to VM networking issue. Check with:
```powershell
curl.exe -4 -v http://localhost:3500
```
If this works but `http://localhost:3500` (without `-4`) does not, see [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md#1-podman-port-mapping-failure-windows-to-vm) for the fix.


## Stopping the Stack
```powershell
# Stop both containers
podman-compose down

# Stop the Podman VM (Windows/WSL2 only)
podman machine stop
```


## Environment
- Windows 11
- Windows Subsystem for Linux (WSL) 2.7.10.0
- Podman 5.8.3 + Podman Desktop 1.28.3 + podman-compose 1.6.0
- Ollama (containerized, `docker.io/ollama/ollama`)
- Open WebUI (containerized, `ghcr.io/open-webui/open-webui:main`)


## Repo Structure
```
self-hosted-llm/
├── README.md            # this file
├── podman-compose.yml   # starts Ollama + Open WebUI together
├── NOTES.md             # concepts learned + commands
└── TROUBLESHOOTING.md   # full debugging log
```


## Future Opportunities
- Develop a Python script that interacts with Ollama's API to support practical local data tasks, such as log summarization. This would demonstrate Python and API integration built on top of the existing infrastructure
- Evaluate alternative vector databases for Open WebUI's RAG and document search capabilities to better understand the tradeoffs involved in moving beyond the default configuration
