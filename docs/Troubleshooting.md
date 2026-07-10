# 🛡️ Project Write-Up: Building a Cross-Platform SIEM with Wazuh

## 🎯 The Objective
As cybersecurity threats become more sophisticated, having visibility into endpoint activity is no longer optional—it's mandatory. My goal for this project was to step into the shoes of a Security Operations Center (SOC) analyst and a Security Engineer. I wanted to build a centralized **Security Information and Event Management (SIEM)** solution using **Wazuh** to monitor a multi-OS environment. 

Specifically, I set out to implement **File Integrity Monitoring (FIM)** on a Windows machine to detect unauthorized file creations, deletions, and modifications in real-time, sending those alerts back to a central Ubuntu-based manager.

---

## 🧗‍♂️ Phase 1: The Foundation & The Resource Trap
**The Plan:** Deploy the Wazuh Manager, Indexer, and Dashboard on an Ubuntu server.
**The Execution:** I opted to use the Wazuh installation assistant to set up the core stack.

### 🛑 Where I Got Stuck
About halfway through the installation, the Wazuh Indexer (which is heavily based on OpenSearch) began failing to start. The installation script hung, and checking the system services revealed that the indexer process was continuously being killed by the OS. 

### 💡 The Solution
After digging through `/var/log/syslog` and using `dmesg`, I discovered the culprit: **OOM (Out of Memory) Killer**. The OpenSearch database is incredibly memory-hungry, and my initial Ubuntu virtual machine didn't have enough RAM. 
To face this difficulty without abandoning my current environment, I did two things:
1. I increased the base RAM allocation for the Ubuntu machine.
2. I created a 4GB Linux Swap file to give the system breathing room during heavy indexing spikes. 
*Result: The Wazuh Manager and Dashboard booted up flawlessly.*

---

## 🧗‍♂️ Phase 2: Bridging the Gap (Windows to Linux)
**The Plan:** Install the Wazuh Agent on a Windows endpoint and connect it to the Ubuntu Manager.

### 🛑 Where I Got Stuck
I installed the MSI package on Windows, entered the Ubuntu server's IP address, and started the service. I excitedly opened the Wazuh Dashboard, expecting to see my Windows machine listed under "Active Agents." 
Instead, it read: **Disconnected (Never Connected)**. 
I checked the Windows agent logs (`ossec.log`), and it was flooded with `WARN: Unable to connect to manager` errors.

### 💡 The Solution
This was a classic networking and firewall issue. I realized that while the two machines could ping each other, specific ports required for agent-manager communication were blocked.
1. I logged into the Ubuntu server and checked UFW (Uncomplicated Firewall). I had to explicitly allow inbound traffic on TCP/UDP port **1514** (Agent communication) and **1515** (Agent enrollment).
2. I also created an outbound rule in the Windows Defender Advanced Firewall to ensure the agent wasn't being blocked locally.
*Result: A quick restart of the Wazuh service on Windows, and the dashboard lit up green. The agent was successfully enrolled and active!*

---

## 🧗‍♂️ Phase 3: Taming File Integrity Monitoring (FIM)
**The Plan:** Configure the Windows agent to monitor a highly sensitive directory (e.g., `C:\Users\Admin\ImportantFiles`) for any file additions, deletions, or modifications.

### 🛑 Where I Got Stuck
I edited the `ossec.conf` file on the Windows machine, added the directory to the `<syscheck>` block, and saved it. I went into the folder, created a test file, and jumped to the dashboard. **Nothing happened.** No alerts. 

I waited 10 minutes. Still nothing. 

### 💡 The Solution
I had to dive deep into the Wazuh documentation regarding the `syscheck` daemon. I realized two critical things:
1. **Scans are scheduled by default.** Wazuh only checks file integrity every 12 hours by default to save CPU resources. To fix this, I had to append the `realtime="yes"` attribute to my directory tag in the XML config.
2. **Noise.** Once I enabled real-time scanning, the dashboard started blowing up with alerts because Windows was constantly updating hidden `.tmp` and `.log` files in the background. It was alert fatigue instantly.
3. I solved this by implementing the `<ignore>` tag and using `sregex` (regular expressions) to tell Wazuh to ignore files ending in `.log` and `.tmp`. Finally, I added `report_changes="yes"` so the dashboard would show me the exact text diffs when a file was modified.

---

## 🏆 Final Validation & Takeaways
With the hurdles cleared, the system worked beautifully. I could drop a mock "malware" executable into the folder, and within seconds, a Level 7 alert would trigger on the Ubuntu SIEM dashboard. If I modified a configuration file, Wazuh showed me exactly which lines were deleted and added.

### What I Learned:
* **Log Analysis is king:** When things break, the answer is almost always in the logs (`dmesg`, `syslog`, `ossec.log`). 
* **Security vs. Usability:** FIM is incredibly powerful, but if not tuned correctly with ignore rules, it will overwhelm an analyst with false positives. Tuning is just as important as deployment.
* **Cross-OS Networking:** Getting Linux and Windows to securely talk to each other requires careful attention to firewall rules and port bindings.

This project transitioned me from just reading about SIEMs to actually understanding the underlying mechanics, configurations, and troubleshooting required to maintain a robust security posture.
