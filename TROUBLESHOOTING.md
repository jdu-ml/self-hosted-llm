# Self-Hosted LLM - Troubleshooting Log
Debugging history for the self-hosted LLM stack. Kept intentionally detailed since the diagnostic process is as much a part of this project as the working result.


## 1. Podman Port Mapping Failure (Windows to VM)
**TL;DR:** Windows resolves `localhost` to IPv6 (`::1`), but Podman's rootless port-forwarding binds to IPv4. Fixed with a `netsh portproxy` v4-to-v4 rule and using `127.0.0.1` instead of `localhost`.

**Symptom:** Podman port mapping (`-p 3500:8080` or similar) between Windows and the Podman WSL2 VM fails consistently. `curl`/`Invoke-WebRequest` establishes a TCP connection, sends the request, and then receives `Recv failure: Connection aborted`.

**Diagnosis (ruled out one by one):**
- From inside the VM, `podman exec open-webui curl localhost:8080` worked, confirming the Open WebUI container was healthy and ruling out a crashed or broken app as the cause
- From Windows, reproduced the failing request while watching `podman logs -f open-webui` running in the VM. The failing request did not appear in the logs, meaning it never reached the container and the connection was dying somewhere earlier in the chain
- Disabled AVG Web Shield. No change
- Added Windows Firewall inbound allow rule for `podman.exe` on the Public profile. No change
- No VPN installed
- No proxy configured
- From Windows, hitting the VM's internal IP directly failed differently ("unable to connect"), ruling out a simple host to VM routing fix
- Updated WSL to 2.7.10.0, Podman to 5.8.3, and Podman Desktop to 1.28.3. No change

**Working theory:** Podman-on-Windows/WSL2 rootless port-forwarding bug

**Issue identified when testing port mapping: `-p 3500:8080`**
- From inside the WSL2 VM (`podman machine ssh` → `curl localhost:3500`): Worked
- From Windows (`curl.exe -v http://localhost:3500`): Failed, `Recv failure: Connection was aborted`

This showed the break was specifically in the Windows to VM hop, not Podman's internal setup. `curl.exe -v` showed it was resolving `localhost` and trying `[::1]` (IPv6 loopback), not `127.0.0.1` (IPv4 loopback).
```text
Trying [::1]:3500...
Recv failure: Connection was aborted
```

**Podman's rootless port-forwarding was binding to IPv4. As a result, every browser request to `http://localhost:3500` was silently hitting a dead IPv6 listener and failing.**

Tested IPv4
```powershell
curl.exe -4 -v http://localhost:3500     # works
curl.exe -v http://127.0.0.1:3500        # works
curl.exe -v http://localhost:3500        # fails (tries ::1 first)
```

**Solution:** Created port proxy via `netsh portproxy` for IPv4 so Windows listens on the port and forwards at the OS network layer directly to the VM's IP.

1. Got Podman VM's IP (used the `inet` line)
    ```powershell
    wsl -d podman-machine-default -- ip addr show eth0
    ```

2. Created port proxy and added firewall rule
    ```powershell
    netsh interface portproxy add v4tov4 listenport=3500 listenaddress=0.0.0.0 connectport=3500 connectaddress=<VM_IP>

    New-NetFirewallRule -DisplayName "Podman 3500 Portproxy" -Direction Inbound -LocalPort 3500 -Protocol TCP -Action Allow
    ```

**Outcome:** `http://127.0.0.1:3500` works. `http://localhost:3500` still fails because Windows prefers IPv6 by default and nothing serves `[::1]:3500`.

**Attempted fix (unresolved): IPv6 via `v6tov4` rule**

Created port proxy
```powershell
netsh interface portproxy add v6tov4 listenport=3500 listenaddress=::1 connectport=3500 connectaddress=<VM_IP>
```

This adds an IPv6-listening rule that should translate to the IPv4 VM address. At the time, confirmed the rule registered correctly and a real listener existed on `[::1]:3500`, but the connection still aborted.

**Working theory:** Assumed `netsh portproxy v6tov4` is more fragile than `v4tov4` due to address-family translation, layered on top of a WSL2 Hyper-V virtual switch interface. Did not chase this further.

**Decision:** Use `http://127.0.0.1:3500` going forward, not `localhost`.

**If revisited later:** Deeper IPv6 diagnosis.


## 2. Open WebUI and Ollama Connection Error (VM to Windows)
**TL;DR:** VM to Windows traffic is routed through NAT on Podman's virtual switch, so standard Windows Firewall inbound rules do not apply at that layer. Rather than patch around it, containerized Ollama itself so both services talk over a shared Podman network via container-name DNS.

The port proxy fix above solves the Windows to VM direction (something on Windows reaching into the container/VM). This problem is the opposite direction: Open WebUI (running inside the Podman WSL2 VM) needs to reach out to Ollama (running natively on Windows). Portproxy has no "VM listens, forwards to Windows" equivalent.

**Symptom:** Open WebUI's Admin Settings → Connections → Verify Ollama connection returns an error.

