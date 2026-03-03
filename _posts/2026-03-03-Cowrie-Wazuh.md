---
layout: post
title:  "Cowrie Honeypot & Wazuh Threat Monitoring Lab"
date:   2026-03-04 02:17:55 +0800
categories: Demo
tags: Cybersecurity
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
## Introduction:
This article includes the overall process of building up a honeypot using Cowrie with a SIEM/XDE tool(Wazuh) to detect and analyze attacks. Here's the following configuration settings:

```bash
VPS1(Wazuh Server)
- vCPU: 4
- RAM: 8GiB
- Storage: 120GiB
- System: Ubuntu 24.04

VPS2(Cowrie + Wazuh agent)
- vCPU: 2
- RAM: 2GiB
- Storage: 75GiB
- System: Ubuntu 24.04
```

<br><br>

## Wazuh Server(VPS1)
- 1. Install Wazuh Server
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

- 2. Get Default User & Password
```bash
User: admin
Password: 9xxxxxxxxxxxxxxxxxxxxxxxx8
```

- 3. You can find password using this command
```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

- 4. Next, try if you can access Wazuh dashboard by directly accessing `https://WAZUH_SERVER_PUBLIC_IP`. If you can't, let's debug and find what the problem is.
```bash
# Ensure these three services is active
sudo systemctl status wazuh-dashboard
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-manager

# Ensure Wazuh binds 443/tcp successfully
sudo ss -lntp | grep ':443'

# Check wazuh on localhost. It should return something like curl: (60) SSL certificate problem: unable to get local issuer certificate (If not, lets continue)
curl -I https://127.0.0.1

# If not, check firewalls and iptables
ufw status verbose
iptables -S

# It should probably be ufw which didn't open 443/tcp by default
ufw allow 443/tcp 
ufw reload
ufw status verbose

# # To                         Action      From
# --                         ------      ----
# 22/tcp                     ALLOW IN    Anywhere                  
# 443/tcp                    ALLOW IN    Anywhere                  
# 22/tcp (v6)                ALLOW IN    Anywhere (v6)             
# 443/tcp (v6)               ALLOW IN    Anywhere (v6)

# ufw status numbered
# ufe delete [No] to delete rules

# Check on localhost again. This time should return "SSL certificate problem"
curl -I https://127.0.0.1
```

- 5. Finally, access Wazuh dashboard via `https://WAZUH_SERVER_PUBLIC_IP`
![Wazuh Dashboard](/assets/img/post-img/wazuh-1.png)

<br>

## Allow connection from VPS2(Honeypot) to VPS1(Wazuh Server)
- In VPS1, allow access from agent
```bash
sudo ufw allow from <HONEYPOT_PUBLIC_IP> to any port 1514 proto tcp
sudo ufw allow from <HONEYPOT_PUBLIC_IP> to any port 1515 proto tcp
sudo ufw status verbose
```

## VPS2(Honeypot) SSH Setting
- Change actual SSH port to 22222
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo sed -i 's/^#Port 22/Port 22222/' /etc/ssh/sshd_config

sudo sshd -t

sudo ufw allow 22222/tcp

sudo systemctl restart ssh
```

- Don't close current terminal. Open another new terminal and try log in via `ssh -p 22222`. Check if you're able to login via port 22222. Otherwise you'll lock yourself

<br>

## Install Cowrie on VPS2(Honeypot)
- 1. Install dependencies
```bash
sudo apt-get update
sudo apt-get install -y git python3-pip python3-venv libssl-dev libffi-dev build-essential libpython3-dev python3-minimal authbind
```

- 2. Create a user to run Cowrie. Because we don't want use `root`.
```bash
sudo adduser --disabled-password cowrie
```

- 3. Git clone Cowrie project, use python venv and install pip libs
```bash
sudo -u cowrie -H bash -lc '
cd ~
git clone https://github.com/cowrie/cowrie
cd ~/cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
python -m pip install --upgrade pip
python -m pip install -e .
'
```

- 4. Change the port which Cowrie uses
```bash
sudo touch /etc/authbind/byport/22
sudo chown cowrie:cowrie /etc/authbind/byport/22
sudo chmod 770 /etc/authbind/byport/22

