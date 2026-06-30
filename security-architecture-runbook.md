This production-grade systems engineering runbook documents the complete security hardening, integration architecture, and diagnostic commands executed on your DigitalOcean VPS and local Windows workstation [2].

---

## Section 1: Host Operating System & OpenSSH Hardening

### File 1: Host OpenSSH Daemon Configuration
*   **File Path:** `/etc/ssh/sshd_config` (VPS Host OS)
*   **Target Port:** `43829` (Administrative Access only) [2]
*   **Security Objective:** Isolate and restrict operating system-level administration [2].

```text
Port 43829
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key

PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no

MaxAuthTries 3
MaxStartups 10:30:100
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

AllowTcpForwarding no
X11Forwarding no
AllowAgentForwarding no
Compression no
TCPKeepAlive no

UsePAM yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server

HostKeyAlgorithms +ssh-rsa
PubkeyAcceptedKeyTypes +ssh-rsa
```

---

### Phase 1 Execution Commands: OpenSSH Validation & Application

During port migration, several structural system commands were executed on the VPS host to apply configurations and resolve system-level constraints:

#### Command 1: Resolve the Ubuntu 24.04 Privilege Separation Directory Bug
Because Ubuntu 24.04 LTS uses systemd socket activation (`ssh.socket`) by default, the background SSH service is not running, meaning systemd has not generated the `/run/sshd` in-memory directory [1.2.3, 1.2.9]. Running syntax tests (`sshd -t`) will crash unless this directory is manually bootstrapped [1.2.8, 1.2.9].

*   **Command:**
    ```bash
    sudo mkdir -p /run/sshd && sudo chmod 0755 /run/sshd
    ```
*   **Location:** VPS Host OS
*   **What it does:** Creates the system directory `/run/sshd` and sets directory read/write/execute permissions to `755` [1.1.1].
*   **Why it exists:** Bypasses a known Ubuntu 24.04 OpenSSH socket-activation startup bug, satisfying the binary's security checks so that config validation tools can run successfully [1.2.9].
*   **Risks:** None. It merely sets up a transient runtime directory in virtual memory (tmpfs) [1.1.5, 1.2.9].
*   **Expected Output:** Exits with status `0` and outputs no text.
*   **Common Failure Modes:** Directory is overwritten or lost if the machine is hard-rebooted before OpenSSH is cleanly enabled as a permanent systemd service [1.2.9].
*   **Troubleshooting Guidance:** Re-run the command if you receive the "missing privilege separation directory" error after a system reboot [1.2.9].

#### Command 2: Test and Validate OpenSSH Syntax
Always test your configuration file syntactically before restarting the service to prevent administrative lockouts.

*   **Command:**
    ```bash
    sudo sshd -t
    ```
*   **Location:** VPS Host OS
*   **What it does:** Runs a syntax check on `/etc/ssh/sshd_config`.
*   **Why it exists:** Evaluates directives, values, and file structures to ensure the config compiles cleanly.
*   **Risks:** None. Performs a dry-run check.
*   **Expected Output:** Exits silently with status `0` if the file is syntactically sound.
*   **Common Failure Modes:** Outputs specific line-number syntax errors (e.g., `Unsupported option` or `Bad port`).
*   **Troubleshooting Guidance:** Open the file, correct the line number referenced in the error, and re-run the command.

#### Command 3: Apply and Restart OpenSSH
Once validated, restart the service to bind OpenSSH permanently to port `43829` [2].

*   **Command:**
    ```bash
    sudo systemctl restart ssh
    ```
*   **Location:** VPS Host OS
*   **What it does:** Restarts the background OpenSSH systemd service [5].
*   **Why it exists:** Terminates active configuration states and spawns new listen sockets on port `43829` [2].
*   **Risks:** High if port `43829` is blocked by a host-level firewall, or if key authentication is misconfigured, which will lock you out of remote SSH administration.
*   **Expected Output:** Exits with status `0`.
*   **Common Failure Modes:** Service fails to start, or starts but rejects incoming connections.
*   **Troubleshooting Guidance:** Do not close your active terminal session. Test the new port in a separate window: `ssh root@angelodavales.info -p 43829`.

---

### File 2: Kernel Network Security parameter Hardening
*   **File Path:** `/etc/sysctl.d/99-security-hardening.conf` (VPS Host OS)
*   **Security Objective:** Secure the TCP/IP network stack against SYN floods, resource exhaustion, and socket starvation.

```ini
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_max_syn_backlog = 4096
net.core.somaxconn = 4096
net.core.netdev_max_backlog = 2500
```

---

### Phase 2 Execution Commands: Applying Sysctl Parameters

