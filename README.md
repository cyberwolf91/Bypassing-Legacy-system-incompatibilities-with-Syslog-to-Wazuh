# Bypassing-Legacy-system-incompatibilities-with-Syslog-to-Wazuh
Metasploitable uses a very old version of Ubuntu. To keep all alert data central to the Wazuh Dashboard we are going to use Syslog.

Built a home lab to demonstrate how you can still utilize Wazuh for legacy systems. Three VMs on a VirtualBox NAT Network called KaliLab - internet access, and VMs can all communicate with each other.

Machines:
  Defender        Ubuntu 26       Wazuh Manager        10.0.2.3
  Metasploitable2 Ubuntu 8.04     Vulnerable target     10.0.2.4
  Ubuntu box      Ubuntu 26       Attacker (this demo)  10.0.2.7
  Kali            Kali Linux      Main attacker (soon in future labs)  10.0.2.x


Why syslog instead of a Wazuh agent on Metasploitable2?
--------------------------------------------------------
Metasploitable2 runs Ubuntu 8.04. Too old for a modern Wazuh agent to install.
Syslog forwarding is the workaround - same thing you'd do for legacy gear in a real environment.


Step 1 - Add syslog listener to Wazuh
--------------------------------------
By default Wazuh only listens for agents on port 1514.
We need to add a second listener for raw syslog on port 514.

Edit the config:
  sudo nano /var/ossec/etc/ossec.conf

Add this block (leave the existing secure/1514 block alone):

  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>10.0.2.0/24</allowed-ips>
  </remote>

Also enable full archiving inside the <global> block:

  <logall>yes</logall>
  <logall_json>yes</logall_json>

Restart and verify:
  sudo systemctl restart wazuh-manager
  sudo ss -ulnp | grep 514

You should see wazuh-remoted bound to 0.0.0.0:514.


Step 2 - Forward syslog from Metasploitable2
---------------------------------------------
Metasploitable2 uses sysklogd, not rsyslog. Different config, stricter syntax.

  sudo nano /etc/syslog.conf

Add this line at the bottom:
  *.*	@10.0.2.3

NOTE: The gap between *.* and @10.0.2.3 must be a TAB not spaces.
The parser silently ignores the line if you use spaces. Verify with:

  cat -A /etc/syslog.conf | tail -3

Look for ^I between *.* and the IP. That is your tab character.
Old syslog.conf also does not support a port number or TCP. UDP/514 only.

Restart sysklogd:
  sudo /etc/init.d/sysklogd restart

Confirm it is running:
  ps aux | grep syslogd

You will see two lines. The real one has /sbin/syslogd at the end.
The other is just the grep command itself - ignore it.


Step 3 - Test the pipeline
---------------------------
On Metasploitable2:
  logger "TEST FROM METASPLOITABLE"

On Defender:
  sudo grep -i "TEST FROM" /var/ossec/logs/archives/archives.log

If the message shows up, the pipeline is working.


Step 4 - Generate a real alert (SSH brute force)
-------------------------------------------------
Metasploitable2's SSH only supports old key types that modern clients dropped.
A plain ssh command fails before even reaching a password prompt.
Use these flags to force compatibility:

  ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa msfadmin@10.0.2.4

Enter the wrong password 3 to 5 times.
Each failure gets logged by Metasploitable2, forwarded via syslog, and picked up by Wazuh.

Check that alerts actually fired:
  sudo grep -i "msfadmin" /var/ossec/logs/alerts/alerts.log | tail -10

You should see PAM authentication failure entries with rhost=10.0.2.7 user=msfadmin.
That means Wazuh's rule engine matched it - not just a raw log sitting in archives.


Step 5 - Find it in the dashboard
-----------------------------------
Open a browser on Defender and go to:
  https://localhost

Or from another VM:
  https://10.0.2.3

Go to Threat Hunting > Events.
Filter by data.srcip: 10.0.2.7 or search msfadmin in the search bar.

One thing worth knowing - the alert will show agent.name: defender-VirtualBox.
That does not mean the event happened on Defender.
It just means Defender is the Wazuh node that received and processed the syslog.
The real origin is Metasploitable2. You can confirm by reading the full_log field
which shows the raw sshd line with msfadmin and the attacker IP inside it.


Log files to know
------------------
/var/ossec/logs/archives/archives.log  - everything received, rule match or not
/var/ossec/logs/alerts/alerts.log      - only what matched a rule
/var/ossec/logs/ossec.log              - Wazuh internals, go here when something breaks
