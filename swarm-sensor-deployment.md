# Deploying a GreyNoise Swarm Sensor on DigitalOcean

This document outlines my deployment of a GreyNoise Swarm community sensor on a minimal DigitalOcean Droplet. It details the infrastructure requirements, network hardening considerations, and the strategic rationale for selecting the **Fortinet FortiOS** emulation profile to maximize threat intelligence collection.

## 1. What is GreyNoise Swarm?
GreyNoise Swarm is a distributed community honeypot network. Unlike legacy honeypots (such as Cowrie or Dionaea) that require heavy local containerization to emulate services, Swarm operates as a lightweight, native Linux daemon utilizing eBPF and network tunneling. 

Raw packets hitting the sensor are captured at the socket layer and tunneled securely back to GreyNoise’s cloud edge, where high-interaction service emulation actually occurs. This architecture allows the sensor to run on exceptionally low-resource hardware.

---

## 2. Infrastructure Setup (DigitalOcean Droplet)

I deployed the sensor on DigitalOcean's smallest available Droplet tier, which is more than sufficient for the Swarm binary.

### Hardware Specifications
* **vCPU:** 1
* **RAM:** 512 MB
* **Storage:** 10 GB SSD
* **Network:** Dedicated Public IPv4 Address (Crucial for receiving unsolicited background scanning)


### Installation
Updating the Ubuntu server was not necessary, but was done out of habit
```bash
sudo apt update && sudo apt upgrade -y && sudo apt full-upgrade -y && sudo apt autoremove -y && sudo apt autoclean
```

Installation of the **Swarm Sensor** is make simple with the script provided by GreyNoise
```bash
GREYNOISE_API_KEY=[redacted] && curl -H "key: ${GREYNOISE_API_KEY}" -L https[:]//api[.]greynoise[.]io/v1/workspaces/[workspace_id]/sensors/bootstrap/script | sudo bash -s -- -k ${GREYNOISE_API_KEY}
```


### Network and Firewall Hardening (_or lack thereof_)
A successful honeypot requires unfiltered access to raw internet noise. Traditional host defense mechanisms will blind the sensor if not properly configured.

1.  **Upstream Cloud Firewalls:** I ensured **no DigitalOcean Cloud Firewalls** were configured or attached to the Droplet. Normally, a cloud firewall dropping uninvited packets is exactly what you want for a secure server. However, a honeypot's entire purpose is to be scanned and attacked. If a DO firewall is active, it destroys the packet at the network edge before it ever reaches the OS.
2.  **Host Firewall (UFW):** I intentionally left the host firewall (`ufw`) **inactive**. Enabling UFW with a default `deny incoming` policy would block scanners from hitting the emulation listeners.
3.  **Management Plane Isolation (SSH):** To prevent my administrative access from being blocked or conflicting with the honeypot's SSH emulation, the host's real OpenSSH daemon is moved off port 22 to a high port.
3a.  To verify the OS was ready and port 22 was clear, I ran:
```bash
ss -tulpn | grep -E ':(22|5[0-9]{4})\b'
```

## Understanding _*the Swarm*_
Within a few minutes my sensor was the target of nearly 3,000 hits

| **Classification** | **Sessions** |
| -------- | -------- |
| malicious | 401 |
| suspicious | 106 |
| benign | 957 |
| unknown | 1,522 |

---

# Why I Deployed a GreyNoise Swarm Sensor (_and why you should too_)

I decided to deploy a GreyNoise Swarm sensor largely out of curiosity and a desire to get my hands on real-world threat data. Reading about new vulnerabilities and emerging exploits is one thing, but having a live sensor allows me to observe the actual exploit payloads hitting the internet in real-time. Because Swarm captures the full interaction, I can download the raw PCAPs of these attacks, dissect the traffic, and analyze exactly how threat actors are probing edge infrastructure.

## Why This Matters for Cybersecurity

By deploying these sensors, we are actively contributing to a massive, distributed early-warning system. The internet is constantly bombarded by automated scanning, botnets, and opportunistic threat actors. Honeypot networks like Swarm absorb this "background radiation," allowing defenders and security researchers to identify emerging zero-day exploitation, track malicious infrastructure, and confidently filter out benign internet noise from highly targeted attacks. Every new sensor deployed adds another vantage point to this global radar.

## Join the Swarm

I highly encourage other cybersecurity professionals, homelabbers, and researchers to deploy their own sensors. It democratizes threat intelligence and is an incredible hands-on learning tool.

GreyNoise has made the barrier to entry practically non-existent. For a one-time fee of just $1 USD, Project Swarm users get enterprise-grade threat hunting capabilities, including:

* Unlimited sensor deployments
* Direct analysis and visualization of your data in the GreyNoise UI
* Deep querying of your data to analyze packets, payloads, binaries, and more
* Bulk PCAP downloads for local dissection
* Full access to the Project Swarm API
* Access to an extensive library of emulation profiles, plus the ability to build your own
* Fleet comparison, allowing you to see what is hitting your specific sensor vs. the global GreyNoise fleet
* Custom tags and detection rules you can write yourself
* Access to the community dataset, and much more.

It is an unbeatable way to support the broader InfoSec community while gaining direct access to actionable, real-world threat intelligence.

---

Thanks for taking the time ツ
