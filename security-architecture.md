This technical document provides a detailed, production-grade architectural review of the security hardening, systems engineering, and network optimization implemented on your Ubuntu Server 24.04 LTS VPS [1, 2].

---

## Architectural Overview: Path B

The system is configured in **Path B** [1]. This architecture is designed to serve a public-facing, sandboxed application directly on standard port **`22`** for frictionless developer access (`ssh angelodavales.info`), while completely isolating and hardening administrative operating system access on port **`43829`** [2, 3].

```
                                  [ INTERNET ]
                                       |
                +----------------------+----------------------+
                |                                             |
                | (Standard SSH Connection)                   | (Hardened Admin Connection)
                v                                             v
        [ Host Port 22 ]                               [ Host Port 43829 ]
                |                                             |
                v (nftables: prerouting -190)                 v (nftables: prerouting -190)
         [ f2b-table: drop ]                           [ f2b-table: drop ]
                |                                             |
                v (Docker Bridge NAT)                         v
         [ Forward Hook ]                             [ Host INPUT Hook ]
                |                                             |
                v                                             v
    [ him-gateway-1 Container ]                     [ Host OS OpenSSH ]
     - Managed-code SSH Server                       - Key-Only Auth (No Passwords) [2]
     - Restricted command sandbox                    - Monitored by Fail2Ban [2]
                |
                v (Private Bridge network)
       [ him-ai Container ]
        - Bound to port 8080
        - Completely invisible to public internet
```

---

## Component 1: Network Layer Architecture & The Netfilter Bypass

### The Vulnerability: Docker Bridging Bypassing the Host `INPUT` Chain
During initial testing in Bridged Mode, we observed **Connection Slot Starvation** [1]. The `.NET` application connection pool was permanently saturated (`19/20` active connections), locking out legitimate users despite active Fail2Ban blocks [1].

#### The Root Cause:
Docker utilizes Destination Network Address Translation (DNAT) inside the Netfilter `prerouting` hook (at priority `-100`) to route public port `22` traffic to the private bridge network [1]. Once a packet’s destination IP is rewritten to the container IP (e.g., `172.18.0.2`), the routing engine identifies the packet as routed, bypassing the host’s local **`input`** hook entirely. Instead, the packets traverse the **`forward`** hook [1].

Fail2Ban's default `nftables` action template (`nftables-allports`) hooks its blocklist (`f2b-chain`) into the host's **`input`** hook [2]. Consequently, banned bot traffic bypassed the Fail2Ban blocklist entirely, hit the Docker proxy, and successfully connected to the `.NET` application [1, 2].

```
Packet -> [PREROUTING] -> [Docker DNAT] -> [Routing Decision: Forward] -> [FORWARD] -> Bypasses [INPUT (where Fail2Ban was)]
```

### The Resolution: Kernel-Layer Prerouting Interception
To stop banned bots before Docker can NAT and forward their packets, we forced Fail2Ban to execute its drop rules at the earliest possible stage: the **`prerouting`** hook [1, 2].

We updated your `/etc/fail2ban/jail.d/him-gateway.local` configuration to override the default Netfilter registration parameters [2]:
```ini
banaction = nftables-allports[blocktype=drop, chain_hook=prerouting, chain_priority=-190]
```

*   **`chain_hook=prerouting`:** Instructs Fail2Ban to register its `f2b-chain` in the `prerouting` hook.
*   **`chain_priority=-190`:** Places the Fail2Ban blocklist extremely early in the packet traversal path, executing prior to connection tracking (`-200`) and Docker's NAT translation (`-100`).

Banned packets are now silently dropped (`drop`) at Layer 3/4 before they can ever trigger a socket allocation on the host or inside the container [1, 2]. Following this change, active connections on your gateway immediately dropped to a healthy `1/20`.

---

## Component 2: Cryptographic Identity & Volume-Mount Hardening

