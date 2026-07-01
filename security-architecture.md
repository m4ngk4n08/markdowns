# Securing and Optimizing an Ubuntu Server 24.04 LTS VPS

This guide documents the engineering choices, troubleshooting steps, and configurations used to establish a dual-purpose server environment: a public-facing developer sandbox alongside a heavily fortified administrative management layer [2, 3].

---

## Architectural Overview: Split-Port Strategy

The architecture is built to serve a public-facing, sandboxed application directly on the standard SSH port (**`22`**) for frictionless developer access (`ssh angelodavales.info`), while completely isolating and hardening administrative operating system access on port **`43829`** [2, 3].

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

## Component 1: The Docker Detour — Reclaiming the Host Firewall

### The Mystery of the Vanishing Ban
During initial testing in Bridged Mode, we observed **Connection Slot Starvation** [1]. The `.NET` application connection pool was permanently saturated (`19/20` active connections), locking out legitimate users despite active Fail2Ban blocks [1]. Banned bad actors were somehow continuing to connect [1].

### The Root Cause: DNAT Bypasses the `INPUT` Chain
Docker utilizes Destination Network Address Translation (DNAT) inside the Netfilter `prerouting` hook (at priority `-100`) to route public port `22` traffic to the private bridge network [1]. 

Once a packet’s destination IP is rewritten to the container IP (e.g., `172.18.0.2`), the routing engine identifies the packet as routed, bypassing the host’s local **`input`** hook entirely. Instead, the packets traverse the **`forward`** hook [1].

```
Packet -> [PREROUTING] -> [Docker DNAT] -> [Routing Decision: Forward] -> [FORWARD] -> Bypasses [INPUT]
                                                                                           ^
                                                                              (Where Fail2Ban default lived!)
```

Because Fail2Ban's default action template (`nftables-allports`) registers its blocklist (`f2b-chain`) in the host's **`input`** hook, the banned bot traffic bypassed the blocklist entirely, hit the Docker proxy, and successfully connected to the container [1, 2].

### The Resolution: Kernel-Layer Prerouting Interception
To stop banned bots before Docker can NAT and forward their packets, we shifted the Fail2Ban drop rules to the earliest possible stage: the **`prerouting`** hook [1, 2].

The `/etc/fail2ban/jail.d/him-gateway.local` configuration was updated to override the default Netfilter registration parameters [2]:

```ini
banaction = nftables-allports[blocktype=drop, chain_hook=prerouting, chain_priority=-190]
```

*   **`chain_hook=prerouting`:** Instructs Fail2Ban to register its `f2b-chain` in the `prerouting` hook instead of `input` [2].
*   **`chain_priority=-190`:** Places the Fail2Ban blocklist extremely early in the packet traversal path, executing prior to connection tracking (`-200`) and Docker's NAT translation (`-100`).

Banned packets are now silently dropped (`drop`) at Layer 3/4 before they can trigger a socket allocation on the host or inside the container [1, 2]. Following this change, active connections on the gateway immediately dropped to a healthy `1/20`.

---

## Component 2: Cryptographic Identity & The C# Zero-Byte Trap

### The Ghost Host Key
On every continuous delivery deployment, clients connecting from Windows received a severe security warning: `REMOTE HOST IDENTIFICATION HAS CHANGED` [3].

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

This occurred due to two distinct permission and code behavior issues:

1.  **Host Directory Permissions Block:** Docker automatically created the host-mapped folder `./keys` under `root:root` ownership [1]. Because the hardened `.NET` application runs as a secure non-root user (specifically `app` with UID `1654` on .NET 8/10), the application was met with `Permission Denied` when trying to save its newly generated host keys [1]. It fell back to generating a transient, in-memory host key that was lost whenever the container restarted.
2.  **The `.NET` Zero-Byte File Trap:** In C#, calling `File.Exists("/app/hostkey.pem")` evaluates to `true` even if the file exists as an empty placeholder (0 bytes) [1]. When we pre-created `hostkey.pem` on the host, the application saw that the file existed, skipped its cryptographic generation block, failed to load the empty file, and silently defaulted to an in-memory key, leaving the host file empty [1].

### The Resolution: Non-Root Bind-Mount Bootstrapping
We executed a precise bootstrapping sequence to resolve the ownership boundaries and the 0-byte file trap:

1.  **De-allocation:** We temporarily commented out the volume mount in `docker-compose.yml` and recreated the container.
2.  **Internal Generation:** Operating with no files at `/app/hostkey.pem`, the C# application's `File.Exists` check correctly returned `false` [1]. It generated its own internal PEM-formatted RSA key block and wrote it to `/app/hostkey.pem` inside the container [1].
3.  **Extraction:** We pulled this valid key from the container namespace to the host VPS disk using `docker cp` [1]:
    ```bash
    sudo docker cp him-portfolio-him-gateway-1:/app/hostkey.pem ~/him-portfolio/keys/hostkey.pem
    ```
