# Pi-hole + Unbound (recursive DNS) + Redis setup

## Official documentation:
* More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
* Unbound info: https://github.com/madnuttah/unbound-docker

---

## Purpose of This Repository

While searching for a complete setup guide for Pi-hole, Unbound, and Redis in Docker, I couldn't find one that fit my needs.
This repository consolidates information from various sources into a tested setup for both host and bridge network modes on a Raspberry Pi 4 (RPI4). Configuration files are optimized for RPI4 hardware but can be adjusted as needed.

---

## Additional considerations

* Nginx Proxy Manager (NPM): If you are using NPM on the same device, some configurations direct DNS queries to its IP for efficient local resolution.
* Network Mode Choice:
  * host mode allows direct DNS resolution on the local network.
  * bridge mode isolates services within Docker, meaning Pi-hole won’t directly receive requests from your network unless explicitly forwarded.
  * Note: When using bridge mode, configure your router's DHCP settings to assign Pi-hole as the DNS server.

---

## Prerequisites & Setup

### Step 1: Directory Structure

Before deploying the stack, create the required directory structure. The easiest way is to extract the provided tar.gz file to your preferred location.

1. Download and extract
`tar -xvzf pihole-unbound-redis.tar.gz -C /opt/docker`
2. Optionally, rename the parent directory but ensure paths match the bind mounts in `docker-compose.yml`
3. Verify the extracted structure:
```
pihole-unbound-redis
    ├── etc-dnsmasq.d
    │   └── 01-pihole-wildcard-dns.conf
    ├── etc-pihole
    ├── redis
    │   ├── data
    │   ├── healthcheck.sh
    │   └── redis.conf
    └── unbound
        ├── cachedb.d
        ├── certs.d
        ├── conf.d
        │   ├── access-control.conf
        │   ├── cachedb.conf
        │   ├── interfaces.conf
        │   ├── logging.conf
        │   ├── performance.conf
        │   ├── remote-control.conf
        │   ├── security.conf
        │   └── trust-anchor.conf
        ├── iana.d
        │   ├── root.key
        │   └── root.zone
        ├── log.d
        │   └── unbound.log
        ├── sbin
        │   └── healthcheck.sh
        ├── unbound.conf
        └── zones.d
            └── auth-zone.conf
```

### Step 2: Network Modes
 
* **Host Mode:** Best for local network DNS resolution. Containers share the host's network stack, meaning Pi-hole directly listens on the machine’s IP.
* **Bridge Mode:** Containers are isolated with their own network. You must manually set your router's DNS relay to point to the bridge network's Pi-hole IP or use a DHCP relay software option.

---

### Step 3: Deploy the stack

#### Option 1: Using Command Line

1. Navigate to the extracted directory
`cd /opt/docker/pihole-unbound-redis`
2. Choose the appropriate compose file

   **Host Mode:**
   `docker-compose -f docker-compose-host.yml up -d`

   **Bridge Mode:**
   `docker-compose -f docker-compose-bridge.yml up -d`

#### Option 2: Using Portainer Stacks

1. Open Portainer and go to Stacks.
2. Click Add Stack.
3. Enter a stack name (e.g., pihole-unbound).
4. In the Web editor, copy-paste the content of the desired docker-compose.yml file.
5. Under Environment Variables, add any required variables.
6. Click Deploy the Stack and wait for it to finish.
7. Once deployed, check the logs using Containers > [Container Name] > Logs for troubleshooting.

---

## Configuration Files

### Wildcard DNS Resolution

To avoid manually adding each subdomain in Pi-hole, modify the following file (existing in tar.gz):
`/opt/docker/pihole-unbound-redis/etc-dnsmasq.d/01-pihole-wildcard-dns.conf`

Content:

```
address=/.example.com/10.0.0.1
address=/.domain.com/10.0.0.2
```

These IPs can point to services like Nginx Proxy Manager (NPM) or other internal services. This setup ensures that all subdomains automatically resolve to the assigned IP without needing individual DNS records.

⚠️ Note: These wildcard entries will not appear in the Pi-hole Web GUI under "Local DNS Records" since they are configured at the dnsmasq level.

⚠️ Note 1: When using custom conf files under dnsmasq.d , environment variable must be set in compose file **FTLCONF_misc_etc_dnsmasq_d: 'true'** (already set in both bridge and host mode variants, this is just as FYI)

---

### Access Control Configuration

**Host Mode** (access-control.conf)

```
server:
  access-control: 127.0.0.1 allow  # Allow only Pi-hole to query Unbound
  access-control: fc00::/7 deny
  access-control: ::1/128 deny
  access-control: ::0/0 deny
  access-control: ::ffff:127.0.0.1 deny
```