**Diagnosis:**
- Discovered Ollama was not listening on all interfaces by default. It was bound to `127.0.0.1:11434`. Added `setx OLLAMA_HOST "0.0.0.0:11434"`. Confirmed `0.0.0.0:11434 LISTENING` via `netstat -ano | findstr 11434`
- Confirmed Ollama was healthy via `curl.exe -4 -v http://127.0.0.1:11434` from Windows. Returned `Ollama is running`
- Got Podman VM's gateway IP (`podman machine ssh` → `ip route | grep default`). Tried this as the connection URL in Open WebUI (`http://<GATEWAY_IP>:11434`). Hung on "Trying..."
- From inside the VM, `curl -v http://<GATEWAY_IP>:11434` hangs at the TCP handshake stage every time, confirming the block is somewhere in the VM to Windows network path, not in Ollama or Open WebUI themselves
- Added Windows Firewall inbound allow rule for port 11434 on all profiles. No change
- Disabled AVG. No change
- Attempted to reclassify the `vEthernet (WSL (Hyper-V firewall))` adapter from Public to Private via `Set-NetConnectionProfile`. Failed since this adapter does not register as a normal `MSFT_NetConnectionProfile` object (it is a special Hyper-V firewall-filtered adapter, not a standard profile-bound NIC)

**Issue identified via `Get-NetFirewallHyperVRule`:** The rule showed `EnforcementStatus: NATInboundRuleNotApplicable`. **This connection type is routed through NAT on Podman's virtual switch and standard Windows Firewall inbound rules (which assume host-level filtering) do not apply to NAT-translated traffic at this layer at all. This is why every firewall-rule attempt was fixing the wrong layer.**

According to the official [Open WebUI documentation](https://github.com/open-webui/open-webui#troubleshooting): "If you're experiencing connection issues, it's often due to the WebUI docker container not being able to reach the Ollama server at 127.0.0.1:11434 (host.docker.internal:11434) inside the container. Use the `--network=host` flag in your docker command to resolve this. Note that the port changes from 3000 to 8080, resulting in the link: `http://localhost:8080`."

Despite it being the officially documented fix, chose not to implement `--network=host`. Leaning toward containerizing Ollama as the cleaner long-term answer.

**Solution:** Containerize Ollama. Moved Ollama into its own Podman container and uninstalled Ollama on Windows. The Ollama container is on the same network as the Open WebUI container, so they can communicate with each other.

1. Create shared network
    ```powershell
    podman network create llm-net
    ```

2. Created fresh Open WebUI container (old volume wiped for a clean slate)
    ```powershell
    podman run -d `
      --network=llm-net `
      --name open-webui `
      -v open-webui:/app/backend/data `
      -p 3500:8080 `
      ghcr.io/open-webui/open-webui:main
    ```

3. Created Ollama container and pulled a small model
    ```powershell
    podman run -d `
      --network=llm-net `
      --name ollama `
      -v ollama-data:/root/.ollama `
      -p 11434:11434 `
      ollama/ollama

    podman exec -it ollama ollama pull gemma2:2b
    ```

4. Verified container-to-container connectivity
    ```powershell
    podman exec open-webui curl http://ollama:11434
    ```

5. In the UI, since `OLLAMA_BASE_URL` was not set during container creation, connected manually via Admin Settings → Connections → Ollama API URL (set to `http://ollama:11434`). Confirmed working, `gemma2:2b` shows up and responds in chat.

**Outcome:** No more connection issues between Open WebUI and Ollama. The port proxy fix is still in place for Windows to VM.

**Separate issue found during solution setup:** `gemma2:2b` does not support tool calling. The model's settings in Open WebUI had Builtin Tools checked by default, which caused an immediate error of "does not support tools" on any message. The solution was to uncheck all the tools.


## 3. RAM / Memory Management
**TL;DR:** WSL2 has no memory ceiling by default and `podman machine set --memory` is not supported on the WSL provider. Fixed by setting a memory limit directly in `.wslconfig`.

**Symptom:** Podman Desktop showed the VM at ~99% memory. In addition, Windows Task Manager showed VM (`vmmem`) climbing to ~88% of 16GB total system RAM.

**Root cause:** WSL2 has no memory ceiling by default. The VM can dynamically claim RAM up to the full host amount.

`podman machine set --memory` is not supported on the WSL provider (`Error: changing memory not supported for WSL machines`). Podman machine does not manage its own separate allocation on WSL. It inherits whatever WSL2 itself is configured with. The `2GiB` shown in `podman machine list` appears to be a stale display value, not an enforced ceiling.

**Solution:** Set memory in `.wslconfig`.
```ini
# C:\Users\<username>\.wslconfig
[wsl2]
memory=6GB
```

Chose 6GB out of 16GB total. This should be enough headroom for the gemma model plus both containers, while leaving ~10GB for Windows and normal apps.

**Outcome:**
- `vmmem` in Task Manager plateaus around ~6GB under load instead of climbing toward full system RAM
- `podman stats` shows actual per-container usage well within the new ceiling:
  ```text
  open-webui   850MB / 6.213GB   (13.68%)
  ollama       2.482GB / 6.213GB (39.94%)
  ```

  Combined ~3.3GB actually used, which is comfortable headroom under the 6GB cap.
