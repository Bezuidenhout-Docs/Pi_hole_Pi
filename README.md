![Status](https://img.shields.io/badge/status-active-brightgreen)
![Version](https://img.shields.io/badge/version-1.0.0-blue)
![Platform](https://img.shields.io/badge/platform-Raspberry%20Pi%203%20B-red)
![Pi-hole](https://img.shields.io/badge/Pi--hole-latest-green)
![Unbound](https://img.shields.io/badge/Unbound-recursive%20DNS-orange)
![Privacy](https://img.shields.io/badge/privacy-self--hosted-success)
![License](https://img.shields.io/badge/license-MIT-blue)

Project Overview
This project transforms a low-power single-board computer into a powerful, network-wide ad-blocking and privacy-enhancing server. It combines Pi-hole, the industry-standard DNS sinkhole for ad blocking, with Unbound, a validating, recursive, caching DNS resolver. The result is a completely self-contained DNS system: your network resolves domain names from the internet's root servers directly, without relying on third-party providers like Google, Cloudflare, or your ISP. No external entity can build a profile of your household's internet activity.

Key Features:

Network-Wide Ad Blocking: Blocks ads, trackers, and malware domains on every device connected to the network—phones, tablets, smart TVs, and computers—without installing software on each device.

Total DNS Privacy: Unbound acts as a recursive resolver, talking directly to the authoritative root servers. No forwarding to external DNS providers.

Improved Performance: Frequently accessed DNS records are cached locally, reducing lookup times to fractions of a millisecond.

Self-Contained: The entire stack runs on a single, low-power device you own and control.

Advanced Security: DNSSEC validation ensures the integrity of DNS responses, protecting against cache poisoning and spoofing attacks.

How It Works
The system is a chain of three components that every DNS query on the network must traverse:

Devices on the Network: A phone, laptop, or smart TV makes a request to load a website.

Pi-hole (The Gatekeeper): The device asks the Pi-hole for the website's IP address. Pi-hole checks the requested domain against its blocklists. If the domain is known to serve ads or malware, Pi-hole returns a null route (a black hole), and the content is never loaded. If the domain is clean, Pi-hole passes the request to Unbound.

Unbound (The Private Investigator): Unbound checks its cache. If the address is new, Unbound doesn't just ask a third party; it starts at the internet's root DNS servers and recursively follows the path until it gets the authoritative answer. It validates the DNSSEC chain to ensure the answer is genuine, caches it, and returns it to Pi-hole, which sends it to the device.

The Device: Receives the correct IP address and loads the website, completely unaware of the silent security check just performed.

Hardware Requirements
Item	Specification	Notes
Single-Board Computer	Raspberry Pi 3 or 4, or similar	The model 3 is perfectly sufficient for a home network.
MicroSD Card	16GB or larger, Class 10	The entire OS and logs comfortably fit here. 32GB is ideal for longevity.
Power Supply	5.1V, 2.5A minimum	A stable, official power supply prevents corruption and crashes.
Ethernet Cable	Cat5e or better	A wired connection is strongly recommended for a DNS server's reliability.
Case (Optional)	With heatsinks	Keeps the board cool and dust-free.
Software Requirements
Operating System: Raspberry Pi OS (Bookworm) Lite or Desktop, or a standard Linux distribution. This guide assumes a fresh, updated installation.

Pi-hole: The core ad-blocking application.

Unbound: The recursive DNS resolver.

A Computer: To flash the OS and access the device remotely.

Installation Guide
Step 1: Prepare the System
Flash the operating system onto the MicroSD card using the official imaging tool.

In the Imager's settings, enable SSH, set a username and password, and configure network connectivity if using wireless for setup.

Boot the device and connect via SSH or terminal.

Update the system:

bash
sudo apt update && sudo apt upgrade -y
Step 2: Set a Static IP Address
A DNS server must have a fixed address.

Critical Pre-Check: Avoid an IP conflict. Either reserve the desired IP in the router's DHCP reservation table using the device's MAC address, or ensure the router's DHCP pool does not include the chosen static address.

Method: Using NetworkManager (Default on Modern Linux Distributions)
Identify the connection name:

bash
nmcli con show
Look for the primary network connection (often named Wired connection 1 or preconfigured).

Apply the static configuration. Replace placeholders with your target values:

bash
sudo nmcli con mod "CONNECTION_NAME" ipv4.addresses STATIC_IP_ADDRESS/CIDR
sudo nmcli con mod "CONNECTION_NAME" ipv4.gateway ROUTER_IP_ADDRESS
sudo nmcli con mod "CONNECTION_NAME" ipv4.dns 1.1.1.1
sudo nmcli con mod "CONNECTION_NAME" ipv4.method manual
STATIC_IP_ADDRESS/CIDR: The intended fixed address with subnet mask (e.g., 192.168.1.100/24).

ROUTER_IP_ADDRESS: The router's internal IP (e.g., 192.168.1.1).

Activate the new settings:

bash
sudo nmcli con down "CONNECTION_NAME" && sudo nmcli con up "CONNECTION_NAME"
The SSH session will drop. Reconnect using the new static IP address.

Step 3: Install Pi-hole
Run the automated installer:

bash
curl -sSL https://install.pi-hole.net | bash
Navigate the text-based installer using Tab, Arrow Keys, and Spacebar.

Static IP Warning: Select Ok.

Interface Selection: Choose the Ethernet interface (eth0).

Upstream DNS Provider: Choose Custom. Enter 1.1.1.1 as a temporary placeholder.

Block Lists: Select Ok to use the default list.

Admin Web Interface: Select Yes and Ok.

Web Server: Select Yes and Ok.

Log Queries: Select Yes and Ok.

Privacy Mode: Select Show everything and Ok.

Save the output at the end. The "Installation Complete!" screen displays a randomly generated admin password and the URL for the web interface. Copy both immediately.

Step 4: Install and Configure Unbound
Install the package:

bash
sudo apt install unbound -y
Download the root server hints file:

bash
wget -O /var/lib/unbound/root.hints https://www.internic.net/domain/named.root
Create a dedicated configuration file:

bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
Paste the following configuration into the file:

yaml
server:
    port: 5335
    interface: 127.0.0.1
    access-control: 127.0.0.1/32 allow
    do-ip6: no
    root-hints: "/var/lib/unbound/root.hints"
    prefetch: yes
    rrset-cache-size: 100m
    msg-cache-size: 50m
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: yes
    edns-buffer-size: 1232
    cache-min-ttl: 300
    serve-expired: yes
    aggressive-nsec: yes
Save and exit (Ctrl+X, Y, Enter).

Enable and start the service:

bash
sudo systemctl restart unbound
sudo systemctl enable unbound
Test the local resolver:

bash
dig pi-hole.net @127.0.0.1 -p 5335
A status: NOERROR and an answer section with an IP address indicates success. Timeouts usually mean a configuration typo.

Step 5: Link Pi-hole to Unbound
Open a web browser and navigate to the Pi-hole admin console (http://YOUR_STATIC_IP/admin).

Log in using the saved admin password.

Go to Settings -> DNS.

Untick all pre-selected third-party upstream DNS servers.

Under Custom 1 (IPv4), enter: 127.0.0.1#5335

In the Advanced DNS settings section, untick:

Never forward non-FQDNs

Never forward reverse lookups for private IP ranges

Click Save at the bottom of the page.

Step 6: Configure Your Router
To enforce network-wide ad blocking, the router must give the Pi-hole's IP as the single DNS server to all clients.

Access the router's administration page.

Locate the DHCP Server or LAN Setup settings.

Find the DNS Server fields. They are often under DHCP settings.

Set the Primary DNS to the Pi-hole's static IP address.

Set the Secondary DNS field to blank or 0.0.0.0. If the interface forces a secondary entry, the only way to guarantee no ad leakage is to set the Pi-hole's IP again. Using a public server like 8.8.8.8 will allow devices to bypass the Pi-hole.

Apply or Save the configuration.

To propagate the change, devices need to reconnect to the network or renew their DHCP lease.

Verification and Testing
Pi-hole Dashboard: Visit the admin console. The dashboard should show an increasing number of total queries and a blocked percentage greater than 0%.

Terminal Test: Run an nslookup on a known ad domain from another machine:

bash
nslookup doubleclick.net
The returned address should be 0.0.0.0, confirming the sinkhole is working.

Browser Test: Browse to an ad-heavy website. You should see empty spaces where banner ads and pop-ups used to be.

Leak Test: Visit an online DNS leak test website. It should show only one DNS server, belonging to no commercial provider, confirming Unbound is resolving directly.

Maintenance and Updates
Regular maintenance ensures the system is secure, performs well, and has up-to-date blocklists.

Update the Operating System:

bash
sudo apt update && sudo apt upgrade -y
Update Pi-hole:

bash
pihole -up
Update Gravity (Blocklist Database):

bash
pihole -g
This can be automated. Pi-hole has a built-in weekly cron job that updates Gravity by default.

Troubleshooting
Problem	Possible Solution
No device is blocking ads.	Verify the router's DHCP settings are pointing all DNS fields to the Pi-hole's IP. Ensure devices have reconnected to the network.
Pi-hole dashboard shows no queries.	The router may still be advertising itself as the DNS server. Double-check the DHCP settings. Some routers segregate "WAN DNS" and "DHCP DNS"; only the latter matters for clients.
Unbound fails to start.	Check the config file for syntax errors: sudo unbound-checkconf. Verify that /etc/unbound/unbound.conf.d/pi-hole.conf contains no typos.
Websites load slowly initially.	This is expected for non-cached domains. Unbound is performing full recursion from the root, which takes longer than a single forwarded request. Subsequent loads will be fast.
Specific sites are broken.	A blocklist may be too aggressive. Check the Pi-hole query log as you load the site, find the blocked domain, and add it to the Whitelist in the admin console.
Architecture Diagram
text
    +---------------------------+      +---------------------------+
    |        Your Devices        |      |        The Internet       |
    |  (Phone, Laptop, TV, etc.)  |      |                           |
    +-------------+-------------+      +-------------+-------------+
                  |                                   ^
                  | DNS Request for site.com          | Authoritative Answer
                  v                                   |
    +-------------+-------------+      +-------------+-------------+
    |         Pi-hole           |      |      Root DNS Servers      |
    |  (Sinkhole / Gatekeeper)   |      |  (.com, .net, .org, etc.) |
    +-------------+-------------+      +-------------+-------------+
                  |  If not blocked                ^
                  |  & not cached                  |
                  v                                |
    +-------------+-------------+                  |
    |         Unbound           +------------------+
    |  (Recursive, Validating    |   Recursive lookup from root
    |   Caching Resolver)       |
    +---------------------------+
The query flow: Device -> Pi-hole -> Unbound -> Root Servers -> Website

Glossary
DNS (Domain Name System): The phonebook of the internet that translates human-readable domain names (like www.example.com) into machine-readable IP addresses.

DNS Sinkhole: A method of blocking malicious or unwanted domains by returning a false or null result for DNS queries.

Recursive Resolver: A DNS server that does the full legwork of a lookup, starting from the internet's root servers, for each query it doesn't have cached.

Forwarding Resolver: A DNS server that simply passes queries to another upstream resolver (like 8.8.8.8) and relays the answer.

DNSSEC: A suite of extensions that adds cryptographic signatures to DNS records, ensuring they haven't been tampered with in transit.

Gravity: Pi-hole's term for its compiled, local domain blocklist database.

DHCP: The protocol a router uses to automatically assign IP addresses and network settings to devices when they connect.
