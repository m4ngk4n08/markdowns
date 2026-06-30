Here are the precise commands to monitor your security system in real time.
These commands let you check your firewall's Ban List, the Watchlist (how many
times IPs have connected), and the Dropped/Rejected connections.

Part 1: View the Ban Lists (Who is currently locked out)

These commands show you who has crossed the line and is currently being ignored
by the server [1, 2].

Command A: View Firewall Blocks (Port 22 - Public Application)

sudo nft list set inet host_firewall app_banlist_v4

  - What it shows: The list of IP addresses currently blocked at the kernel
    layer for attempting to connect to your .NET app too fast [1].
  - What to look for: Look under elements = { ... } [1]. It will show the banned
    IP address and exactly how many minutes/seconds are left before their 1-hour
    ban expires [1].

Command B: View Administrative Blocks (Port 43829 - Fail2Ban)

sudo fail2ban-client status sshd

  - What it shows: The list of IPs currently banned from accessing your Host OS
    login port 43829 [2].
  - What to look for: Look at the bottom of the output next to Banned IP list:
    [2].

Part 2: View the Watchlist (Tracking connection limits)

Before a bot is banned, the firewall tracks how many times they connect in a
rolling 60-second window. This is their "watchlist" or "speedometer" [1].

Command:

sudo nft list set inet host_firewall app_ssh_flood_v4

  - What it shows: The database of IPs currently connecting to your .NET app,
    their current "knock" count, and how much time is left before their tracking
    window resets [1].
  - What to look for: Under elements = { ... }, you will see entries like:
    122.54.87.191 limit rate over 5/minute timeout 5m expires 13s
    This means the IP 122.54.87.191 is currently being tracked [1]. If they stop
    knocking, they will disappear from this list in 13 seconds [1]. If they
    knock again before then, the counter updates [1].

Part 3: View Blocked & Rejected Connections

These commands show you the actual logs of traffic being stopped—both by the
host firewall and internally by your .NET app [1].

Command A: View Firewall Blocks (Port 22/2222 and other blocked ports)

sudo journalctl -k --no-pager -n 50 | grep "INPUT_BLOCKED"

  - What it shows: The last 50 attempts to connect to your server that were
    blocked and logged at the kernel layer [1].
  - What to look for: Look at the SRC= (source IP) and DPT= (destination port)
    [1]. This shows you who tried to scan you and which closed port they were
    trying to hit [1].

Command B: View Application Rejections (Internal .NET Sandbox Blocks)
Confirm that Fail2Ban is now running both the standard .NET gateway jail and the escalation jail [2]:
code Bash

sudo fail2ban-client status

Expected Output:
code Text

Status
|- Number of jail:      3
`- Jail list:           him-gateway, recidive, sshd

To view the active repeat-offender ban list at any time, run [2]:
code Bash

sudo fail2ban-client status recidive
Navigate to your project directory and check your app's logs:

cd ~/him-portfolio
sudo docker compose logs him-gateway --tail=50 | grep "REJECTED"

  - What it shows: The last 50 times an attacker successfully made a connection
    to port 22 but tried to run an unauthorized command (like a shell execution)
    and was kicked out by your .NET code's sandbox [1].
  - What to look for: Look for lines like:
    [Security] REJECTED request | Type: exec | User: root
    This shows your .NET app's custom security layer identifying and blocking
    malicious commands.

sudo fail2ban-client status sshd
sudo nft list table inet f2b-table

Confirm that Fail2Ban is now running both the standard .NET gateway jail and the escalation jail [2]:
code Bash

sudo fail2ban-client status

Expected Output:
code Text

Status
|- Number of jail:      3
`- Jail list:           him-gateway, recidive, sshd

To view the active repeat-offender ban list at any time, run [2]:
code Bash

sudo fail2ban-client status recidive