#### Command 4: Reload Sysctl Parameters
Load the hardened network parameters into active kernel memory without rebooting.

*   **Command:**
    ```bash
    sudo sysctl --system
    ```
*   **Location:** VPS Host OS
*   **What it does:** Scans, parses, and loads sysctl configuration files from `/etc/sysctl.d/` and applies their values live.
*   **Why it exists:** Updates active kernel memory immediately.
*   **Risks:** None. The configured values are well within safe enterprise network bounds.
*   **Expected Output:** Prints a detailed execution trace showing the parameters being applied:
    ```text
    * Applying /etc/sysctl.d/99-security-hardening.conf ...
    net.ipv4.tcp_syncookies = 1
    net.ipv4.tcp_synack_retries = 2
    net.ipv4.tcp_max_syn_backlog = 4096
    net.core.somaxconn = 4096
    net.core.netdev_max_backlog = 2500
    ```
*   **Common Failure Modes:** Typing errors in parameters (e.g., `net.ipv4.tcp_syncookie` without the `s`) will cause a parsing failure.
*   **Troubleshooting Guidance:** Correct the typo in the drop-in file and execute `sysctl --system` again.

---

## Section 2: Production Container & Docker Hardening

### File 3: Production Docker Compose Configuration
*   **File Path:** `~/him-portfolio/docker-compose.yml` (VPS Host OS)
*   **Security Objective:** Enforce Bridged Network Mode isolation, establish internal service routing, and implement file-level cryptographic host-key persistence [1, 3].

```yaml
services:
  him-gateway:
    image: ghcr.io/m4ngk4n08/him-gateway:latest
    ports:
      - "22:22"
    environment:
      - SshSettings__Port=22
      - AiServiceSettings__BaseUrl=http://him-ai:8080
      - TERM=xterm-256color
      - DOTNET_SYSTEM_CONSOLE_ALLOW_ANSI_COLOR_REDIRECTION=true
      - LANG=en_US.UTF-8
      - LC_ALL=en_US.UTF-8
    volumes:
      - ./keys/hostkey.pem:/app/hostkey.pem
    depends_on:
      - him-ai
    restart: always

  him-ai:
    image: ghcr.io/m4ngk4n08/him-ai:latest
    restart: always
```

---

### Phase 3 Execution Commands: Resolving the 0-Byte C# File Mount Trap

During deployment, we discovered that `File.Exists("/app/hostkey.pem")` in C# returns `true` even if the file exists on the host disk as an empty (0-byte) placeholder created by a `touch` command [1]. This tricked the `.NET` application's key-loader into skipping key generation, keeping the file empty and falling back to temporary in-memory keys. 

To resolve the permission boundaries and the 0-byte file trap, the following sequence was executed:

#### Command 5: Down and Delete Containers
Ensure no active volume locks or network allocations exist.

*   **Command:**
    ```bash
    sudo docker compose down
    ```
*   **Location:** VPS Host OS (inside `~/him-portfolio`)
*   **What it does:** Stops and removes all running container namespaces, bridge network interfaces, and volume mounts associated with the compose file [5].
*   **Why it exists:** Releases in-memory locks on host directory paths so we can safely adjust permissions and files.
*   **Risks:** Temporary application downtime during termination.
*   **Expected Output:**
    ```text
    ✔ Container him-portfolio-him-gateway-1 Removed
    ✔ Container him-portfolio-him-ai-1      Removed
    ✔ Network him-portfolio_default         Removed
    ```

#### Command 6: Extract the Internally Generated Key (The Bootstrap)
We temporarily booted the container *without* the volume mount. The C# application's `File.Exists` check returned `false`, triggering it to generate its own internal PEM-formatted key and write it to `/app/hostkey.pem` inside the container [1]. We extracted this valid key file:

*   **Command:**
    ```bash
    sudo docker cp him-portfolio-him-gateway-1:/app/hostkey.pem ~/him-portfolio/keys/hostkey.pem
    ```
*   **Location:** VPS Host OS
*   **What it does:** Copies the successfully generated private key file directly out of the running container namespace and writes it permanently over the empty host placeholder file [1].
*   **Why it exists:** Feeds the container's cryptographically sound SSH host key back to the host filesystem so it can survive future deployments [1].
*   **Risks:** None. Read-write file-system execution.
*   **Expected Output:** Exits silently with status `0`.
*   **Common Failure Modes:** `No such container` error if the container name is typed incorrectly.
*   **Troubleshooting Guidance:** Run `sudo docker ps` to verify the active container name and repeat the copy.

#### Command 7: Set Host Key Ownership to Root
Align the filesystem permissions of the keys folder with the active process (which runs as root inside the container).

