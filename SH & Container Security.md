# Production SSH & Container Security Playbook
**Platform:** Ubuntu Server 24.04 LTS (Noble)  
**Host Services:** OpenSSH (Port 43829)  
**Container Services:** .NET 8 SSH Application (`him-gateway` via Docker, Host Port 2222)  
**Firewall Framework:** Native `nftables` (no `iptables-legacy`, no `ufw`)  

---

## Packet Flow & System Architecture

When protecting a containerized service (such as a .NET SSH application mapped to host port 2222) that lacks traditional `/etc/ssh/sshd_config` management, security controls must be applied at the kernel layer before packets are routed into Docker's virtual network. 

```
                           [ INTERNET ]
                                |
                                v
                     [ Network Interface Card ]
                                |
                                v
               [ netfilter: PREROUTING Hook ]
                 | (Priority -200: conntrack)
                 v
                 | (Priority -150: Our Custom Rate Limiting) <-- Abusive IPs dropped here!
                 v
                 | (Priority -100: Docker NAT / DNAT)       <-- Port 2222 mapped to Container 22
                 v
                     [ Routing Decision ]
                               / \
                    To Local  /   \  To Routed
                      Host   /     \   Container
                            /       \
                           v         v
                  [ INPUT Hook ]   [ FORWARD Hook ]         <-- Bypasses INPUT completely!
                        |            |
                        v            v
                 [ OpenSSH (43829) ] [ Docker Bridge (docker0) ]
                                     |
                                     v
                             [ Container Veth ]
                                     |
                                     v
                        [ Container Net Namespace ]
                                     |
                                     v
                          [ .NET SSH Server (22) ]
```

### 1. Host-Level Filtering Prior to Container Delivery
In Linux Netfilter, destination network address translation (DNAT) occurs in the `prerouting` hook at priority `-100` (`NF_IP_PRI_NAT_DST`). Docker uses this mechanism to translate incoming packets on host port `2222` to the container’s internal IP (e.g., `172.18.0.2`) and internal port `22`. 

Once DNAT occurs and the destination IP is rewritten to an address within the Docker bridge range, the routing engine identifies the packet as destined for a different network interface. Consequently, the packet **bypasses the `input` hook entirely** and traverses the `forward` hook. 