### The Vulnerability: Transient Host Keys & Host Identification Warnings
On every continuous delivery deployment, clients connecting from Windows received a severe security warning: `REMOTE HOST IDENTIFICATION HAS CHANGED` [3].

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

#### The Root Cause:
1.  **Host Directory Permissions Block:** Docker automatically created the host-mapped folder `./keys` under `root:root` ownership [1]. Because your hardened `.NET` application runs as a secure non-root user (specifically `app` with UID `1654` on .NET 8/10), the application was met with `Permission Denied` when trying to save its newly generated host keys [1]. It fell back to generating a transient, in-memory host key that was lost whenever the container restarted.
2.  **The `.NET` Zero-Byte File Trap:** In C# (`.NET`), calling `File.Exists("/app/hostkey.pem")` evaluates to `true` even if the file exists as an empty placeholder (0 bytes) [1]. When we pre-created `hostkey.pem` on the host, the application saw that the file existed, skipped its cryptographic generation block, failed to load the empty file, and silently defaulted to an in-memory key, keeping the host file empty [1].

### The Resolution: Non-Root Bind-Mount Bootstrapping
We executed a precise bootstrapping sequence to resolve the ownership boundaries and the 0-byte file trap:

1.  **De-allocation:** We temporarily commented out the volume mount in `docker-compose.yml` and recreated the container.
2.  **Internal Generation:** Operating with no files at `/app/hostkey.pem`, the C# application's `File.Exists` check returned `false` [1]. It successfully generated its own internal PEM-formatted RSA key block and wrote it to `/app/hostkey.pem` inside the container [1].
3.  **Extraction:** We pulled this valid, cryptographically sound key from the container namespace to the host VPS disk using `docker cp` [1]:
    ```bash
    sudo docker cp him-portfolio-him-gateway-1:/app/hostkey.pem ~/him-portfolio/keys/hostkey.pem
    ```
4.  **Ownership Alignment:** We reset the ownership of the host directory and key file to `root:root` (since your container process runs as root in this specific image namespace) and set appropriate read-write permissions [1]:
    ```bash
    sudo chown -R root:root ~/him-portfolio/keys
    sudo chmod 755 ~/him-portfolio/keys
    sudo chmod 666 ~/him-portfolio/keys/hostkey.pem
    ```
5.  **Direct File Bind-Mount:** We re-enabled the volume mount in `docker-compose.yml` to bind the specific file directly:
    ```yaml
    volumes:
      - ./keys/hostkey.pem:/app/hostkey.pem
    ```

Because the server now mounts a valid, persistent host key file from the VPS hard drive on startup, the cryptographic fingerprint of your gateway remains permanent across all future image updates [1, 3].

---

## Component 3: Inter-Container Microservice Communication

### The Vulnerability: Loopback Address Namespace Isolation
After transitioning your application from Host Network Mode (`network_mode: "host"`) to Bridged Network Mode for container-level security, the gateway threw a severe exception:
```text
System.Net.Http.HttpRequestException: Connection refused (127.0.0.1:8080)
```

#### The Root Cause:
Under Host Network Mode, the container shared the host’s network stack directly, allowing it to reach the AI service on the host's loopback address (`127.0.0.1:8080`) [1]. 

Once we shifted to Bridged Mode, the container received its own isolated network loopback namespace [1]. Inside the gateway container, `127.0.0.1` pointed to the *gateway container itself*, where no service was listening on port `8080` [1].

### The Resolution: Docker Internal Service Discovery
To resolve this, we configured your `.NET` microservices to communicate using Docker's built-in **Embedded DNS Server** [1].

1.  We updated your production `docker-compose.yml` environment variables [1]:
    ```yaml
    - AiServiceSettings__BaseUrl=http://him-ai:8080
    ```
2.  Because both containers run within the same default bridge network, Docker automatically resolves the service name `him-ai` to its dynamic internal container IP (e.g., `172.18.0.x`) [1]. 

Your gateway now communicates with the AI microservice over the secure, isolated virtual bridge network, successfully returning `200 OK` responses [1].