* In host mode, Unbound will only accept queries from the Pi-hole instance running on the same machine.
* All other requests are denied, ensuring Unbound is not exposed to external queries.

**Bridge Mode** (access-control.conf)

```
server:
  access-control: 172.23.0.7 allow  # Update to Pi-hole container’s bridge IP
  access-control: fc00::/7 deny
  access-control: ::1/128 deny
  access-control: ::0/0 deny
  access-control: ::ffff:127.0.0.1 deny
```

* In bridge mode, Pi-hole runs in an isolated Docker network and communicates with Unbound via its container IP.
* The IP 172.23.0.7 must be manually updated based on the actual bridge network configuration.
* All external queries are denied, restricting access to only Pi-hole.

---

### Performance Optimization (performance.conf)

This configuration optimizes Unbound for an Raspberry Pi 4 (RPI4) but can be adjusted for different hardware.

```
server:
  num-threads: 2
  num-queries-per-thread: 2048
  cache-max-ttl: 86400
  cache-min-ttl: 30
  edns-buffer-size: 1232
  rrset-roundrobin: yes
  neg-cache-size: 2M
  delay-close: 1000
  rrset-cache-size: 128m
  rrset-cache-slabs: 2
  ratelimit: 500
  unwanted-reply-threshold: 10000
  infra-cache-slabs: 2
  infra-cache-numhosts: 50000
  msg-cache-size: 128m
  msg-cache-slabs: 2
  key-cache-size: 2m
  key-cache-slabs: 2
  prefetch: yes
  prefetch-key: yes
  serve-expired: no
  max-udp-size: 4096
  msg-buffer-size: 32768
  stream-wait-size: 2m
  outgoing-range: 8192
  outgoing-port-permit: 32768
```

**Explanation**
* `num-threads`: 2 → Uses two threads, optimal for RPI4’s 4-core CPU. Adjust based on available cores.
* `cache-max-ttl`: 86400 → Maximum caching time for DNS records (1 day).
* `cache-min-ttl`: 30 → Minimum caching time (30 seconds) to avoid excessive upstream queries.
* `edns-buffer-size`: 1232 → Optimized for modern networks, balancing performance and compatibility.
* `rrset-cache-size`: 128m → Increases memory for cached DNS responses to improve performance.
* `ratelimit`: 500 → Prevents abuse by limiting requests per second.
* `infra-cache-numhosts`: 50000 → Expands the infrastructure cache to handle more queries efficiently.
* `prefetch`: yes & prefetch-key: yes → Prefetches frequently accessed domains to reduce query latency.

**Customization**

Modify this file based on your hardware:
* CPU Cores: Increase num-threads if using a more powerful system.
* RAM: Adjust cache sizes (e.g., rrset-cache-size, msg-cache-size) based on available memory.
* Network Speed: Lower delay-close for fast networks to reduce query latency.
* Expected Query Load: Increase outgoing-range for high-query environments.

Or you can feed teh existing performance.conf file into an AI model and ask it to modofy according to your HW specs

---

### DNS IP Propagation in Bridge Mode

When running Pi-hole with Unbound and Redis in bridge mode using Docker, the containers' IP addresses (e.g., 172.X.X.X) will be pushed to clients.
However, clients will not be able to resolve DNS queries without manual configuration, as the DNS settings need to be broadcast to the network.

**Configuring DNS IP Broadcast**

To ensure DNS resolution works correctly for all clients, follow these steps:

1. Create a custom DNS configuration file:
Under your persistent bind mount (e.g., /opt/docker/pihole-unbound-redis/etc-dnsmasq.d), create a new configuration file named 02-pihole-host-ip-dns-broadcast.conf (or any name you prefer). Add the following content:
`dhcp-option=6,10.0.0.3,10.0.0.3,10.0.0.3`

2. Modify dnsmasq.conf (if needed):
If you're using a custom directory (/etc/dnsmasq.d/) for configurations, you'll need to remove the default dhcp-option=option:dns-server entry from dnsmasq.conf to avoid conflicts.

Alternatively, you can modify the existing entry in /opt/docker/pihole-unbound-redis/etc-pihole/dnsmasq.conf to point to the correct DNS IP and add multiple DNS entries if required:
`dhcp-option=option:dns-server,10.0.0.3,10.0.0.3,10.0.0.3`

Replace 10.0.0.3 with the primary DNS IP of your Pi-hole container. You can also add a secondary DNS IP (e.g., 10.0.0.4).

3. Restart Docker Compose:
After creating or modifying the configuration, start your Docker Compose stack once. Then, make any necessary changes to the files, as they should be applied during the next restart.

⚠️ Note: When using custom conf files under dnsmasq.d , environment variable must be set in compose file **FTLCONF_misc_etc_dnsmasq_d: 'true'** (already set in both bridge and host mode variants, this is just as FYI)

---