By positioning our custom filtering and rate limiting in the `prerouting` hook at priority `-150` (which executes *after* connection tracking at `-200` but *before* Docker's DNAT at `-100`), we intercept malicious packets while they are still addressed to the host’s public IP and port `2222`. This allows us to drop abusive traffic before any DNAT state or forwarding overhead is incurred.

### 2. Efficiency of Kernel-Layer Filtering vs. Application-Layer Filtering
*   **Kernel Layer (`nftables` / `conntrack`):** Drops packets in nanoseconds. It evaluates incoming packets against memory-resident red-black trees and hash tables without context-switching to user space, allocating heavy memory structures, or completing the TCP three-way handshake (when dropping at the first SYN).
*   **Application Layer (.NET Runtime / CLR):** Letting abusive traffic bypass the firewall forces the Linux kernel to complete the TCP handshake, allocate a socket file descriptor, context-switch to user space, schedule a thread within the container's runtime, and initiate cryptographic key exchange handshakes. Under connection-flood or brute-force conditions, this quickly starves the container of file descriptors and CPU cycles, causing a Denial of Service (DoS) for legitimate users.

### 3. Docker's Integration with `nftables`
Docker manages its firewalling rules using standard `iptables` interfaces. On modern systems like Ubuntu 24.04, this translates to the `iptables-nft` compatibility layer, which automatically populates legacy-named nftables tables (`ip filter`, `ip nat`, etc.) behind the scenes. 

If an administrator executes `flush ruleset` in their nftables configuration, it destroys Docker’s dynamically created translation tables, completely breaking container NAT and forwarding. To coexist with Docker, we must define our rules inside a custom, isolated nftables table (e.g., `inet host_firewall`). We then only flush our specific table, leaving Docker's auto-generated tables intact.

---

## Phase -1: Backup & Rollback Preparation

Before introducing any modifications to system configurations, establish a known-good recovery point.

### 1. Create Backups of Configuration Files
Execute the following commands to archive current configuration states:
```bash
# Backup existing nftables ruleset
sudo nft list ruleset > /etc/nftables.conf.bak 2>/dev/null || echo "No existing nftables rules"

# Backup kernel parameters
sudo cp /etc/sysctl.conf /etc/sysctl.conf.bak
if [ -d /etc/sysctl.d ]; then
    sudo tar -czf /etc/sysctl.d.bak.tar.gz /etc/sysctl.d/
fi

# Backup OpenSSH daemon configuration
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
if [ -d /etc/ssh/sshd_config.d ]; then
    sudo tar -czf /etc/ssh/sshd_config.d.bak.tar.gz /etc/ssh/sshd_config.d/
fi
```

### 2. Immediate Rollback Script
Create an emergency restore script at `/root/rollback.sh` to revert all modifications instantly if you are locked out but maintain an active terminal session, or if you use an out-of-band management console:
```bash
cat << 'EOF' | sudo tee /root/rollback.sh > /dev/null
#!/bin/bash
echo "Initiating emergency configurations rollback..."

# Restore OpenSSH
if [ -f /etc/ssh/sshd_config.bak ]; then
    cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
    if [ -f /etc/ssh/sshd_config.d.bak.tar.gz ]; then
        rm -rf /etc/ssh/sshd_config.d/
        tar -xzf /etc/ssh/sshd_config.d.bak.tar.gz -C /
    fi
    systemctl restart ssh
    echo "[✓] OpenSSH restored."
fi

# Restore Sysctl
if [ -f /etc/sysctl.conf.bak ]; then
    cp /etc/sysctl.conf.bak /etc/sysctl.conf
    rm -f /etc/sysctl.d/99-security-hardening.conf
    if [ -f /etc/sysctl.d.bak.tar.gz ]; then
        rm -rf /etc/sysctl.d/
        tar -xzf /etc/sysctl.d.bak.tar.gz -C /
    fi
    sysctl --system
    echo "[✓] Kernel parameters restored."
fi

# Restore nftables
if [ -s /etc/nftables.conf.bak ]; then
    nft -f /etc/nftables.conf.bak
    echo "[✓] Original nftables ruleset restored."
else
    # Fallback to a basic open rule if backup doesn't exist
    nft flush ruleset
    echo "[!] Warning: nftables completely flushed to prevent lockout."
fi

# Stop and Disable Fail2Ban during rollback
systemctl stop fail2ban
systemctl disable fail2ban
echo "[✓] Fail2Ban disabled."
EOF

sudo chmod +x /root/rollback.sh
```

---

## Phase 0: Pre-flight Validation

Verify the system’s initial networking state and protect the active administration session prior to applying structural firewall modifications.

### 1. Collect Active State Data
Run the following verification suite to document current listening sockets, active containers, and active rules:
```bash
# Check active listening ports
sudo ss -tlnp

# Verify Docker container status
sudo docker ps

# Record current administrator IP and source port
echo "Your connection parameters: $SSH_CLIENT"
```

### 2. Temporarily Allow current Administrator IP
To prevent an immediate lockout when initializing `nftables`, isolate your public IP address and verify it is whitelisted. Run this to store your current IP in a variable:
```bash
EXPORTED_ADMIN_IP=$(echo $SSH_CLIENT | awk '{print $1}')
echo "Your administrator IP is: $EXPORTED_ADMIN_IP"
```
*If `$SSH_CLIENT` is empty (e.g., you are connected over a nested terminal), manually set the variable: `EXPORTED_ADMIN_IP="your.actual.ip.here"`.*

---

## Phase 1: OpenSSH Hardening

Modify the SSH daemon to enforce cryptographic standards, prevent automated brute-forcing, and limit resource exposure.

### 1. Configure `/etc/ssh/sshd_config`
Write the hardened configuration file. Ensure any drop-in files under `/etc/ssh/sshd_config.d/` are removed or accounted for, as they can override these settings.

```text
# /etc/ssh/sshd_config
Port 43829
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

# Host Key Files
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key

# Authentication Restrictions
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no

# Session Settings & Resource Management
MaxAuthTries 3
MaxStartups 10:30:100
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

# Channel & Forwarding Restrictions
AllowTcpForwarding no
X11Forwarding no
AllowAgentForwarding no
Compression no
TCPKeepAlive no

# System Integration
UsePAM yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
```

### 2. Operational Parameter Analysis

| Directive | Configuration Value | Security Impact | Potential Failure Mode |
| :--- | :--- | :--- | :--- |
| `Port` | `43829` | Shifts OpenSSH away from port 22, reducing automated background script log noise. | If firewalls are not adjusted first, new connections will be blocked immediately upon service restart. |
| `PermitRootLogin` | `prohibit-password` | Allows `root` to connect exclusively via SSH keys, rendering brute-force password cracking attempts on root ineffective. | If root’s `authorized_keys` file is missing or contains corrupt keys, root access is entirely lost. |
| `PasswordAuthentication` | `no` | Mandates cryptographic public keys for authentication, neutralizing credentials-based automated attacks. | If a client does not have their key pair loaded or matches the host's configured key types, access is denied. |
| `MaxAuthTries` | `3` | Limits authentication attempts per connection to 3. Connections are dropped after 3 failures. | Software that cycles through multiple local SSH keys may hit this limit and get dropped before trying the correct key. |
| `MaxStartups` | `10:30:100` | Throttles unauthenticated connections. Drops new attempts with a 30% probability at 10 concurrent connections, scaling to 100% drops at 100 connections. | Legitimate rapid connections (e.g., multiple parallel CI/CD runners) might be dropped during heavy deployment periods. |
| `LoginGraceTime` | `30` | Reduces the window to complete authentication to 30 seconds (down from the default of 120). | Slow network connections or manual entry of SSH key passphrases might time out before completing login. |
| `ClientAliveInterval` | `300` | Sends encrypted alive-check packets every 300 seconds (5 minutes) to the client. | If the client network is temporarily unstable, inactive sessions are closed relatively quickly. |
| `ClientAliveCountMax` | `2` | Terminates the session if the client fails to respond to 2 consecutive keepalive packets (after 10 minutes). | Legitimate long-running idle terminal sessions on shaky connections will be disconnected. |
| `AllowTcpForwarding` | `no` | Prevents users from tunnel routing local/remote TCP traffic through the SSH connection. | Breaks administrative SSH-tunneling use cases (e.g., local port forwarding to access internal databases). |
| `X11Forwarding` | `no` | Prevents remote GUI applications from forwarding graphics display protocols back to the client. | Disables GUI window streaming from the VPS (rarely needed on headless production nodes). |
| `AllowAgentForwarding` | `no` | Prevents local client agents from forwarding access to remote hosts. | Breaks administrative "chaining" to other systems via the active agent without copying private keys to the VPS. |
| `Compression` | `no` | Disables packet compression, eliminating potential side-channel timing leaks and reducing server CPU overhead. | Negligibly increases network bandwidth consumption for highly repetitive text-heavy streams. |

### 3. Verification & Application
Never restart the SSH daemon without testing the updated configuration file syntactically.

```bash
# Validate sshd syntax
sudo sshd -t
```
*   **Expected Output:** Command returns with exit status `0` and outputs no text.
*   **Failure Symptoms:** Errors indicating unrecognized options, incorrect values, or file permissions.
*   **Troubleshooting:** Open another terminal (do not close your current shell) and correct the line number specified in the error message.

Once validation passes, restart the service:
```bash
sudo systemctl restart ssh
```

---

## Phase 2: Kernel Hardening

Apply kernel-level parameters to secure the network stack against TCP resources exhaustion, specifically SYN floods and socket starvation.

### 1. Configuration File
Create the hardening drop-in file at `/etc/sysctl.d/99-security-hardening.conf`:

```ini
# /etc/sysctl.d/99-security-hardening.conf

# Enable SYN Cookie Protection
net.ipv4.tcp_syncookies = 1

# Limit retries of SYN-ACK segments
net.ipv4.tcp_synack_retries = 2

# Increase the maximum SYN backlog queue size
net.ipv4.tcp_max_syn_backlog = 4096

# Increase maximum backlog queue for socket listeners
net.core.somaxconn = 4096

# Increase maximum number of packets queued in the input driver
net.core.netdev_max_backlog = 2500
```

### 2. Operational Parameter Analysis

#### `net.ipv4.tcp_syncookies = 1`
*   **Why it exists:** When the connection queue (`tcp_max_syn_backlog`) is full, the kernel stops storing state for new SYN requests. Instead, it generates a cryptographic hash based on the client's IP, port, timestamp, and MSS, returning this hash as the sequence number in the SYN-ACK. When the client returns the final ACK, the kernel validates the hash to open the socket.
*   **When beneficial:** Vital during TCP SYN flood attacks, ensuring legitimate users can still connect even when the half-open connection queue is saturated.
*   **Drawbacks:** Bypasses TCP extensions (such as window scaling) on connection establishment unless TCP timestamps (`net.ipv4.tcp_timestamps`) are enabled. It also incurs a tiny CPU overhead to calculate the hash for every incoming connection attempt under load.

#### `net.ipv4.tcp_synack_retries = 2`
*   **Why it exists:** Controls the number of times the kernel attempts to retransmit the SYN-ACK segment before terminating an unacknowledged handshaking connection.
*   **When beneficial:** The system default is usually `5` retries (which holds resources open for roughly 180 seconds). Restricting this value to `2` reaps incomplete half-open connections after approximately 15 seconds, releasing kernel resources much faster.
*   **Drawbacks:** Clients on highly latent or dropping connections (e.g., unstable mobile data links) might fail to complete handshakes on their first connection attempts.

#### `net.ipv4.tcp_max_syn_backlog = 4096`
*   **Why it exists:** Dictates the maximum number of half-open TCP connections (client has sent SYN, server returned SYN-ACK, client hasn't replied with final ACK yet) the host can hold in state.
*   **When beneficial:** Increasing this parameter from standard defaults (often `256` or `512`) provides a buffer to accommodate spikes in legitimate traffic or low-rate SYN attacks without triggering SYN cookies prematurely.
*   **Drawbacks:** Allocates a tiny amount of additional kernel memory (Slab allocator) to track connection tracking entries.

#### `net.core.somaxconn = 4096`
*   **Why it exists:** The maximum backlog size for established sockets awaiting an `accept()` system call by user-space applications (e.g., Nginx, .NET runtimes).
*   **When beneficial:** Essential for applications that accept high rates of incoming connections. It buffers fully established TCP sessions during transient user-space scheduling lags.
*   **Drawbacks:** If the application process hangs, setting this too high queues up client connections in kernel memory, masking the application's failure rather than letting clients fail fast.

#### `net.core.netdev_max_backlog = 2500`
*   **Why it exists:** Determines how many raw network packets can queue up in the input buffer of the network interface card (NIC) driver before the operating system kernel processes them.
*   **When beneficial:** Prevents physical network ring buffer packet drops on systems processing packets at multi-gigabit speeds.
*   **Drawbacks:** Allocates a small portion of physical system RAM for network packet buffering.

### 3. Application & Verification
Apply the settings:
```bash
sudo sysctl --system
```
Verify the active parameters:
```bash
sysctl net.ipv4.tcp_syncookies net.ipv4.tcp_synack_retries net.ipv4.tcp_max_syn_backlog net.core.somaxconn net.core.netdev_max_backlog
```

---

## Phase 3: `nftables` Deployment

Define and apply the persistent, non-disruptive `nftables` firewall configuration.

### 1. Configuration File `/etc/nftables.conf`
We will establish our rules inside a custom table `inet host_firewall`. We perform a targeted flush on our own table to avoid wiping Docker's dynamic rules.

```text
#!/usr/sbin/nft -f

# Ensure the custom table exists
add table inet host_firewall

# Flush ONLY our custom table to guarantee idempotent reloads without destroying Docker's structures
flush table inet host_firewall

table inet host_firewall {
    # -----------------------------------------------------------------
    # Sets & Maps (Dynamic Blacklists and Whitelists)
    # -----------------------------------------------------------------

    # Track abusive hosts targeting OpenSSH (Port 43829)
    set admin_ssh_flood_v4 {
        type ipv4_addr
        flags dynamic, timeout
        timeout 5m
        size 65535
    }
    set admin_ssh_flood_v6 {
        type ipv6_addr
        flags dynamic, timeout
        timeout 5m
        size 65535
    }

    # Banned hosts for OpenSSH
    set admin_banlist_v4 {
        type ipv4_addr
        flags dynamic, timeout
        timeout 1h
        size 65535
    }
    set admin_banlist_v6 {
        type ipv6_addr
        flags dynamic, timeout
        timeout 1h
        size 65535
    }

    # Track abusive hosts targeting .NET SSH (Port 2222)
    set app_ssh_flood_v4 {
        type ipv4_addr
        flags dynamic, timeout
        timeout 5m
        size 65535
    }
    set app_ssh_flood_v6 {
        type ipv6_addr
        flags dynamic, timeout
        timeout 5m
        size 65535
    }

    # Banned hosts for .NET SSH
    set app_banlist_v4 {
        type ipv4_addr
        flags dynamic, timeout
        timeout 1h
        size 65535
    }
    set app_banlist_v6 {
        type ipv6_addr
        flags dynamic, timeout
        timeout 1h
        size 65535
    }

    # Static/Dynamic Whitelists (Persisted across loads)
    set admin_whitelist_v4 {
        type ipv4_addr
        elements = {
            127.0.0.1
        }
    }
    set admin_whitelist_v6 {
        type ipv6_addr
        elements = {
            ::1
        }
    }

    # -----------------------------------------------------------------
    # Chains
    # -----------------------------------------------------------------

    # 1. PREROUTING: Intercept packets prior to routing decisions & NAT
    chain prerouting {
        type filter hook prerouting priority -150; policy accept;

        # Allow Loopback interface traffic
        iif "lo" accept

        # Bypass checks for whitelisted administrator IPs
        ip saddr @admin_whitelist_v4 accept
        ip6 saddr @admin_whitelist_v6 accept

        # Drop packets from banned IPs immediately at Layer 3
        ip saddr @admin_banlist_v4 drop
        ip6 saddr @admin_banlist_v6 drop
        ip saddr @app_banlist_v4 drop
        ip6 saddr @app_banlist_v6 drop

        # Protect Host OpenSSH (Port 43829)
        # Allows 5 new connection attempts per minute. Excess IPs banned for 1 hour.
        tcp dport 43829 ct state new ip saddr update @admin_ssh_flood_v4 { ip saddr limit rate over 5/minute } add @admin_banlist_v4 { ip saddr } drop
        tcp dport 43829 ct state new ip6 saddr update @admin_ssh_flood_v6 { ip6 saddr limit rate over 5/minute } add @admin_banlist_v6 { ip6 saddr } drop

        # Protect Container .NET SSH (Port 2222)
        # Allows 10 new connection attempts per minute. Excess IPs banned for 1 hour.
        tcp dport 2222 ct state new ip saddr update @app_ssh_flood_v4 { ip saddr limit rate over 10/minute } add @app_banlist_v4 { ip saddr } drop
        tcp dport 2222 ct state new ip6 saddr update @app_ssh_flood_v6 { ip6 saddr limit rate over 10/minute } add @app_banlist_v6 { ip6 saddr } drop
    }

    # 2. INPUT: Protect the Host OS itself
    chain input {
        type filter hook input priority 0; policy drop;

        # Loopback
        iif "lo" accept

        # Reject Invalid packets (SYN-FLOOD mitigation and state hygiene)
        ct state invalid log prefix "INVALID_PACKET: " flags all drop

        # Connection State Filtering
        ct state established,related accept

        # Rate-limited ICMP (Ping)
        ip protocol icmp icmp type echo-request limit rate 2/second accept
        ip6 nexthdr icmpv6 icmpv6 type echo-request limit rate 2/second accept

        # Explicitly open admin port (already rate-limited in prerouting)
        tcp dport 43829 accept

        # Log and drop everything else
        log prefix "INPUT_BLOCKED: " flags all drop
    }

    # 3. FORWARD: Coexist with Docker Network Forwarding
    # Docker manages its own FORWARD chain in its table. We set policy to accept
    # here to defer to Docker's filter forwarding.
    chain forward {
        type filter hook forward priority 0; policy accept;
    }

    # 4. OUTPUT: Outbound connections
    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

### 2. Operational Ruleset Analysis

#### Chains & Hooks
*   **`prerouting` Hook (Priority `-150`):** Operates on incoming packets immediately after connection tracking (`-200`) resolves their state, but before Docker’s NAT table modifies the destination address (`-100`). Evaluating rate limits here protects both local and containerized ports.
*   **`input` Hook (Priority `0`):** Handles traffic destined to local applications on the host. All non-established inbound connections are rejected by default unless specifically allowed (such as port 43829).
*   **`forward` Hook (Priority `0`):** Intercepts traffic crossing the system's interfaces to other destinations (such as physical interfaces routing to the Docker virtual network). Leaving the policy as `accept` here ensures that Docker’s dynamic forwarding rules (configured inside separate iptables-nft tables) continue to work properly.

#### Connection States
*   **`ct state invalid`:** Tracks packets that do not belong to any known active session (e.g., malformed headers or out-of-order sequence flags). Rejecting these prevents resource allocation for dead sessions.
*   **`ct state established, related`:** Fast-tracks packets that belong to existing, authenticated TCP sessions or related helpers (such as ICMP replies). It bypasses evaluation of rule lists for open connections, optimizing system throughput.
*   **`ct state new`:** Targets only the initial packet of a connection attempt (the TCP SYN). Applying limits to `new` connections stops brute-force tools from exhausting listening threads without disrupting active, established sessions.

#### Rate-Limiting Expressions & BAN Actions
The rate limiting statement uses a single-line sequential logic construct:
```text
tcp dport 43829 ct state new ip saddr update @admin_ssh_flood_v4 { ip saddr limit rate over 5/minute } add @admin_banlist_v4 { ip saddr } drop
```
1.  **State Match:** The packet matches port `43829`, has `new` connection state, and has an IPv4 source address.
2.  **Set Update:** It updates the tracking set `admin_ssh_flood_v4` with the source IP address key.
3.  **Threshold Check:** The `limit rate over 5/minute` stateful expression is evaluated.
    *   **Under Limit:** The expression returns `false`. Evaluation of the current rule halts instantly, the remaining statements in the rule (`add` and `drop`) are ignored, and the packet transitions to the next rule (and eventually to the `input` chain where it is accepted).
    *   **Over Limit:** The expression returns `true`. Evaluation continues to the next sequential statement: `add @admin_banlist_v4 { ip saddr }`, which inserts the IP into our blocklist set with a 1-hour expiration.
    *   **Action Execution:** The final statement `drop` executes, discarding the packet.

#### DROP vs. REJECT
*   **`drop`** silently discards packets. It forces attackers to wait for connection timeouts, slowing down automated scanners and saving outbound bandwidth. We use `drop` for blocked connection rate limits and generic firewall drops.
*   **`reject`** returns an explicit ICMP Destination Unreachable / Port Unreachable message. It tells legitimate clients instantly that the port is closed, avoiding connection hanging. 

---

## Phase 4: Fail2Ban

Deploy Fail2Ban to monitor OpenSSH authorization logs and dynamically ban IPs that fail key-based SSH authentication.

### 1. Excluding Container SSH from Fail2Ban
Fail2Ban is **incapable** of protecting our custom .NET 8 `him-gateway` containerized SSH application:
*   The application does not run OpenSSH and does not write to standard host log formats like `/var/log/auth.log` or the systemd journal.
*   Authentications happen within an isolated container namespace where the host has no native log context.
*   Scraping container logs via `docker logs` from the host introduces performance delays, parsing challenges, and complex integration dependencies.

Instead, the container's port `2222` is fully protected strictly at the kernel layer (L3/L4) using the rate-limiting and dynamic auto-banning nftables configurations established in **Phase 3**.

### 2. Configure Fail2Ban for Host OpenSSH
Define a localized jail configuration at `/etc/fail2ban/jail.d/openssh-hardened.local`:

```ini
# /etc/fail2ban/jail.d/openssh-hardened.local

[DEFAULT]
# Global defaults
bantime  = 1h
findtime = 10m
maxretry = 3

# Use the highly performant systemd journal backend
backend = systemd

[sshd]
enabled  = true
port     = 43829
filter   = sshd
backend  = systemd
maxretry = 3
findtime = 10m
bantime  = 1h
```

### 3. Why the `systemd` Backend is Highly Recommended
On Ubuntu 24.04 LTS, `rsyslog` is often not installed by default. This means text-based files like `/var/log/auth.log` may not exist or may be empty.
*   **Structured Data Access:** The `systemd` backend queries the binary systemd journal database (`journald`) directly via native API bindings, avoiding the overhead of continuously polling and parsing text streams.
*   **Tamper Resistance:** Attackers cannot easily inject arbitrary carriage returns or fake log lines to spoof auth failures, as journald fields are structured and populated securely by the kernel and system services.
*   **Zero Log-Rotation Latency:** Fail2Ban is notified of authentication failures instantly through the systemd journal API, allowing it to apply bans immediately instead of waiting for file write buffers to flush to disk.

---

## Phase 5: Persistence & Service Management

Configure systemd to ensure the hardened configurations persist correctly across reboots and start up in the correct order.

### 1. Systemd Startup Sequence
Systemd initializes services during boot based on target definitions and dependency relationships.

```
[ Boot Init ]
     |
     v
[ nftables.service ] (DefaultDependencies=no, Before=network-pre.target)
     |
     v
[ Network Interfaces Online ]
     |
     v
[ docker.service ]   (Configures bridge and NAT forwarding tables)
     |
     v
[ fail2ban.service ] (Monitors journald, applies jail rules to nftables)
```

1.  **`nftables.service`** is executed exceptionally early during system boot (`DefaultDependencies=no`, `Before=network-pre.target`). This guarantees that our custom firewall boundaries are active *before* network interfaces are configured and exposed to the public internet.
2.  **Network interfaces** are initialized and brought online.
3.  **`docker.service`** starts. It reads local configurations, starts the virtual bridge networking, and populates the `iptables-nft` translation tables. Because we designed our `/etc/nftables.conf` to modify *only* our custom table (`inet host_firewall`) and explicitly avoided `flush ruleset`, our custom rules and Docker's routing rules coexist cleanly without overwriting each other.
4.  **`fail2ban.service`** starts last (`After=network.target nftables.service`). It hooks into `journald` to monitor authentication logs and applies dynamic iptables/nftables actions.

### 2. Enabling Persistence Services
Verify and configure systemd to enable both services on system startup:

```bash
# Verify nftables configuration syntax before loading
sudo nft -c -f /etc/nftables.conf

# Enable and start nftables
sudo systemctl enable nftables
sudo systemctl restart nftables

# Enable and start Fail2Ban
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
```

---

## Phase 6: Validation Suite

Execute the following verification tests to confirm your configurations are active and working correctly.

### 1. Active Listening Sockets
```bash
sudo ss -tlnp
```
*   **Expected Output:** 
    *   OpenSSH is listening exclusively on port `43829`: `LISTEN ... 0.0.0.0:43829` and `[::]:43829`.
    *   Docker proxy (or container) is listening on host port `2222`: `LISTEN ... 0.0.0.0:2222` and `[::]:2222`.

### 2. Active Firewall Table Rules
```bash
sudo nft list table inet host_firewall
```
*   **Expected Output:** Prints the full, structured ruleset defined in Phase 3. 
*   Ensure that the `admin_whitelist_v4` contains your local administrator IP.

### 3. Fail2Ban OpenSSH Jail Status
```bash
sudo fail2ban-client status sshd
```
*   **Expected Output:** 
    *   `Status for the jail: sshd`
    *   `Filter: ... Journal`
    *   `Actions: ... ban IP`
    *   `Currently banned: 0` (or active counts).

### 4. Hardened Kernel Parameters
```bash
sudo sysctl net.ipv4.tcp_syncookies net.ipv4.tcp_synack_retries net.ipv4.tcp_max_syn_backlog net.core.somaxconn
```
*   **Expected Output:** Output matches the values defined in Phase 2.

### 5. Running Containers
```bash
sudo docker ps --filter "name=him-gateway"
```
*   **Expected Output:** Shows the container active, healthy, and exposing port `2222` on the host mapping to `22` in the container.

---

## Phase 7: Systems Monitoring

Use these diagnostic tools to monitor active system health and identify potential attacks.

### 1. Stream Dropped & Rate-Limited Network Traffic
Stream firewall log outputs in real time to monitor blocked connection attempts:
```bash
sudo journalctl -k -f | grep -E "INPUT_BLOCKED|INVALID_PACKET"
```

### 2. Monitor Active IP Bans
Inspect active dynamic IP blocklists in our nftables sets:
```bash
# View IPs banned from Admin Port 43829
sudo nft list set inet host_firewall admin_banlist_v4
sudo nft list set inet host_firewall admin_banlist_v6

# View IPs banned from Application Port 2222
sudo nft list set inet host_firewall app_banlist_v4
sudo nft list set inet host_firewall app_banlist_v6
```

To view current rate limiting metrics and tracked IPs:
```bash
sudo nft list set inet host_firewall admin_ssh_flood_v4
sudo nft list set inet host_firewall app_ssh_flood_v4
```

### 3. Stream Fail2Ban Jail Bans
Stream Fail2Ban logs to monitor dynamic OpenSSH authentication bans:
```bash
sudo tail -f /var/log/fail2ban.log
```

### 4. Stream OpenSSH Authentication Logs
Monitor OpenSSH logins and key authentication attempts in real time:
```bash
sudo journalctl -u ssh -f
```

### 5. Monitor Active Connection Tracking (`conntrack`) Statistics
Check conntrack tables to verify system utilization and potential state exhaustion:
```bash
sudo conntrack -S
```
*This command outputs the current state of connection tracking tables, tracking details like total active entries, searched entries, inserts, and delete events.*

---

## Phase 8: Emergency Recovery Procedures

In the event of an accidental lockout or configuration failure, use your VPS provider’s **out-of-band management console** (VNC/KVM/iDRAC/IPMI) to log into the system and run these recovery procedures.

```
[ Accidental Admin Lockout ]
             |
             v
[ Access VPS Provider VNC/KVM Console ]
             |
             v
[ Log in as Root or Administrator ]
             |
             +----------------------------+---------------------------+
             |                            |                           |
             v                            v                           v
  [ Scenario A: Firewall ]      [ Scenario B: OpenSSH ]     [ Scenario C: Kernel ]
             |                            |                           |
   • Stop nftables:             • Run syntax test:          • Remove hardening file:
     systemctl stop nftables      sshd -t                     rm -f /etc/sysctl.d/99-...
   • Clear ruleset:             • Restore backup config:    • Reload kernel config:
     nft flush ruleset            cp sshd_config.bak sshd     sysctl --system
   • Restore from file:         • Restart OpenSSH:
     nft -f /etc/nftables.conf    systemctl restart ssh
```

### Scenario A: Accidental Firewall Lockout (nftables)
If your IP address gets dynamically banned or your administrator IP is misconfigured, follow these steps to restore access.

#### 1. Temporarily Stop and Flush the Firewall
Log in via VNC/KVM and run:
```bash
# Stop nftables
sudo systemctl stop nftables

# Alternatively, clear our custom table specifically to avoid disrupting container bridges
sudo nft flush table inet host_firewall
```

#### 2. Whitelist your Admin IP
To prevent getting locked out again on the next reload, add your administrator IP address directly to the persistent configuration file `/etc/nftables.conf`.
Open `/etc/nftables.conf` and locate the set definition:
```text
    set admin_whitelist_v4 {
        type ipv4_addr
        elements = {
            127.0.0.1,
            your.new.admin.ip  <-- Insert your current IP address here
        }
    }
```

#### 3. Re-test and Apply
```bash
# Validate ruleset syntax
sudo nft -c -f /etc/nftables.conf

# Restart the firewall once validated
sudo systemctl restart nftables
```

---

### Scenario B: Broken SSH Daemon Configuration
If the OpenSSH service fails to start or rejects valid client certificates because of a configuration issue.

#### 1. Validate Syntax Errors
Log in via VNC/KVM and run:
```bash
sudo sshd -t
```
*Read the output details carefully to locate any incorrect directives or syntax typos in `/etc/ssh/sshd_config`.*

#### 2. Revert to Backup
If you cannot quickly identify or fix the syntax issue, restore the backup configuration file we created in Phase -1:
```bash
sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
sudo systemctl restart ssh
```

---

### Scenario C: Unstable Kernel Settings (sysctl)
If applying the hardened kernel parameters causes system instability, network interface drops, or packet processing errors.

#### 1. Delete the Hardening Configuration File
Log in via VNC/KVM and run:
```bash
sudo rm -f /etc/sysctl.d/99-security-hardening.conf
```

#### 2. Reapply Default Parameters
```bash
# Re-load sysctl configuration to restore defaults
sudo sysctl --system
```