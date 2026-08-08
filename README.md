# Wazuh Security Monitoring Lab

# Overview
This project was created as a hands-on Wazuh security monitoring lab. The objective was to deploy a Wazuh Manager, connect an Ubuntu endpoint as an agent, generate and observe endpoint activity and verify that the Wazuh interface could collect, classify and display the resulting security information.

The project was more of a working lab than a theoretical one. Most of the work was spent troubleshooting the agent-manager connection when the Ubuntu agent showed as disconnected in the Wazuh interface.

# What was achieved
1. Wazuh Manager was running successfully on the Ubuntu VM.
2. Wazuh Agent 4.14.7 was installed on the Ubuntu VM.
3. The Ubuntu Wazuh agent service was enabled and running.
4. The agent was configured to communicate with the Wazuh Manager over TCP port 1514.
5. An IP-address change on the manager was identified as the cause of the later connectivity problem.
6. The agent configuration was updated to the manager's current IP address.
7. The agent subsequently reported a successful connection to the manager.
8. Wazuh was collecting activity from the Ubuntu endpoint and displaying it in the dashboard.
9. The dashboard sections used in the project, including Overview, MITRE-related information, filtering and agent/event views, were functioning.

# Lab Architecture
The lab consisted of two Linux virtual machines. Ubuntu hosted the Wazuh Manager, while the Ubuntu endpoint was also used as the monitored agent.
<div align="center">
<table>
  <thead>
    <tr>
      <th>Component</th>
      <th>Role</th>
      <th>Observed details</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Ubuntu VM</td>
      <td>Wazuh Manager</td>
      <td>Wazuh Manager 4.14.7; TCP 1514 listening</td>
    </tr>
    <tr>
      <td>Ubuntu VM</td>
      <td>Wazuh Agent</td>
      <td>Ubuntu endpoint; Wazuh Agent 4.14.7</td>
    </tr>
    <tr>
      <td>Wazuh Dashboard</td>
      <td>Monitoring interface</td>
      <td>Overview, MITRE, filters, agents and events</td>
    </tr>
  </tbody>
</table>
</div>

<div align="center">
  <table>
    <tr>
      <td align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/1546cbb03ced63493fc78c6a55ca6f3180769b5e/images/Wazuh%20Manager.png" width="200" height="200" />
        <br />
        <b>Wazuh Manager</b>
      </td>
      <td align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/1546cbb03ced63493fc78c6a55ca6f3180769b5e/images/Wazuh%20Agent.png" width="200" height="200" />
        <br />
        <b>Wazuh Agent</b>
      </td>
    </tr>
  </table>
</div>

# Wazuh Manager Setup
The Wazuh Manager was deployed on the Ubuntu VM. Before troubleshooting the agent, the manager service was verified as running.
```text sudo systemctl status wazuh-manager ```.

The service status showed the manager as active and running. The manager was also listening on TCP port 1514 through wazuh-remoted, which is used for agent communication.
```sudo ss -tlnp | grep 1514 ```.

The output confirmed that wazuh-remoted was listening on 0.0.0.0:1514.
<div align="center">
  <table>
    <tr>
      <td align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/blob/c5c0afa9fd2e2dfcb98fc40763e216294cb4bb17/images/Wazuh%20Manager%20service%20status.png" width="200" height="200" />
        <br />
        <b>Wazuh Manager service status</b>
      </td>
      <td align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/blob/c5c0afa9fd2e2dfcb98fc40763e216294cb4bb17/images/Port%201514%20listening.png" width="200" height="200" />
        <br />
        <b>Port 1514 listening</b>
      </td>
    </tr>
  </table>
</div>

# Ubuntu Agent Installation
The Ubuntu machine was selected as the endpoint to be monitored. Because Ubuntu is Debian-based and the VM uses an x86_64/amd64 architecture, the DEB amd64 Wazuh Agent package was used.
```text
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.7-1_amd64.deb
sudo WAZUH_MANAGER='<MANAGER-IP>' WAZUH_AGENT_GROUP='default' WAZUH_AGENT_NAME='ubuntu' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb
```
The package installation completed successfully and the Wazuh agent service was started and enabled.
```text
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```
The service status showed Active: active (running), confirming that the agent processes were running locally.
<div align="center">
  <table>
    <tr>
      <td align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/blob/70de3007fc43bee83799d1e39dff55e1371eff21/images/Wazuh%20Agent%20running.png" width="200" height="200" />
        <br />
        <b>Wazuh Agent running</b>
      </td>
    </tr>
  </table>