4.  **Ownership Alignment:** We reset the ownership of the host directory and key file to `root:root` and set appropriate read-write permissions [1]:
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

## Component 3: The Loopback Illusion — Bridging the Microservice Gap

### The Loopback Address Namespace Isolation
After transitioning the application from Host Network Mode (`network_mode: "host"`) to Bridged Network Mode for container-level security, the gateway threw a connection exception:

```text
System.Net.Http.HttpRequestException: Connection refused (127.0.0.1:8080)
```

### The Root Cause: Localhost Isn't Local Anymore
Under Host Network Mode, the container shared the host’s network stack directly, allowing it to reach the AI service on the host's loopback address (`127.0.0.1:8080`) [1]. 

Once we shifted to Bridged Mode, the container received its own isolated network loopback namespace [1]. Inside the gateway container, `127.0.0.1` pointed to the *gateway container itself*, where no service was listening on port `8080` [1].

### The Resolution: Docker Internal Service Discovery
To resolve this, we configured the `.NET` microservices to communicate using Docker's built-in **Embedded DNS Server** [1].

1.  We updated the production `docker-compose.yml` environment variables [1]:
    ```yaml
    - AiServiceSettings__BaseUrl=http://him-ai:8080
    ```
2.  Because both containers run within the same default bridge network, Docker automatically resolves the service name `him-ai` to its dynamic internal container IP (e.g., `172.18.0.x`) [1]. 

Your gateway now communicates with the AI microservice over the secure, isolated virtual bridge network, successfully returning standard `200 OK` responses [1].

---

## Component 4: Active Application Hardening — Building a Self-Healing Connection Pool

Publicly exposed SSH ports (Port 22) are subjected to relentless, automated internet scanning. Even with a kernel-level firewall, advanced bots can slowly exhaust application-layer connection slots by opening sockets and remaining entirely silent [1.1.2]. 

To protect the containerized .NET connection pool from resource exhaustion, we implemented a highly resilient, lock-free **8-Layer Self-Healing Defense Pipeline** [1]:

### 1. The Multi-Tier Disarmable Timeouts
Bots frequently attempt to consume sockets by halting the connection flow at specific protocol stages [1.1.2]. We mitigated this by introducing two distinct, disarmable application-layer countdowns:
*   **The 15-Second Handshake Timeout:** When a client connects, they are granted exactly 15 seconds to complete the SSH cryptographic handshake. If they fail (a silent TCP probe), a `handshakeCts` cancellation token fires, tearing down the task and freeing the slot.
*   **The 15-Second Pre-Shell Negotiation Timeout:** A classic loophole involves bots that successfully negotiate the SSH transport layer (`Session negotiated | User: unknown`) but **never request a shell channel**. Since they never launch the shell, they bypass the keyboard idle timeout and stay connected forever. We established a `negotiationCts` timer that forces a clean teardown if a shell channel is not successfully requested and accepted within 15 seconds of negotiation [1.2.7].

```csharp
// Disarming the pre-shell countdown only upon successful shell launch
case "shell":
    e.IsAuthorized = true;
    try { negotiationCts.CancelAfter(Timeout.InfiniteTimeSpan); } catch { }
```

### 2. Immediate Session Teardown on Forbidden Requests
*   **The Problem:** Automated scanners immediately request an `exec` channel to run rapid command payloads [1.1.3]. In typical configurations, setting `e.IsAuthorized = false` merely rejects the request but keeps the parent SSH session (and TCP socket) open, allowing the bot to linger in memory.
*   **The Resolution:** We modified the C# request handler. The moment a client requests a forbidden channel (like `exec`), the gateway logs a sanitization-shielded event and immediately **terminates the parent SshSession** [1.1.3, 3.1]:
    ```csharp
    try
    {
        _ = channel.Session.CloseAsync(SshDisconnectReason.ByApplication, "Execution Rejected");
    }
    ```
    This abruptly drops the physical TCP socket and releases the connection slot back to the semaphore within milliseconds [3.1].

### 3. OS-Level Socket Keep-Alives (Half-Open Connection Reclamation)
If a client silently drops offline (e.g., cellular loss or sudden reboot) *after* launching the TUI, the socket enters a "half-open" state. By default, the Linux kernel can take up to 2 hours to detect and clean up this dead socket. 

We configured aggressive, native TCP keep-alive parameters directly at the accepted socket layer [2.2.1]:
```csharp
socket.SetSocketOption(SocketOptionLevel.Socket, SocketOptionName.KeepAlive, true);
socket.SetSocketOption(SocketOptionLevel.Tcp, SocketOptionName.TcpKeepAliveTime, 60); // Wait 60s of silence before probing
socket.SetSocketOption(SocketOptionLevel.Tcp, SocketOptionName.TcpKeepAliveInterval, 10); // Wait 10s between unanswered probes
socket.SetSocketOption(SocketOptionLevel.Tcp, SocketOptionName.TcpKeepAliveRetryCount, 5); // Close after 5 failed probes
```
This forces the OS to automatically sweep away dead sockets in under 2 minutes, restoring slot capacity [2.2.1].