---

## Component 4: Proactive Host Hardening & Log-Based Intrusion Prevention

### The Strategy: Immediate Zero-Day Rejections
Port `22` is exposed to continuous automated scanning. Rather than allowing bots to brute-force usernames or spam connections, we implemented two proactive countermeasures:

#### 1. Banning immediately on `Type: exec`
*   **The Heuristic:** Legitimate users visiting your portfolio connect using an interactive terminal session (which requests a `pty` and `shell` channel) [3]. Automated scanners, however, explicitly request an **`exec`** channel to run silent commands instantly (e.g., trying to run malware payloads) [1.1.3].
*   **The Control:** We updated your C# logging statement to write the client's IP on the rejection log line:
    ```text
    [Security] ... | REJECTED request | IP: 176.65.139.251 | Type: exec | User: xem
    ```
    We then wrote a custom Fail2Ban filter to immediately parse this line. Any host requesting an `exec` channel is **banned instantly on their very first packet** for 24 hours [1.1.3, 1.2.5].

#### 2. Banning immediately on Handshake Failures
*   **The Heuristic:** Legitimate clients naturally support standard modern RSA host keys (`rsa-sha2-256` / `rsa-sha2-512`). Legacy botnets and exploit scanners frequently use outdated libraries that only support deprecated ciphers, causing a `KeyExchangeFailed` error [1].
*   **The Control:** We added a second regex pattern to your Fail2Ban filter to catch handshake errors [1.1.3]:
    ```text
    Connection error | IP: 172.236.228.227 | Session closed... Reason: KeyExchangeFailed
    ```
    Any scanner attempting to negotiate insecure algorithms is instantly blocked at Layer 3/4 before they can attempt further handshakes [1].

---

## Component 5: Local Client Workstation Optimization

### The Vulnerability: Windows OpenSSH Permission Checks
On your local Windows machine, attempts to use the administrative alias `ssh vps-admin` were blocked with:
```text
Bad owner or permissions on C:\Users\angel/.ssh/config
```

#### The Root Cause:
The Windows OpenSSH client enforces strict file-system permissions. Because the `config` file was created in your user directory, it automatically inherited permissions from the parent folder, allowing other local user accounts (like `DESKTOP-SP3HC83\test`) read access to the file [1.2.3, 1.2.6]. OpenSSH identifies this as an insecure configuration and rejects the file [1.2.3].

### The Resolution: ACL Inheritance Stripping
We used the Windows command-line Access Control list utility (**`icacls`**) to strip away inherited access and grant exclusive read/write rights to your active Windows identity [1.1.6, 1.2.3]:

```cmd
# Disable inheritance and remove other users (like test)
icacls C:\Users\angel\.ssh\config /inheritance:r

# Grant exclusive read/write permissions to your active account
icacls C:\Users\angel\.ssh\config /grant:r "%username%:(R,W)"
```

This successfully resolved the client-side validation errors, allowing you to connect using your customized administrative aliases [1.2.3].

---

## Verification and Monitoring Commands Reference

### 1. View Active Sockets and Connections
```bash
# Verify which processes are actively holding your public ports
sudo ss -tanp '( sport = :22 or sport = :43829 )'
```

### 2. View Active Kernel-Layer Blocks
```bash
# View automated, short-term bans (Fail2Ban)
sudo nft list table inet f2b-table

# View persistent, manual administrative bans
sudo nft list set inet host_firewall app_banlist_v4

# View your rate-limiting watchlists
sudo nft list set inet host_firewall app_ssh_flood_v4
```

### 3. Check Intrusion Prevention States
```bash
# Check the status of your gateway jail
sudo fail2ban-client status him-gateway

# Check the status of your repeat-offender jail
sudo fail2ban-client status recidive
```

### 4. Stream Live Drop Actions
```bash
# Watch the firewall silently drop attacker connections in real-time
sudo journalctl -k -f | grep "INPUT_BLOCKED"
```