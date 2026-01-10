1. Lab Infrastructure (Environment Setup)
This lab environment was designed to monitor and secure a hybrid infrastructure consisting of both Windows and Linux endpoints.

Wazuh Manager: Hosted on an Ubuntu Server, acting as the central brain for log collection, correlation, and alerting.

Linux Endpoint: An Ubuntu Server running an Nginx web server, monitored by a Wazuh Agent for authentication and system logs.

Windows Endpoint: A Windows 10 workstation monitored by a Wazuh Agent and Sysmon, providing granular visibility into process creation and network connections.
