- [Server Tokens & Versions](#markdown-header-server-tokens-versions)
- [Ports, Firewalls & Anti-Virus](#markdown-header-ports-firewalls-anti-virus)

_Last Update: **March 2022**

In this guide we are going to go over some common steps in prep for a pen test or to help you clean up an already existing report,

There is no real order in which to tackle each thing and some of it may have been done already, so this is more of a reference guide to help clean up an existing report.

### Starting point

I’m going to assume that you can connect to the server in question via SSH, and that you understand references to common theme files inside a WordPress build.

# Server Tokens & Versions

Hackers can do an awful lot with a tiny piece of info such as what version of server-side software you are using. Fresh installs of a hosting environments almost always default to a development mode, meaning many critical details are fed to the front end, details that give attackers a head start.

### WordPress

copy the following into the theme’s functions.php

    remove_action('wp_head', 'wp_generator');

If the site uses WooCommerce, also add the following to the functions.php file

    remove_action('wp_head', 'woo_version');

**DON’T BE LAZY…** please inspect your code and look for any meta data from plugins giving away their usage, and more importantly their versions. Don’t just leave it and hope the pen testers won’t flag it, Go and google the functions.php code required to remove generator tags, and when possible, the version numbers from the URL’s of the JS and CSS libraries.

### Apache on our AWS Ubuntu 20 & Hetzner Ubuntu 20 + Plesk servers

Connect to the server via SSH

    sudo su
### 

    cd /etc/apache2
### 

    nano apache2.conf

Find and set the “ServerTokens” variable. If the config file doesn’t have this value set at all, add the line at the end. It defaults to dev mode, so this must be set.

    ServerTokens Prod

Staying in the same file, find or add the “ServerSignature” variable to the end.

    ServerSignature Off

Save the file

`CRTL + O or CMD + O`

`Hit enter to confirm`

Exit the nano editor

`CRTL + X or CMD + X`

We will now stop the server feeding back it’s UNIX time stamp, I don’t understand the risks of the front end knowing what time the server thinks it is, but it’s flagged as a moderate risk and easy to take out.

    cd /etc/

    nano sysctl.conf

Find or add the “net.ipv4.tcp_timestamps” variable and set it as:

    net.ipv4.tcp_timestamps=0

Save the file

`CRTL + O or CMD + O`

`Hit enter to confirm`

Exit the nano editor

`CRTL + X or CMD + X`

    reboot

# Ports, Firewalls & Anti-Virus

Ports are access points for attack, and some present added vulnerability to DDoS, guessing attacks and sometimes insecure connections that would be vulnerable on infected systems and insecure public networks.

On the server, we will only be adding basic rules to block all ports other than the 3 we need. For obvious reasons it’s best to put any IP specific rules inside the hosting provider’s firewall.

### AWS Servers

On AWS we will configure the IP tables via SSH, so log into the server via SSH

    sudo su

First, we need to install a tool to make our changes here persistent upon reboot

    apt install iptables-persistent -y

'Save current IPv4 rules? Yes'
'Save current IPv6 rules? Yes'

**From this point forward, DO NOT CLOSE YOUR SSH CONNECTION! If you disconnect during the following steps, there will be no way to connect to the server and it will need a full operating system reinstall.**

We first want to reject all connections by default, that way we only open what the server needs. To do that punch in the following:

    iptables -A INPUT -i lo -j ACCEPT
###

    iptables -A OUTPUT -o lo -j ACCEPT
###

We want the system to maintain established connection information in memory:

    iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
    iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED -j ACCEPT
    iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

We need SFTP and SSH access. For this one we need to set rules for the sport and dport

    iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
    iptables -A INPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT
    iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT
    iptables -A OUTPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT

Next comes our HTTP/HTTPS ports

    iptables -A INPUT -p tcp -m multiport --dports 80,443,8080 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
    iptables -A OUTPUT -p tcp -m multiport --dports 80,443,8080 -m conntrack --ctstate ESTABLISHED -j ACCEPT

Those are enough to save knowing we’ll be able to reconnect via SSH and SFTP, and the ports to accept web requests are open. Every other port will be closed unless we explicitly add rules to open them.

Let’s save that so the server boots with those rules enabled.

    netfilter-persistent save

**THIS NEXT BIT IS VERY IMPORTANT! DO NOT CLOSE YOUR SSH WINDOW!!!**

Open a new Terminal or Command Prompt window and connect to the server via SSH, if you connect without any problem, we haven’t bricked the server.
For some reason I’ve noticed that the reboot after these rule updates is an extra-long reboot, and it might scare you into thinking you’ve bricked it. So, if the server doesn’t let you log in after about nighty seconds, give it 5 minutes then try again.

    reboot

Reconnect via SSH and make sure the IP Table rules loaded with the reboot, and remember to go back to super user.
    sudo su
###
    iptables -L -v
###

And to add to the firewall, we want fail-to-ban so malicious activity is logged and blocked.

    apt install fail2ban -y

With that installed we need to set up the configs for it.

    cd /
###

    cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
###

Fail-to-ban is good to go out the box, but if you do need to add any custom configurations, you would make them in the jail.conf file you just created.

### Hetzner Plesk Servers

On Plesk servers a few things need to be installed, so log into your Plesk control panel and go to:

    Tools & Settings -> Plesk -> Updates

It should have opened a page in a new tab, find the following and mark it for install if not already installed

    Fail2Ban
	
    Web Hosting -> ModSecurity
    
    Plesk extensions -> Plesk Firewall
    
    Plesk extensions -> ImunifyAV

Click continue and let plesk install that.

Next, go to

    Tools & Settings -> Security -> Security Policy

    Select allow only secure FTPS connections and apply

Next, go to

    Tools & Settings -> Security -> Firewall

Enable the firewall if not already, the default rules are ok

Next, go to

    Tools & Settings -> Security -> IP Address Banning (Fail2Ban) -> Settings

Toggle “Enable intrusion detection” on and click apply, the default rules are good.

### LIVE SERVERS ONLY

This next one is for production servers, or for pen testing. Leave this off on dev servers to spare all the false positives amending files, and getting locked out.

    Tools & Settings -> Security -> Web Application Firewall

    Turn it on

    Use Comodo (free)

    check to update daily

    Set the predefined set of values to “Fast”

Click ok and let it update, but if this is a dev server, seriously, turn this off or you’ll keep getting locked out after uploading changes.
