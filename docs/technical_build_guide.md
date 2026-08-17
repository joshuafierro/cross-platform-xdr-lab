# Technical Build Guide



## Setting up Virtual Machines (VMs)

The genesis of this project began with scaffolding three virtual machines which would each represent part of a enterprise environment. It was decided that two VMs would be Linux based and one would be a Windows 11 machine. Considering the resources of the host machine it was important to choose Linux distributions (Distros) which utilized few resources but were highly compatible with the EDR/SIEM of choice and other common security tools. Thus **Ubuntu** was chosen as an endpoint device  (Endpoint A) housing the vulnerable web application and **CentOS** would be the central management server for the EDR/SIEM. **Windows 11** was also chosen to be a monitored endpoint device (Endpoint B). It should be noted that each Endpoint was configured to maintain a static IP address to ensure the reliability of network communications between agents and the central management service.

### Virtual Machine Specifications 

| Operating System | RAM  | Storage |
| ---------------- | ---- | ------- |
| Ubuntu           | 4GB  | 20GB    |
| Windows 11       | 4GB  | 64GB    |
| CentOS           | 10GB | 75GB    |



## EDR/XDR/SIEM Scaffolding

Since the objective of this experiment includes the creation of a EDR/XDR/SIEM Detection environment it was imperative to choose an appropriate EDR/XDR/SIEM solution. In the previous section the desire to have a Windows 11 machine as a endpoint and CentOs as the central management server was expressed therefore Microsoft Sentinel was ruled out as possible solution. That left Splunk and Wazuh as possible tools. At the time of writing Splunk is a Free solution up to a point. Despite having meaningful experience with Splunk this would not be acceptable for what is trying to be achieved in this experiment. This left [Wazuh](https://wazuh.com/), an open source solution that "unifies XDR and SIEM protection by collecting deep endpoint telemetry, network logs, and cloud data into a single rule engine for cross-domain correlation and automated response". It was not only promising but would deliver the functionality needed at no cost. 

### Deploying Agents



## Deploying Vulnerable Application on Endpoint A

In this phase the intentionally vulnerable web application I had previously developed was deployed to the Ubuntu endpoint. Here it will be tested against for the purpose of simulating a live enterprise web application. 

The script for executing the application was slightly altered to force it to run on 0.0.0.0 so it could be accessed by the greater local network. 



## Setting Up Rules and Decoder for Flask logs

The vulnerable web application running is great however its logs need to be accessible in order to fulfill the purpose of this experiment. The next step was to alter the vulnerable web application to write logs to a file on the server and make sure that the user space for the wazuh manager had permission to read and access the log file and directory it is saved within.

Raw flask logs are not natively supported by Wazuh so we needed to create a decoder which will hold a regex that identified the Flask HTTP logs. Additionally rules were created/defined to trigger alerts for event logs. Specific rules were created to catch generic 404 and 500 errors as well under their own respective ids. 

Once created and saved the next step was to test the decoder with the native **wazuh-logtest** found in `/var/ossec/bin/wazuh-logtest`. It requires a sample to test the decoder against. This was obtained from the running app with the following command: 

`tail -n 1 /var/log/flask/access.log`

Once the test passed all that was left was to restart the manager: 

`systemctl restart wazuh-manager`



## Analyzing Logs and Running Attacks Against Endpoint

Once logs were being ingested by the agent from the Flask application I proceeded to run manual attacks against the web application. The first of which was a Reflective Cross Site Scripting attack (XSS). 

After waiting a minute for the logs to propagate, I navigated to the **Wazuh Threat Intelligence** dashboard and searched the **Events** for the attack:

```
full_log="<img"
```

This simple search returned the log containing the attack: 

```
<img src="" onerror=alert(1) />
```

Tests for benign 404 errors was conducted to see if these rules were being picked up as well. Searching for a 404 error returned the expected log: 

```
192.168.68.130 - - [14/Aug/2026 22:17:35] "GET /testnotfound HTTP/1.1" 404 -
```

