# DNS Server Configuration Guide - BIND on Fedora

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Main Configuration](#main-configuration)
5. [DNS Zone Configuration](#dns-zone-configuration)
6. [Verification and Testing](#verification-and-testing)
7. [Troubleshooting](#troubleshooting)

---

## Overview

This guide provides step-by-step instructions for configuring a DNS server using BIND (Berkeley Internet Name Domain) on Fedora or CentOS. The server will manage the `ensa.ac.ma` domain with the following infrastructure example:

| Service | Hostname | IP Address |
|---------|----------|------------|
| DNS Server | ns1.ensa.ac.ma | 192.168.1.132 |
| Web Server | www.ensa.ac.ma | 192.168.1.134 |
| Proxy Server | proxy.ensa.ac.ma | 192.168.1.135 |
| LDAP Server | ldap.ensa.ac.ma | 192.168.1.136 |

**Network:** 192.168.1.0/24

---

## Prerequisites

Before starting, ensure you have:
- Root or sudo access to the server
- Rocky Linux, CentOS, Fedora system
- Static IP address configured (192.168.1.132)
- Internet connectivity for package installation
- Basic understanding of DNS concepts

---

## Installation

### Step 1: Install BIND and Utilities

Install the BIND DNS server and associated utilities:

```bash
sudo dnf install bind bind-utils -y
```

**Packages installed:**
- `bind`: The DNS server daemon (named)
- `bind-utils`: DNS diagnostic tools (dig, nslookup, host)

### Step 2: Verify Installation

Check that BIND is installed correctly:

```bash
named -v
```

This should display the BIND version number.

---

## Main Configuration

### Step 3: Configure named.conf

The main configuration file `/etc/named.conf` controls the global behavior of the DNS server.

Open the configuration file:

```bash
sudo nano /etc/named.conf
```

Replace the entire content with the following configuration:

```bind
options {
    listen-on port 53 { 127.0.0.1; 192.168.1.132; };
    listen-on-v6 port 53 { ::1; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query     { localhost; 192.168.1.0/24; };
    recursion yes;
    dnssec-validation auto;
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
};

zone "." IN {
    type hint;
    file "named.ca";
};

// Forward zone for ensa.ac.ma
zone "ensa.ac.ma" IN {
    type master;
    file "ensa.ac.ma.zone";
    allow-update { none; };
};

// Reverse zone
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "1.168.192.rev";
    allow-update { none; };
};

include "/etc/named.rfc1912.zones";
```

#### Configuration Breakdown

**Options Section:**
- `listen-on port 53`: Specifies which IP addresses the server listens on (localhost and 192.168.1.132)
- `allow-query`: Restricts DNS queries to localhost and the 192.168.1.0/24 network
- `recursion yes`: Allows the server to perform recursive queries for clients
- `dnssec-validation auto`: Automatically validates DNSSEC signatures

**Zone Declarations:**
- **Root zone (.)**: Uses root hints for internet DNS resolution
- **Forward zone (ensa.ac.ma)**: Maps hostnames to IP addresses
- **Reverse zone (1.168.192.in-addr.arpa)**: Maps IP addresses back to hostnames

Save and exit the file (Ctrl+X, then Y, then Enter in nano).

---

## DNS Zone Configuration

### Step 4: Create the Forward Zone File

The forward zone file maps hostnames to IP addresses.

Create the zone file:

```bash
sudo nano /var/named/ensa.ac.ma.zone
```

Add the following content:

```dns
$TTL 86400
@       IN      SOA     ns1.ensa.ac.ma. admin.ensa.ac.ma. (
                        2025012001      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL

; Name servers
@       IN      NS      ns1.ensa.ac.ma.

; A records
ns1     IN      A       192.168.1.132
www     IN      A       192.168.1.134
web     IN      A       192.168.1.134
proxy   IN      A       192.168.1.135
squid   IN      A       192.168.1.135
ldap    IN      A       192.168.1.136
ldapserver IN   A       192.168.1.136

; Wildcard for all subdomains
*       IN      A       192.168.1.134
```

#### Zone File Explanation

**SOA Record (Start of Authority):**
- `ns1.ensa.ac.ma.`: Primary nameserver for this zone
- `admin.ensa.ac.ma.`: Email address of the zone administrator (admin@ensa.ac.ma)
- `Serial`: Version number (format: YYYYMMDDNN) - increment when making changes
- `Refresh`: Time (in seconds) secondary DNS servers should check for updates
- `Retry`: Time to wait before retrying a failed refresh
- `Expire`: Maximum time secondary servers should keep zone data without refresh
- `Minimum TTL`: Default time-to-live for zone records

**NS Record:**
- Declares `ns1.ensa.ac.ma` as the authoritative nameserver

**A Records:**
- Map hostnames to IPv4 addresses
- Multiple hostnames can point to the same IP (aliases)

**Wildcard Record (*):**
- Catches all undefined subdomains and directs them to 192.168.1.134

### Step 5: Create the Reverse Zone File

The reverse zone file allows reverse DNS lookups (IP to hostname).

Create the reverse zone file:

```bash
sudo nano /var/named/1.168.192.rev
```

Add the following content:

```dns
$TTL 86400
@       IN      SOA     ns1.ensa.ac.ma. admin.ensa.ac.ma. (
                        2025012001      ; Serial
                        3600            ; Refresh
                        1800            ; Retry
                        604800          ; Expire
                        86400 )         ; Minimum TTL

@       IN      NS      ns1.ensa.ac.ma.

132     IN      PTR     ns1.ensa.ac.ma.
134     IN      PTR     www.ensa.ac.ma.
135     IN      PTR     proxy.ensa.ac.ma.
136     IN      PTR     ldap.ensa.ac.ma.
```

#### Reverse Zone Explanation

**PTR Records:**
- Map IP addresses (last octet only) to fully qualified domain names
- Format: `[last-octet] IN PTR [fqdn]`
- Note: Only one PTR record per IP is typically used (removed duplicates for 134, 135, 136)

### Step 6: Set Correct Permissions

BIND requires specific ownership and permissions for zone files:

```bash
sudo chown named:named /var/named/ensa.ac.ma.zone
sudo chown named:named /var/named/1.168.192.rev
sudo chmod 644 /var/named/ensa.ac.ma.zone
sudo chmod 644 /var/named/1.168.192.rev
```

**Permissions Breakdown:**
- Owner: `named` user and group (the BIND service user)
- Mode: 644 (read/write for owner, read-only for others)

---

## Verification and Testing

### Step 7: Validate Configuration

Before starting the service, validate all configuration files:

**Check main configuration:**
```bash
sudo named-checkconf
```
- No output means the configuration is valid
- Any errors will be displayed with line numbers

**Check forward zone:**
```bash
sudo named-checkzone ensa.ac.ma /var/named/ensa.ac.ma.zone
```
Expected output:
```
zone ensa.ac.ma/IN: loaded serial 2025012001
OK
```

**Check reverse zone:**
```bash
sudo named-checkzone 1.168.192.in-addr.arpa /var/named/1.168.192.rev
```
Expected output:
```
zone 1.168.192.in-addr.arpa/IN: loaded serial 2025012001
OK
```

### Step 8: Enable and Start the Service

Enable BIND to start automatically on boot:
```bash
sudo systemctl enable named
```

Start the BIND service:
```bash
sudo systemctl start named
```

Check the service status:
```bash
sudo systemctl status named
```

You should see `active (running)` in green.

### Step 9: Configure Firewall

Open DNS port (53) in the firewall:

```bash
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

Verify the firewall rule:
```bash
sudo firewall-cmd --list-services
```

### Step 10: Test DNS Resolution

**Test forward lookup:**
```bash
dig @192.168.1.132 www.ensa.ac.ma
```

Expected output should include:
```
;; ANSWER SECTION:
www.ensa.ac.ma.     86400   IN  A   192.168.1.134
```

**Test reverse lookup:**
```bash
dig @192.168.1.132 -x 192.168.1.134
```

Expected output should include:
```
;; ANSWER SECTION:
134.1.168.192.in-addr.arpa. 86400 IN PTR www.ensa.ac.ma.
```

**Test wildcard subdomain:**
```bash
dig @192.168.1.132 test.ensa.ac.ma
```

Should resolve to 192.168.1.134.

**Alternative testing with nslookup:**
```bash
nslookup www.ensa.ac.ma 192.168.1.132
nslookup ldap.ensa.ac.ma 192.168.1.132
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Service Fails to Start

**Check logs:**
```bash
sudo journalctl -xeu named
sudo tail -f /var/log/messages
```

**Common causes:**
- Syntax errors in configuration files
- Permission issues on zone files
- Port 53 already in use by another service

#### 2. Configuration Syntax Errors

Run validation commands again:
```bash
sudo named-checkconf
sudo named-checkzone ensa.ac.ma /var/named/ensa.ac.ma.zone
```

Pay attention to:
- Missing semicolons
- Incorrect IP addresses
- Mismatched parentheses in SOA records

#### 3. DNS Queries Not Working

**Verify the service is running:**
```bash
sudo systemctl status named
```

**Check if port 53 is listening:**
```bash
sudo ss -tulpn | grep :53
```

**Verify firewall rules:**
```bash
sudo firewall-cmd --list-all
```

**Test locally first:**
```bash
dig @localhost www.ensa.ac.ma
```

#### 4. Zone File Changes Not Reflected

After modifying zone files:
1. Increment the serial number in the SOA record
2. Reload the service:
```bash
sudo systemctl reload named
```

Or restart:
```bash
sudo systemctl restart named
```

#### 5. Permission Denied Errors

Ensure correct ownership and permissions:
```bash
ls -l /var/named/ensa.ac.ma.zone
ls -l /var/named/1.168.192.rev
```

Both files should show:
```
-rw-r--r--. 1 named named [size] [date] [filename]
```

### Useful Commands

**View BIND logs:**
```bash
sudo journalctl -u named -f
```

**Reload configuration without restarting:**
```bash
sudo rndc reload
```

**Check BIND version:**
```bash
named -v
```

**Query specific record types:**
```bash
dig @192.168.1.132 ensa.ac.ma NS
dig @192.168.1.132 ensa.ac.ma SOA
dig @192.168.1.132 www.ensa.ac.ma A
```

---

## Configuration for Client Machines

To use this DNS server, configure client machines with:

**DNS Server:** 192.168.1.132

**On Linux clients:**
Edit `/etc/resolv.conf`:
```
nameserver 192.168.1.132
```

**On Windows clients:**
Set DNS server in network adapter properties to `192.168.1.132`

---

## Security Recommendations

1. **Restrict queries:** The current configuration allows queries from 192.168.1.0/24 only
2. **Disable recursion for public zones:** If this DNS server will be accessible from the internet
3. **Implement DNSSEC:** For enhanced security (requires additional configuration)
4. **Regular updates:** Keep BIND updated with security patches
5. **Monitor logs:** Regularly check `/var/log/messages` and `journalctl` for suspicious activity
6. **Rate limiting:** Consider implementing rate limiting to prevent DNS amplification attacks

---

## Maintenance Tasks

### Updating Zone Files

When adding or modifying records:

1. Edit the zone file:
```bash
sudo nano /var/named/ensa.ac.ma.zone
```

2. **Increment the serial number** (very important!)

3. Validate the changes:
```bash
sudo named-checkzone ensa.ac.ma /var/named/ensa.ac.ma.zone
```

4. Reload the service:
```bash
sudo systemctl reload named
```

### Monitoring

**Check service status regularly:**
```bash
sudo systemctl status named
```

**Monitor query statistics:**
```bash
sudo rndc stats
cat /var/named/data/named_stats.txt
```

---

## Conclusion

You now have a fully functional DNS server managing the `ensa.ac.ma` domain. This server provides:
- Forward DNS resolution (hostname to IP)
- Reverse DNS resolution (IP to hostname)
- Wildcard subdomain support
- Recursive queries for clients on the local network

