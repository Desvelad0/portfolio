Wazuh is a really cool free, open source tool that unifies XDR and SIEM capabilities into one neat package. It has features such as vulnerability detection, file integrity monitoring and log data analysis, and a lot more. I have previously used Wazuh in a professional capacity, and have been meaning to install it into my own environment, which is what this post is about.

**Objective**

- Vulnerability management for clients, starting with PCs in the network, and later on adding in servers
  - Automatic scanning network-wide in faster intervals than what the default is

- Making dashboards for monitoring vulnerabilities, perhaps setting up an automatic messaging system for when vulnerabilities are found (Telegram/Discord bot)
- File integrity monitoring for PCs
- Making use of log collecting and perhaps exporting them to another platform to visualize said logs
- Look into the IPS/IDS capabilities of the tool and see what could be implemented
- Any other cool thing I come across

Wazuh has some really good documentation on their website which is what I'll be using to guide me through the installation of this tool (https://documentation.wazuh.com/current/index.html). There's a few ways to install Wazuh:

- Ready-to-use machines in the form of OVAs; pre-built VM images that can be imported to any OVA compatible virtualization systems
- Amazon Machine Image (AMI), which can be launched directly on an AWS cloud instance
- Containers (Docker or Kubernetes)
- Offline
- From source 

Since I have my own homelab, I'll be installing Wazuh there.