### 4. Lock-Free Sliding-Window Rate Limiting
*   **The Problem:** Under a massive, globally distributed connection flood, a single global lock statement (`_historyLock`) blocking list evaluations will cause high CPU thread contention, degrading performance for active users.
*   **The Resolution:** We refactored the per-IP rolling sliding window history. The dictionary was transitioned to a fully lock-free concurrent queue model:
    ```csharp
    private readonly ConcurrentDictionary<string, ConcurrentQueue<DateTime>> _connectionHistory = new();
    ```
    Pruning old history items is handled asynchronously using lock-free `TryPeek`/`TryDequeue` operations, completely eliminating synchronization bottlenecks under load.

### 5. Log Injection Shielding (Fail2Ban Defense)
Because Fail2Ban parses container console logs to execute blocks [1], the logging pipeline is protected against log spoofing. If an attacker passes a malicious username containing Carriage Return/Line Feed (`\r\n`) characters or ANSI color escape codes, they could forge log entries to ban innocent IPs.

We integrated a robust regex-based sanitization wrapper on all network-parsed logging parameters (Usernames and Request Types) before they write to the console:
```csharp
private static string SanitizeLogInput(string? input, int maxLength = 50)
{
    if (string.IsNullOrEmpty(input)) return "unknown";
    var truncated = input.Length > maxLength ? input.Substring(0, maxLength) : input;
    var clean = Regex.Replace(truncated, @"[\p{C}\r\n]", string.Empty); // Strips control characters
    return Regex.Replace(clean, @"\x1B\[[^@-_]*[0-9a-zA-Z]", string.Empty); // Strips ANSI escape codes
}
```

### 6. Bounded Tarpits (Thread Pool Protection)
Application-level "tarpitting" (introducing artificial delays to slow down scanners) runs asynchronously in the background. However, under a volumetric connection flood, spawning thousands of unmetered `Task.Delay` tasks can cause extreme heap memory bloat.

We introduced an active tarpit gatekeeper. If the count of concurrent tarpitted tasks exceeds `100`, the gateway automatically bypasses the tarpit delay for subsequent rejects and drops the sockets instantly to protect the server’s Thread Pool:
```csharp
if (Interlocked.Increment(ref _activeTarpits) > MaxConcurrentTarpits)
{
    Interlocked.Decrement(ref _activeTarpits);
    try { client.Close(); } catch { }
    return;
}
```

---

## Component 5: Taming Windows — Restricting OpenSSH Permissions

### The local OpenSSH Permissions Block
On your local Windows machine, attempts to use the administrative alias `ssh vps-admin` were blocked with:

```text
Bad owner or permissions on C:\Users\angel/.ssh/config
```

### The Root Cause: Permissive Windows ACL Inheritance
The Windows OpenSSH client enforces strict file-system permissions. Because the `config` file was created in your user directory, it automatically inherited permissions from the parent folder, allowing other local user accounts (like `DESKTOP-SP3HC83\test`) read access to the file [1.2.3, 1.2.6]. OpenSSH identifies this as an insecure configuration and rejects the file [1.2.3].

### The Resolution: ACL Inheritance Stripping
We used the Windows command-line Access Control list utility (**`icacls`**) to strip away inherited access and grant exclusive read/write rights to your active Windows identity [1.1.6, 1.2.3]:

```cmd
:: Disable inheritance and remove other users
icacls C:\Users\angel\.ssh\config /inheritance:r

:: Grant exclusive read/write permissions to your active account
icacls C:\Users\angel\.ssh\config /grant:r "%username%:(R,W)"
```

This successfully resolved the client-side validation errors, allowing you to connect using your customized administrative aliases [1.2.3].

---

## The Administrator's Toolbelt: Verification & Monitoring

Here is a reference guide of commands for keeping tabs on your server's state:

### 1. View Active Sockets and Connections
Verify which processes are actively holding your public ports:
```bash
sudo ss -tanp '( sport = :22 or sport = :43829 )'
```

### 2. View Active Kernel-Layer Blocks
Check automated bans, manual administrative bans, or rate-limiting watchlists:
```bash
# View automated, short-term bans (Fail2Ban)
sudo nft list table inet f2b-table

# View persistent, manual administrative bans
sudo nft list set inet host_firewall app_banlist_v4

# View rate-limiting watchlists
sudo nft list set inet host_firewall app_ssh_flood_v4
```

### 3. Check Intrusion Prevention States
Inspect the health and block count of your active jails:
```bash
# Check the status of the gateway jail
sudo fail2ban-client status him-gateway

# Check the status of the repeat-offender jail
sudo fail2ban-client status recidive
```

### 4. Stream Live Drop Actions
Watch the firewall silently drop attacker connections in real-time:
```bash
sudo journalctl -k -f | grep "INPUT_BLOCKED"
```