sudo -u cowrie -H bash -lc 'cat > /home/cowrie/cowrie/etc/cowrie.cfg << "EOF"
[ssh]
listen_endpoints = tcp:22:interface=0.0.0.0
EOF'
```

- 5. Allow inbound 22/tcp in VPS2(Honeypot)
```bash
sudo ufw allow 22/tcp
sudo ufw status verbose
```

- 6. Run Cowrie
```bash
sudo -u cowrie -H bash -lc '
cd /home/cowrie/cowrie
source cowrie-env/bin/activate
AUTHBIND_ENABLED=yes cowrie start
'
```

- 7. Check logs on Cowrie
```bash
sudo -u cowrie -H bash -lc 'cd /home/cowrie/cowrie && tail -n 50 var/log/cowrie/cowrie.log'
sudo ss -lntp | grep ':22'
```

<br>

## Install Wazuh Agent
- 1. Add Wazuh repo
```bash
sudo apt-get install -y gnupg apt-transport-https
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
```

- 2. Install wazuh-agent. Note that `<WAZUH_SERVER_IP>` should be replaced with your VPS1 public IP.
```bash
sudo WAZUH_MANAGER="<WAZUH_SERVER_IP>" apt-get install -y wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

- 3. Check the status and logs of wazuh-agent
```bash
sudo systemctl status wazuh-agent 
sudo journalctl -u wazuh-agent -n 50 
```

<br>

## Connect Cowrie with Wazuh-Agent
- 1. Add cowrie JSON output file to wazuh-agent
```bash
sudo vim /var/ossec/etc/ossec.conf
# Add the following to /var/ossec/etc/ossec.conf
<localfile>
  <location>/home/cowrie/cowrie/var/log/cowrie/cowrie.json*</location>
  <log_format>json</log_format>
  <label key="cowrie_source">cowrie</label>
</localfile>
```

- 2. Restart the service
```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

## Examination
This process is the most difficult and annoying part because you may encounter many issues depending on individual settings. I can provide several issues that I encountered during this examination process.

1. Check Wazuh-Server Dashboard on VPS1
You should go check out the dashboard on VPS1 via `https://VPS1_PUBLIC_IP`, login in, and check the top left corner which describes the connected agents. If there's no agent there, it means something goes wrong.
![Wazuh Manager Dashboard](/assets/img/post-img/wazuh-3.png)

2. Check on VPS2
```bash
# Check the logs first
journalctl -u wazuh-agent -f

# Check the state of wazuh-agent, if it's "pending" means it's trying to connect to the wazuh-server/manager. You may see some error logs in the next step
grep ^status /var/ossec/var/run/wazuh-agentd.state

# Check the detail error logs of wazuh-agent. You may see trying to connect prompts and the root cause. Mine is "Invalid agent name: xxx" which I have two same names as my agent and my manager.
tail -n 100 /var/ossec/logs/ossec.log
```

3. If you have the same problem as mine, do the following on VPS2(honeypot):
- Modify the `/var/ossec/etc/ossec.conf` file:
```bash
vim /var/ossec/etc/ossec.conf
```

- Add `<enrollment></enrollment>` part which changes the agent name of VPS2
```xml
<client>
  <server>
    <address>149.28.118.25</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>

  <enrollment>
    <enabled>yes</enabled>
    <manager_address>149.28.118.25</manager_address>
    <port>1515</port>
    <agent_name>cowrie-vps-01</agent_name>
  </enrollment>
</client>
```

- 4. Restart `wazuh-agent` service again
```bash
sudo systemctl restart wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
```

- 5. Check on Wazuh Server Dashboard or via CLI:
```bash
# You should see new agent's name appeared in the log
sudo /var/ossec/bin/agent_control -l
```

<br>

## Result
Right now, we have a Cowrie honeypot running with a Wazuh agent, sending logs to the Wazuh Server(Manager) for analysis and monitoring. This is a great project to enhance the ability for further analyzing cyber attacks or getting some practical experience to build up a comprehensive home lab. I'll let them running for like a half month to collect the data and will update after that. See ya!
![Wazuh Manager Dashboard](/assets/img/post-img/wazuh-2.png)


<br><br>

## Materials
1. [Wazuh Introduction (Youtube)](https://www.youtube.com/watch?v=Hq58_yGJwHk)
2. [Wazuh Setup - Detail Process (Youtube)](https://www.youtube.com/watch?v=bltbJ2TUQWU)
3. [Wazuh Official Documentation - QuickStart](https://documentation.wazuh.com/current/quickstart.html)
4. [Wazuh Github](https://github.com/wazuh/wazuh)

</div>
</body>
</html>