*   **Command:**
    ```bash
    sudo chown -R root:root ~/him-portfolio/keys && sudo chmod 755 ~/him-portfolio/keys && sudo chmod 666 ~/him-portfolio/keys/hostkey.pem
    ```
*   **Location:** VPS Host OS
*   **What it does:** Changes the owner of the `keys` directory and its files to `root` on the host VPS, sets directory read/write/execute permissions to `755`, and grants write permissions on the file itself [1].
*   **Why it exists:** Aligns the host's directory ownership with the containerized application process, preventing `Permission Denied` write errors [1].
*   **Expected Output:** Exits silently with status `0`.

#### Command 8: Recreate Containers with Force-Recreate
Recreate the container with the newly mapped persistent key.

*   **Command:**
    ```bash
    sudo docker compose up -d --force-recreate
    ```
*   **Location:** VPS Host OS (inside `~/him-portfolio`)
*   **What it does:** Builds and launches the container environment from scratch, forcing all volume mappings, network interfaces, and environment variables to recreate [5].
*   **Why it exists:** Guarantees that the new `- ./keys/hostkey.pem:/app/hostkey.pem` volume mount is applied [1].
*   **Expected Output:**
    ```text
    ✔ Network him-portfolio_default         Created
    ...
    ✔ Container him-portfolio-him-gateway-1 Started
    ```

---

### File 4: Docker Daemon Logging Configuration
*   **File Path:** `/etc/docker/daemon.json` (VPS Host OS)
*   **Security Objective:** Route container console logs directly into the systemd binary journal (`journald`) to enable host-level intrusion prevention [1].

```json
{
  "log-driver": "journald"
}
```

---

### Phase 4 Execution Commands: Enabling the `journald` Log Driver

By default, Docker logs everything to hidden JSON files on disk [1]. To allow Fail2Ban to read the `.NET` application security logs natively from `journald` [1], the following commands were executed:

#### Command 9: Write the Daemon JSON file
*   **Command:**
    ```bash
    sudo tee /etc/docker/daemon.json << 'EOF'
    {
      "log-driver": "journald"
    }
    EOF
    ```
*   **Location:** VPS Host OS
*   **What it does:** Writes the logging driver parameters cleanly to `/etc/docker/daemon.json`.
*   **Expected Output:** Displays the written lines in your terminal.

#### Command 10: Restart the Docker Service
Apply the new logging driver to the system daemon.

*   **Command:**
    ```bash
    sudo systemctl restart docker
    ```
*   **Location:** VPS Host OS
*   **What it does:** Restarts the docker daemon service.
*   **Why it exists:** Forces Docker to reload `/etc/docker/daemon.json` and change its default logging engine from `json-file` to `journald` [1].
*   **Risks:** Temporary container downtime during service reload.
*   **Expected Output:** Exits silently with status `0`.

#### Command 11: Verify Journald Log Streaming
Verify that Docker is successfully streaming your `.NET` C# console logs into the systemd journal [1].

*   **Command:**
    ```bash
    sudo journalctl CONTAINER_NAME=him-portfolio-him-gateway-1 -n 20 --no-pager
    ```
*   **Location:** VPS Host OS
*   **What it does:** Queries the systemd binary journal database for log entries generated by your specific container, printing the last 20 lines [1].
*   **Why it exists:** Proves the container-to-host logging bridge is operational before configuring Fail2Ban [1].
*   **Expected Output:** Displays the live C# logs, confirming success [1].

---

## Section 3: Firewall & Intrusion Prevention Configurations (VPS)

### File 5: Firewall Ruleset Configuration
*   **File Path:** `/etc/nftables.conf` (VPS Host OS)
*   **Target Hook:** `prerouting` (Priority `-150` Mangle), `input` (Priority `0` Filter) [1]
*   **Security Objective:** Enforce Layer 3/4 rate-limiting and connection-flood blocklists natively in the kernel, protecting container slots from exhaustion [1].

*(The complete `/etc/nftables.conf` file is documented in Part 1, File 4. Below are the execution commands used to apply and test it.)*

---

### Phase 5 Execution Commands: Deploying the Firewall

#### Command 12: Validate ruleset Syntax
Always test your firewall configuration before reloading it to prevent network lockouts.

*   **Command:**
    ```bash
    sudo nft -c -f /etc/nftables.conf
    ```