</div>

# Problem Faced: Agent Showing as Disconnected
The main troubleshooting issue in the project was that the Wazuh Agent service was running on the Ubuntu endpoint, but the Wazuh Manager/dashboard showed the agent as disconnected.

This was an important distinction: an active agent service did not necessarily mean that the agent had an active connection with the Wazuh Manage.

<b>Initial Evidence</b>
The Ubuntu agent log showed that it was repeatedly attempting to connect to the previous manager address:
```text
Trying to connect to server ([192.168.0.9]:1514/tcp).
ERROR: (1216): Unable to connect to '[192.168.0.9]:1514/tcp': 'Transport endpoint is not connected'
```
The Wazuh Manager itself was running normally. Checking its network configuration showed that its current IPv4 address had changed to:
10.116.200.165.

The manager was also confirmed to be listening on TCP port 1514:
```text
LISTEN 0 128 0.0.0.0:1514 0.0.0.0:* users:(("wazuh-remoted",...))
```

<b>Root Cause</b>
The Ubuntu agent was configured with the manager's previous IP address, 192.168.0.9. The manager's IP had changed to 10.116.200.165, but the agent configuration still pointed to the old address.

<b>Resolution</b>
The manager address in the Ubuntu agent configuration was updated to the manager's current IP, and the Wazuh Agent service was restarted.
```text
sudo nano /var/ossec/etc/ossec.conf
```
The address value was changed to:
```text
<address>10.116.200.165</address>
```
The agent was then restarted:
```text
sudo systemctl restart wazuh-agent
```

<b>Verification</b>
After the configuration change, the agent log showed a successful connection:
```text
wazuh-agentd: INFO: (4102): Connected to the server ([10.116.200.165]:1514/tcp).
```
This confirmed that the agent–manager connectivity issue had been resolved and the Ubuntu agent was successfully communicating with the Wazuh Manager.

# Wazuh Dashboard Validation
After restoring the agent–manager connection, the Ubuntu endpoint began reporting activity to the Wazuh Manager. The dashboard was used to verify that the collected data was being received and displayed correctly.

<b>Sections Validated</b>
- <b>Overview:</b> Reviewed the overall security and monitoring status.
- <b>Agents:</b> Verified the Ubuntu endpoint and its connection status.
- <b>MITRE ATT&CK:</b> Reviewed activity mapped to relevant techniques.
- <b>Alerts/Events:</b> Inspected security events generated from the Ubuntu endpoint.
- <b>Filtering:</b> Used filters to narrow down and examine specific events.
- <b>Endpoint Information:</b> Confirmed that the Ubuntu machine was being monitored and reporting data.
The validation focused only on the Wazuh features used during this project. The sections above were tested and confirmed to be functioning with the connected Ubuntu endpoint.

<div align="center">
  <table width="100%">
    <!-- Row 1 -->
    <tr>
      <td width="50%" align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/21a9091839dad3fd637f66777074b3ce6af42e07/images/Wazuh-Vulne..png" width="100%" />
        <br />
        <b>Wazuh Vulnerability</b>
      </td>
      <td width="50%" align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/21a9091839dad3fd637f66777074b3ce6af42e07/images/Wazuh-ThreatHunting.png" width="100%" />
        <br />
        <b>Wazuh Threat Hunting</b>
      </td>
    </tr>
    <!-- Row 2 -->
    <tr>
      <td width="50%" align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/21a9091839dad3fd637f66777074b3ce6af42e07/images/Wazuh-MalwareDect-Events.png" width="100%" />
        <br />
        <b>Wazuh Malware Detection Events</b>
      </td>
      <td width="50%" align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/21a9091839dad3fd637f66777074b3ce6af42e07/images/Wazuh-MITRE.png" width="100%" />
        <br />
        <b>Wazuh MITRE ATT&CK</b>
      </td>
    </tr>
    <!-- Row 3 -->
    <tr>
      <td width="50%" align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/21a9091839dad3fd637f66777074b3ce6af42e07/images/Wazuh-ITHygiene.png" width="100%" />
        <br />
        <b>Wazuh IT Hygiene</b>
      </td>
      <td width="50%" align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/21a9091839dad3fd637f66777074b3ce6af42e07/images/Wazuh-Endpoint.png" width="100%" />
        <br />
        <b>Wazuh Endpoint Security</b>
      </td>
    </tr>
    <!-- Row 4 -->
    <tr>
      <td width="50%" align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/21a9091839dad3fd637f66777074b3ce6af42e07/images/Wazuh-Discover.png" width="100%" />
        <br />
        <b>Wazuh Discover</b>
      </td>
      <td width="50%" align="center">
        <img src="https://github.com/AyushDongarwar/Wazuh-Security-Monitoring-Lab/raw/21a9091839dad3fd637f66777074b3ce6af42e07/images/Wazuh-MalwareDect-Dashboard.png" width="100%" />
        <br />
        <b>Wazuh Malware Detection Dashboard</b>
      </td>
    </tr>
  </table>
