# Self-Hosted LLM - Notes

## Project Planning
Build and document a self-hosted LLM stack as a portfolio project, demonstrating self-hosted infrastructure skills.

**Stack:**
- **Podman:** container engine (chosen over Docker for its rootless by default setting)
- **Ollama:** model server, runs as a container
- **Open WebUI:** browser-based chat interface, runs as a container

> **Note:** The original plan was to install Ollama natively on Windows rather than as a container. This was scratched after running into a connection problem between Open WebUI and native Ollama. See [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md#2-open-webui-and-ollama-connection-error-vm-to-windows) for the full story.


## Concepts Learned
- **Rootless containers**: containers run with root/admin privileges inside their own namespace but that root identity is mapped to an unprivileged user account outside the container (host). This reduces the potential blast radius by limiting the ability of a compromised container to gain true root access or full control over the host VM
- **Client/server architecture**: Ollama runs as a background server (`localhost:11434`). The terminal, Open WebUI, and any script are different clients talking to the same server
- **OpenAI-compatible API**: Ollama follows OpenAI's API format, allowing existing tools and libraries to interact with local models without requiring significant changes
- **Chat memory is an illusion**: Ollama itself is stateless per request. Applications like Open WebUI have a memory layer that stores conversation history in a database and includes relevant context with each request. This creates the appearance that the model "remembers" previous interactions


## Running the Stack
`podman-compose` must be run from the folder containing `podman-compose.yml`.

**Navigate to project folder**
```powershell
cd path\to\self-hosted-llm
```

**Start VM and containers**
```powershell
podman machine start
podman-compose up -d
```

**Stop containers and VM**
```powershell
podman-compose down
podman machine stop
```

**Quick reference**
| Action | Command |
|---|---|
| Go to project folder | `cd path\to\self-hosted-llm` |
| Start VM | `podman machine start` |
| Start containers | `podman-compose up -d` |
| Stop containers | `podman-compose down` |
| Stop VM | `podman machine stop` |
| Check status | `podman ps` |
| View logs | `podman-compose logs -f` |

**podman-compose file**
- `ollama-data`, `open-webui`, and `llm-net` are marked `external: true`. This tells compose to reuse the existing volumes and network from the earlier manual setup instead of creating fresh, empty ones. It preserves data and chat history, and avoids re-pulling models
- `OLLAMA_BASE_URL=http://ollama:11434` is set as an environment variable to avoid manual setup in the UI


## Next Steps
- [x] Look into port mapping issue (see [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md#1-podman-port-mapping-failure-windows-to-vm))
- [x] Look into Open WebUI not connecting to native-installed Ollama (see [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md#2-open-webui-and-ollama-connection-error-vm-to-windows))
- [x] Look into exploding memory (when containers are running) recorded in Podman Desktop and Windows Task Manager (see [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md#3-ram--memory-management))
- [x] Write `podman-compose.yml` to start both services with one command
- [x] Write `README.md`