*   **Location:** VPS Host OS
*   **What it does:** Runs a syntax check on `/etc/nftables.conf`.
*   **Why it exists:** Catches syntax errors (such as missing brackets or bad keywords) before applying rules to the active running kernel.
*   **Risks:** None. Read-only syntax validation.
*   **Expected Output:** Exits silently with status `0` if the file is syntactically sound.
*   **Common Failure Modes:** Truncated lines or missing symbols (e.g., `unexpected update`) will throw compilation errors.
*   **Troubleshooting Guidance:** Correct the specific line referenced in the compilation trace and re-test.

#### Command 13: Apply ruleset to Kernel
*   **Command:**
    ```bash
    sudo systemctl restart nftables
    ```
*   **Location:** VPS Host OS
*   **What it does:** Applies `/etc/nftables.conf` live into running kernel memory.
*   **Expected Output:** Exits silently with status `0`.
*   **Common Failure Modes:** If your active admin IP is not whitelisted, your terminal session will freeze immediately.
*   **Troubleshooting Guidance:** Connect via the DigitalOcean console and run `systemctl stop nftables` to recover.

---

### File 6: Fail2Ban Custom Filter Configuration
*   **File Path:** `/etc/fail2ban/filter.d/him-gateway.conf` (VPS Host OS)
*   **Security Objective:** Translate application-layer console violations into matching patterns for IP extraction [1].

```ini
[Definition]
failregex = ^.*\[Security\].*RATE LIMIT \| IP: <HOST>
            ^.*\[Security\].*CONCURRENT LIMIT \| IP: <HOST>
            ^.*\[Security\].*BANNED IP \| Rejected: <HOST>
            ^.*\[Security\].*TARPIT \| IP: <HOST>
            ^.*\[Security\].*REJECTED request \| IP: <HOST> \| Type: exec
            ^.*\[Gateway\].*Connection error \| IP: <HOST> \|.*Reason: KeyExchangeFailed
```

---

### File 7: Fail2Ban Custom Gateway Jail Configuration
*   **File Path:** `/etc/fail2ban/jail.d/him-gateway.local` (VPS Host OS)
*   **Security Objective:** Direct Fail2Ban to monitor container logs via systemd and inject blocks directly into the kernel's `prerouting` hook [1, 2].

```ini
[him-gateway]
enabled  = true
port     = 22
backend  = systemd
journalmatch = CONTAINER_NAME=him-portfolio-him-gateway-1
filter   = him-gateway
banaction = nftables-allports[blocktype=drop, chain_hook=prerouting, chain_priority=-190]
maxretry = 1
findtime = 10m
bantime  = 24h
```

---

### File 8: Fail2Ban Custom Repeat Offender Jail Configuration
*   **File Path:** `/etc/fail2ban/jail.d/recidive.local` (VPS Host OS)
*   **Security Objective:** Automatically escalate persistent, repeat offenders to long-term bans [1].

```ini
[recidive]
enabled   = true
backend   = systemd
banaction = nftables-allports[blocktype=drop, chain_hook=prerouting, chain_priority=-190]
bantime   = 1w
findtime  = 5d
maxretry  = 3
```

---

### Phase 6 Execution Commands: Deploying & Verifying intrusion Prevention

#### Command 14: Write the custom Filter and Jail files
*   **Command:**
    ```bash
    sudo tee /etc/fail2ban/filter.d/him-gateway.conf << 'EOF'
    [Definition]
    failregex = ^.*\[Security\].*RATE LIMIT \| IP: <HOST>
                ^.*\[Security\].*CONCURRENT LIMIT \| IP: <HOST>
                ^.*\[Security\].*BANNED IP \| Rejected: <HOST>
                ^.*\[Security\].*TARPIT \| IP: <HOST>
                ^.*\[Security\].*REJECTED request \| IP: <HOST> \| Type: exec
                ^.*\[Gateway\].*Connection error \| IP: <HOST> \|.*Reason: KeyExchangeFailed
    EOF
    ```
*   **Location:** VPS Host OS
*   **What it does:** Writes the filter parameters cleanly to `/etc/fail2ban/filter.d/him-gateway.conf`.
*   **Expected Output:** Displays the written lines in your terminal.

#### Command 15: Validate Regex Against Simulated Log Line
Verify that your regular expressions correctly capture the IP address from a simulated log line.

*   **Command:**
    ```bash
    sudo fail2ban-regex "him-gateway-1 | [Security] 2026-06-29 12:43:25 UTC | REJECTED request | IP: 176.65.139.251 | Type: exec | User: xem" /etc/fail2ban/filter.d/him-gateway.conf
    ```
