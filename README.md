# 🐳 Docker: Container Networking

**Objective:** Master Docker container networking fundamentals through hands-on demonstration

---

## 📋 Table of Contents

- [🔧 Section 1: Docker Networks Fundamentals](#section-1-docker-networks-fundamentals)
- [🌐 Section 2: Default vs Custom Networks](#section-2-default-vs-custom-networks)
- [⚙️ Section 3: Creating and Managing Networks](#section-3-creating-and-managing-networks)
- [🔍 Section 4: Container Network Exploration](#section-4-container-network-exploration)
- [🛠️ Section 5: Network Discovery and Testing](#section-5-network-discovery-and-testing)
- [🔀 Section 6: Multi-Network Architecture](#section-6-multi-network-architecture)
- [🌐 Section 7: Port Publishing and External Access](#section-7-port-publishing-and-external-access)
- [🏗️ Section 8: Multi-Container Applications](#section-8-multi-container-applications)
- [📝 Section 9: Key Concepts and Best Practices](#section-9-key-concepts-and-best-practices)
- [🧹 Section 10: Cleanup](#section-10-cleanup)

---

## Section 1: Docker Networks Fundamentals

### What Are Docker Networks?

Docker networks are virtual networking layers that enable containers to communicate with each other and external systems. They function as virtual switches connecting containers while maintaining isolation and security.

### Network Drivers

**Bridge Driver:** Creates isolated networks on a single host (default)
**Host Driver:** Containers use the host's network directly (no isolation)
**None Driver:** Completely isolates containers from any network

### Basic Commands

**List all networks:**
```bash
sudo docker network ls
```

**Inspect network details:**
```bash
sudo docker network inspect bridge
```

**Expected default networks:**
- `bridge`: Default network for containers
- `host`: Direct host network access
- `none`: Network isolation mode

---

## Section 2: Default vs Custom Networks

### Default Bridge Limitations - Practical Demonstration

Let's see these limitations in action:

**Create containers on the default bridge:**
```bash
sudo docker run -d --name default-app1 alpine:3.8 sleep 300
sudo docker run -d --name default-app2 alpine:3.8 sleep 300
```

**Test 1: No DNS Resolution**
```bash
sudo docker exec default-app1 ping -c 3 default-app2
# Expected: "ping: bad address 'default-app2'" - NAME RESOLUTION FAILS
```

**Test 2: Must Use IP Addresses**
```bash
# Get IP of default-app2
sudo docker inspect default-app2 | grep IPAddress
# Use the IP (e.g., 172.17.0.3) instead of name
sudo docker exec default-app1 ping -c 3 172.17.0.3
# Expected: SUCCESS - but only because we used IP, not name
```

**Test 3: Single Flat Network**
```bash
sudo docker network inspect bridge
# Shows ALL containers share same subnet (usually 172.17.0.0/16)
# No network segmentation possible
```

**Test 4: Limited Security Isolation**
```bash
# ALL containers on default bridge can reach each other
# No way to isolate "database" from "web" containers
# Security relies only on application-level controls
```

### Custom Networks Advantages - Practical Demonstration

**Create custom network and containers:**
```bash
sudo docker network create --subnet 10.0.50.0/24 demo-network
sudo docker run -d --name custom-app1 --network demo-network alpine:3.8 sleep 300
sudo docker run -d --name custom-app2 --network demo-network alpine:3.8 sleep 300
```

**Test 1: DNS Resolution Works**
```bash
sudo docker exec custom-app1 ping -c 3 custom-app2
# Expected: SUCCESS - name resolves automatically
```

**Test 2: Network Isolation**
```bash
# Containers on different custom networks cannot communicate
sudo docker network create --subnet 10.0.51.0/24 isolated-network
sudo docker run -d --name isolated-app --network isolated-network alpine:3.8 sleep 300

sudo docker exec custom-app1 ping -c 3 isolated-app
# Expected: "ping: bad address 'isolated-app'" - NETWORK ISOLATION WORKS
```

**Test 3: Security Segmentation**
```bash
# Can create separate networks for different application tiers
sudo docker network create --subnet 10.0.52.0/24 database-network
sudo docker network create --subnet 10.0.53.0/24 web-network
# Database containers are isolated from web containers by default
```

### Why This Matters

| Feature | Default Bridge | Custom Networks |
|---------|----------------|-----------------|
| **Container Communication** | IP addresses only | Names + IP addresses |
| **Security** | All containers can reach each other | Network-level isolation |
| **Scalability** | Single flat network | Multiple isolated networks |
| **Service Discovery** | Manual IP management | Automatic DNS resolution |
| **Production Ready** | ❌ Not recommended | ✅ Industry standard |

---

## Section 3: Creating and Managing Networks

### Understanding Subnets

**Subnet notation:** `10.0.42.0/24` means:
- 256 possible IP addresses (10.0.42.0 to 10.0.42.255)
- `/24` = 24 bits for network, 8 bits for hosts
- Provides clear network boundaries

**IP Range:** `10.0.42.128/25` limits container IPs to 128 addresses (10.0.42.128 to 10.0.42.255), reserving the lower half for other purposes.

### Create Custom Network

```bash
sudo docker network create \
  --driver bridge \
  --subnet 10.0.42.0/24 \
  --ip-range 10.0.42.128/25 \
  network-A
```

**Parameter explanations:**
- `--driver bridge`: Bridge networking for same-host containers
- `--subnet 10.0.42.0/24`: Network's IP address space
- `--ip-range 10.0.42.128/25`: Restricts container IP allocation
- `network-A`: Descriptive network name

---

## Section 4: Container Network Exploration

### Launch Container on Custom Network

```bash
sudo docker run -it \
  --network network-A \
  --name container1 \
  alpine:3.8 \
  sh
```

**Parameter explanations:**
- `-it`: Interactive terminal (`-i` interactive + `-t` pseudo-TTY)
- `--network network-A`: Connects to custom network
- `--name container1`: Assigns memorable name
- `alpine:3.8`: Lightweight Linux (5MB), industry standard for testing
- `sh`: Starts shell for exploration

### Analyze Network Interfaces

**Inside container, check IP configuration:**
```bash
ip -f inet -4 -o addr
```

**Parameter breakdown:**
- `ip`: Modern Linux networking tool
- `-f inet`: Filter for internet protocol family
- `-4`: IPv4 addresses only
- `-o`: One-line output format
- `addr`: Display IP addresses

**Expected output:**
```
1: lo    inet 127.0.0.1/8 scope host lo
6: eth0  inet 10.0.42.129/24 brd 10.0.42.255 scope global eth0
```

---

## Section 5: Network Discovery and Testing

### Install Network Tools

```bash
apk update && apk add nmap
```

**Command breakdown:**
- `apk`: Alpine Package Keeper
- `update`: Refresh package repository
- `&&`: Run second command only if first succeeds
- `add nmap`: Install Network Mapper for discovery

### Network Scanning

```bash
nmap -sn 10.0.42.* -oG /dev/stdout | grep Status
```

**Parameter explanations:**
- `nmap`: Network mapping tool
- `-sn`: Ping scan (checks host availability without port scanning)
- `10.0.42.*`: Scans entire subnet (10.0.42.1-254)
- `-oG /dev/stdout`: Greppable output to terminal
- `| grep Status`: Filter to show only host status

**Detach from container:** `Ctrl+P` then `Ctrl+Q`

---

## Section 6: Multi-Network Architecture

### Create Second Network

```bash
sudo docker network create --subnet 10.0.43.0/24 network-B
```

### Connect Container to Multiple Networks

```bash
sudo docker network connect network-B container1
```

Now container1 has interfaces on both networks and can communicate with containers on either network.

### Test Multi-Network Communication

**Create container on second network:**
```bash
sudo docker run -d \
  --name container2 \
  --network network-B \
  alpine:3.8 \
  sleep 300
```

**Parameter explanations:**
- `-d`: Detached mode (background)
- `sleep 300`: Keeps container running for 5 minutes (prevents immediate exit)

**Test connectivity:**
```bash
sudo docker attach container1
ping -c 5 container2
```

**Ping parameters:**
- `-c 5`: Send exactly 5 packets (professional testing approach)
- `container2`: Target resolved via Docker DNS

**Testing Network Isolation:**
```bash
# Test failed connectivity - containers on different networks
sudo docker run -d --name isolated-container --network bridge alpine:3.8 sleep 300
ping -c 3 isolated-container
# Expected: "ping: bad address 'isolated-container'" - DNS resolution fails

# Test non-existent container
ping -c 3 nonexistent-container
# Expected: "ping: bad address 'nonexistent-container'"
```

**What This Demonstrates:**
- **Successful pings:** Network connectivity and DNS resolution are working
- **Failed pings:** Network isolation prevents cross-network communication
- **DNS resolution:** Container names automatically resolve to IP addresses within the same network

---

## Section 7: Port Publishing and External Access

### Port Mapping Concepts

**Format:** `host_port:container_port`
- Maps host port to container port
- Enables external access to containerized services
- Allows multiple containers running the same service on different ports

### Create Container with Published Port

```bash
sudo docker run -d \
  -p 8080:8080 \
  --name webserver \
  alpine:3.8 \
  sleep 300
```

**Port publishing breakdown:**
- `-p 8080:8080`: Maps host port 8080 to container port 8080
- First 8080: Port on the host machine
- Second 8080: Port inside the container

**Verify port mapping:**
```bash
sudo docker port webserver
```

**Security tip:** Use specific host IPs: `-p 127.0.0.1:8080:8080`

---

## Section 8: Multi-Container Applications

### Create Application Containers

```bash
sudo docker run -d --name app1 --network network-A alpine:3.8 sleep 300
sudo docker run -d --name app2 --network network-A alpine:3.8 sleep 300
```

### Test Inter-Container Communication

**Test containers on the same network:**
```bash
sudo docker exec app1 ping -c 5 app2
```

**Command breakdown:**
- `docker exec`: Execute a command in a running container
- `app1`: Source container
- `ping -c 5 app2`: Test connectivity and DNS resolution

**Test network isolation:**
```bash
# Create container on different network
sudo docker run -d --name isolated-app --network bridge alpine:3.8 sleep 300

# This should fail - different networks
sudo docker exec app1 ping -c 3 isolated-app
# Expected: "ping: bad address 'isolated-app'" - DNS resolution fails
```

**What This Demonstrates:**
- **Same network:** Automatic service discovery and connectivity
- **Different networks:** Network isolation prevents communication
- **DNS resolution:** Only works within the same custom network

---

## Section 9: Key Concepts and Best Practices

### Essential Networking Principles

**Custom Networks Benefits:**
1. **Name Resolution:** Containers communicate by name, not IP
2. **Isolation:** Better security through network separation
3. **Flexibility:** Multiple networks per container support complex architectures
4. **Scalability:** Easy addition/removal of containers

**Production Security:**
- Use custom networks instead of default bridge
- Minimize published ports: `-p 127.0.0.1:8080:8080`
- Implement network segmentation for sensitive applications
- Plan IP ranges to avoid conflicts

**Performance Tips:**
- Use appropriate network drivers for your use case
- Consider container placement for minimal network hops
- Monitor network traffic and optimize as needed

### Key Definitions

**Bridge Network:** Virtual switch connecting containers on a single host
**Container DNS:** Automatic name-to-IP resolution within custom networks
**Network Namespace:** Isolated network stack per container
**Port Publishing:** Mapping host ports to container ports for external access
**Subnet:** Network segment defined by IP range
**IPAM:** IP Address Management system for address allocation

---

## Section 10: Cleanup

### Remove All Resources

```bash
# Stop all running containers
sudo docker stop $(sudo docker ps -q)

# Remove unused containers, networks, images
sudo docker system prune -f
```

**Command explanations:**
- `docker ps -q`: Lists only container IDs
- `$()`: Command substitution
- `system prune -f`: Force cleanup without confirmation

---

## 📚 References

### Official Documentation
- [Docker Network Overview](https://docs.docker.com/network/)
- [Docker Network Commands](https://docs.docker.com/engine/reference/commandline/network/)
- [Bridge Network Driver](https://docs.docker.com/network/bridge/)

### Tools Used
- **Docker Engine** - Container runtime and networking
- **Alpine Linux** - Lightweight container base (5MB)
- **nmap** - Network discovery tool
- **iproute2** - Modern Linux networking utilities

---

**Created by:** [Iqra Ali]  
**Last Updated:** July 2025  
**Version:** 1.0.0
