1. Lab Infrastructure
This lab environment was designed to monitor and secure a hybrid infrastructure consisting of both Windows and Linux endpoints.

Wazuh Manager: Hosted on an Ubuntu Server, acting as the central brain for log collection, correlation, and alerting.
Linux Endpoint: An Ubuntu Server running an Nginx web server, monitored by a Wazuh Agent for authentication and system logs.
Windows Endpoint: A Windows 10 workstation monitored by a Wazuh Agent and Sysmon, providing granular visibility into process creation and network connections.


2. Attack Simulation: Multi-Protocol Brute Force
To test the detection capabilities of the SIEM, I performed a dictionary-based brute force attack against both protocols: RDP (Windows) and SSH (Linux) using Hydra on a Ubuntu Linux machine.

Attack Proof (Ubuntu Linux):
Targeting: Attempting to crack the Administrator account on Windows and the soc account on Linux.
Outcome: The attack was successful, identifying the correct password (Qweqwe123) for both systems.

Linux attack command -> hydra -vV -l soc -P passwords_linux.txt 192.168.101.11 ssh
<img width="1291" height="501" alt="Hydra Atak - Linux" src="https://github.com/user-attachments/assets/731ea80c-6ad6-4c19-a129-f4253464e1a4" />