*   **Location:** VPS Host OS
*   **What it does:** Compiles your custom regex file and checks it against a mock log string containing the IP `176.65.139.251` [1.1.3].
*   **Why it exists:** Confirms the accuracy of your regex patterns before putting them live.
*   **Expected Output:**
    ```text
    Results
    =======
    Failregex: 1 total
    |-  #) [# of hits] regular expression
    |   5) [1] ^.*\[Security\].*REJECTED request \| IP: <HOST> \| Type: exec
    `-

    Lines: 1 lines, 0 ignored, 1 matched, 0 missed
    ```

#### Command 16: Validate Regex Against Live systemd Journal Database
*   **Command:**
    ```bash
    sudo fail2ban-regex --journalmatch="CONTAINER_NAME=him-portfolio-him-gateway-1" systemd-journal /etc/fail2ban/filter.d/him-gateway.conf
    ```
*   **Location:** VPS Host OS
*   **What it does:** Queries the live systemd journal database for your running container and applies the filter regex rules [1].
*   **Why it exists:** Confirms that the Fail2Ban system is actively communicating with and parsing your container's live log history [1].
*   **Expected Output:** Displays a summary of successfully matched and parsed historical log lines from the container [1].

#### Command 17: Apply and Restart Fail2Ban
*   **Command:**
    ```bash
    sudo systemctl restart fail2ban
    ```
*   **Location:** VPS Host OS
*   **What it does:** Restarts the Fail2Ban service, loading all new filters, jails, and actions [5].
*   **Expected Output:** Exits silently with status `0`.

---

## Section 4: Host Auditing & Monitoring Sockets

#### Command 18: Check Active Sockets and Connections
Monitor active connections on port 22 and port 43829.

*   **Command:**
    ```bash
    sudo ss -tanp '( sport = :22 or sport = :43829 )'
    ```
*   **Location:** VPS Host OS
*   **What it does:** Displays all active TCP sockets bound to port 22 and port 43829.
*   **Why it exists:** Verifies socket states (`LISTEN` or `ESTAB`) and tracks the IP addresses of connected clients.
*   **Expected Output:** Shows the `docker-proxy` process listening on port `22` and the `sshd` process listening on port `43829` [2].

#### Command 19: View Active Fail2Ban Jails
*   **Command:**
    ```bash
    sudo fail2ban-client status
    ```
*   **Location:** VPS Host OS
*   **What it does:** Queries the Fail2Ban daemon for all active, configured monitoring jails.
*   **Expected Output:**
    ```text
    Status
    |- Number of jail:      3
    `- Jail list:           him-gateway, recidive, sshd
    ```

---

## Section 5: Local Client Workstation Hardening (Windows)

### File 9: Local SSH Client Configuration
*   **File Path:** `C:\Users\angel\.ssh\config` (Local Windows Machine)
*   **Security Objective:** Establish administrative connection aliases and automate port routing [3].

```text
Host admin.angelodavales.info vps-admin
    HostName angelodavales.info
    Port 43829
    User root
```

---

### Phase 7 Execution Commands: Client Hardening (Windows Command Prompt)

Windows OpenSSH enforces strict file permission boundaries [1.2.3, 1.2.6]. It will reject your `config` file if other local user accounts (like `DESKTOP-SP3HC83\test`) have read access [1.2.3, 1.2.6].

#### Command 20: Strip File Permission Inheritance
*   **Command:**
    ```cmd
    icacls C:\Users\angel\.ssh\config /inheritance:r
    ```
*   **Location:** Local Windows Machine (Command Prompt)
*   **What it does:** Disables file permission inheritance on the `config` file, removing access for other local user profiles [1.1.6, 1.2.3].
*   **Why it exists:** Satisfies OpenSSH client security constraints so the local system will load the file [1.2.3].
*   **Expected Output:** `Successfully processed file: C:\Users\angel\.ssh\config` [1.1.6].

#### Command 21: Grant Exclusive Owner Access
*   **Command:**
    ```cmd
    icacls C:\Users\angel\.ssh\config /grant:r "%username%:(R,W)"
    ```
*   **Location:** Local Windows Machine (Command Prompt)
*   **What it does:** Grants exclusive read and write permissions to your active Windows user profile [1.1.6, 1.2.3].
*   **Expected Output:** `Successfully processed file: C:\Users\angel\.ssh\config` [1.1.6].

#### Command 22: Clear Windows Local Key Cache
When the server's identity signature is updated, clear the outdated host key cached on your local Windows client to prevent MITM warnings [3].

*   **Command:**
    ```cmd
    ssh-keygen -R angelodavales.info
    ```
*   **Location:** Local Windows Machine (Command Prompt)
*   **What it does:** Purges old cached fingerprints associated with `angelodavales.info` from your local `known_hosts` file [3].
*   **Expected Output:** Outputs a confirmation that the cached entries have been successfully updated.