</div>

# Endpoint Activity and Detection
The Ubuntu VM was used as the monitored endpoint. After the Wazuh Agent successfully connected to the Manager, activity performed on the Ubuntu machine was collected and displayed in Wazuh.

This confirmed the complete flow from endpoint activity to collection by the Wazuh Agent and visibility in the Wazuh Manager/dashboard.

In the final project state, the Ubuntu machine was not only connected as an agent; Wazuh was actively receiving and displaying activity from the endpoint.

# Troubleshooting Timeline
<table>
  <thead>
    <tr>
      <th>Step</th>
      <th>Action</th>
      <th>Result</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td align="center"><b>1</b></td>
      <td>Installed Wazuh Agent</td>
      <td>DEB amd64 package installed successfully on Ubuntu.</td>
    </tr>
    <tr>
      <td align="center"><b>2</b></td>
      <td>Started the agent</td>
      <td><code>wazuh-agent.service</code> showed active (running).</td>
    </tr>
    <tr>
      <td align="center"><b>3</b></td>
      <td>Dashboard showed disconnected</td>
      <td>The service was running, but communication was not stable.</td>
    </tr>
    <tr>
      <td align="center"><b>4</b></td>
      <td>Checked agent logs</td>
      <td>Logs showed connection attempts to <code>192.168.0.9:1514</code> and connection failures.</td>
    </tr>
    <tr>
      <td align="center"><b>5</b></td>
      <td>Checked manager</td>
      <td>Wazuh Manager was active and <code>wazuh-remoted</code> was listening on <code>0.0.0.0:1514</code>.</td>
    </tr>
    <tr>
      <td align="center"><b>6</b></td>
      <td>Checked manager IP</td>
      <td>The manager's current IP was <code>10.116.200.165</code>.</td>
    </tr>
    <tr>
      <td align="center"><b>7</b></td>
      <td>Updated agent configuration</td>
      <td>Ubuntu agent was changed to use <code>10.116.200.165</code>.</td>
    </tr>
    <tr>
      <td align="center"><b>8</b></td>
      <td>Restarted agent</td>
      <td>Agent reconnected successfully.</td>
    </tr>
    <tr>
      <td align="center"><b>9</b></td>
      <td>Validated dashboard</td>
      <td>Ubuntu activity was visible across the Wazuh monitoring interface.</td>
    </tr>
  </tbody>
</table>

# What I Learned
- A running systemd service does not guarantee successful network communication.
- Checking the Wazuh agent logs was more useful than relying only on the dashboard status.
- The manager's current IP must match the address configured on the agent.
- Checking port 1514 with ss helped distinguish a Wazuh service issue from a network configuration issue.
- Virtualized environments can have changing IP addresses that affect agent–manager connectivity.
- Testing the connection after each configuration change made troubleshooting easier to follow.
- Final validation should include actual endpoint activity, not just an active agent service.

# Practical Notes
This project was built as a personal lab environment rather than a production SOC deployment. The setup used virtual machines with IP-based communication. The Wazuh Manager's dynamically assigned IP changed during the lab, which caused the agent–manager connectivity issue.

For a production deployment, additional considerations such as stable addressing, DNS, firewall rules, access control, key management, backups and network segmentation would need to be configured appropriately.

# Short Project Description
Built a Wazuh security monitoring lab using Ubuntu as the Wazuh Manager and Ubuntu as the monitored endpoint. Configured and troubleshot agent–manager communication, diagnosed an IP-address mismatch that caused the agent to appear disconnected, restored connectivity over TCP 1514 and validated endpoint activity through Wazuh's dashboard, alerts, filtering and MITRE ATT&CK views.

# Final Outcome
The final environment was operational: the Ubuntu Wazuh Agent was running, the Wazuh Manager was receiving agent data, and endpoint activity was visible in the Wazuh interface. The main troubleshooting issue was the manager IP changing while the agent retained the previous address. Identifying the mismatch through logs and network checks, updating the configuration and verifying the connection completed the core lab.
