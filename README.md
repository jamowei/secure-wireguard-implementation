# A Comprehensive Guide On WireGuard/DNSCrypt/SSH on Debian server #

### Introduction ###

The plan in this guide is to create a secure WireGuard VPN which has its
own embedded DNSCrypt DNS resolver, this ensures that all connections
including DNS requests made by the user are tunnelled through the server
and is encrypted end to end. This is also expanded to include security
that resolves around this process making the server as secure as
possible from external agents. The following represents the proposed
network diagram for this guide.

![](media/image1.jpeg)

Why use WireGuard? As you can see in the image after this paragraph,
whilst on the WireGuard VPN speed decrease against a direct connection
to the internet is negligible (~3Mbps), this is because WireGuard runs
within the kernel space and thus ensures the secure tunnel can run at
high speed, it is even now part of the latest Linux Kernel 5.6. But
while this was my personal reason for implementing WireGuard there is
also the benefits in its simplicity in both development, with a lean
codebase of 4000 lines (compared to 100,000 in OpenVPN) but also in its
implementation -- which will be illustrated in this guide.
Fundamentally, you install the service and a client and exchange keys;
it can't be easier than that. WireGuard also supports better
cryptographic methodologies than OpenVPN and easier to expand and
distribute among peers.

![](media/image2.png)

### Setting up the server ###

Be sure to issue `apt update` and `apt upgrade` and do this regularly. I also recommend changing
the password that they give you and securing your account in the method you prefer.

The first thing to do is to now secure the SSH connection and ultimately
customise it.

Start by installing fail2ban, an active intrusion detection system
designed to ban brute force attempts towards your SSH. Issue the
following commands to install fail2ban:

-   `apt install fail2ban`

-   `cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local`

-   `cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`

Once this is done you'll need to modify the `/etc/fail2ban/jail.local`
file to adjust the time limits, this will mean that users whom attempt
to login to your SSH will get banned for x period of time after x
attempts etc.

![](media/image4.jpeg)

From here you can issue the command `systemctl status fail2ban.service`
to confirm the service is indeed running. You can also view the IP's
that are currently banned with the command `fail2ban-client status
sshd`, through the iptable rules or view the log file directly at
`/var/log/fail2ban.log`.

![](media/image5.jpeg)

Another important step is to change the default SSH port of 22 to
something else. This will aid in preventing automated bots from scanning
your server, though it would not prevent somebody from discovering it
eventually. This can be done by modifying the SSH config file at
`/etc/ssh/sshd_config`. Don't forget to update your fail2ban to suit the
change (and restart it), but I have noticed that fail2ban does not seem to enjoy being
modified after the fact. If this is the case for you just modify the
iptables manually. To delete the original rule, find its number with
`iptables -L -v -n --line-numbers` and deleting it with `iptables -D
INPUT #`. Now to add your bespoke SSH port with fail2ban issue the
following command `iptables -A INPUT -p tcp --dport SSHPORT# -j
f2b-sshd`.

![](media/image6.jpeg)

### Who Are You? ###

Well okay, so you've setup fail2ban and the whole time you've likely been root.
Well for the next phase you have the choice to continue entirely as root, or make
a seperate account and sudo root instead. The security implications are for you to
research, but in my case I have chosen to login to the server not via root, but
with an account with no sudo access whatsoever. From there I log into root.
I feel this adds another tiny extra little itsy bitsy layer of protection should my
keys for SSH be divulged. So what did I do? Well first I changed the default root password
with `sudo passwd` then I made a new blank user with `sudo adduser debian`. Now I can
swap between the accounts by issuing the `su` command. So `su debian` or just `su` to get back
to root. Easy!

### Password-free Entry ###

The next step is to be able to login to the server without a password or
even a password prompt (although this is optional), thus we would need
to use SSH key pairs.

Use the command `ssh-keygen -t ed25519 -C "your_email@example.com"` on your machine to get an secure SSH key pair.
You should see private key `.ssh/id_ed25519` and public key `.ssh/id_ed25519.pub` in your home directory.

Now you have to upload the public key to the `/home/<username>/.ssh/authorized_keys` file on the server.
This can be easily achieved by `ssh-copy-id <username>@<server-ip>`.
Or you create the file first `touch .ssh/authorized_keys` and set the permissions `chmod 700 .ssh/authorized_keys`.
Then you have to copy the public key in `.ssh/id_ed25519.pub` and paste it to the 
`/home/<username>/.ssh/authorized_keys` file on server.

After this you can disable password authentication within the SSH config file on server `/etc/ssh/sshd_config`.
Modify the following two settings `PasswordAuthentication no` and `UsePAM no`. 
At the bottom of the configuration file add the following `AuthenticationMethods publickey`.
If you are not going to log in as root, it is important to change the setting here, so set `PermitRootLogin no`. 

At this point you should restart the SSH service with the command
`systemctl restart sshd`. Open another SSH session with the
appropriate private key added and attempt to connect to the server. If
all goes well you will be prompted for a username and will be instantly
logged in.

### Firewall Setup (With ufw) ###

I like to use the [ufw](https://wiki.ubuntuusers.de/ufw/) (uncomplicated firewall) programm
for it. It handles all iptables settings we have to make in order to secure our server.

If you want to see the current status of ufw just type `sudo ufw status verbose`.

Here are our first rules we need before we start ufw:
-  `sudo ufw default deny incoming` - denys all incoming traffic by default
-  `sudo ufw allow 22/tcp comment ssh` - allow ssh connection

Now we can start ufw by `sudo ufw enable`.

## DNSCrypt ##

*UFW*: Run `ufw allow proto udp from 127.0.0.1 port 53 comment localhost to dns`
If you want other clients on your local network to use the dns proxy, 
you have run `ufw allow proto udp from <network> port 53 comment network to dns`

Installing this on the server allows full ownership over DNS traffic both for your Wireguard client/s and the local network. 
Your DNS traffic will be forwarded to DNSCrypt which will in turn facilitate DNSSEC and the encryption of DNS requests. 
Now the next thing to step is to install [DNSCrypt-Proxy](https://github.com/DNSCrypt/dnscrypt-proxy) itself.

You must first add the repositories required for installing either the testing or unstable version
this is done by running the following two commands.

`echo "deb https://deb.debian.org/debian/ testing main" | sudo tee /etc/apt/sources.list.d/testing.list`

Start with `sudo apt update` and then `sudo apt install -t testing dnscrypt-proxy`

It is recommended then to reset and delete `rm /etc/apt/sources.list.d/testing.list`.

The next step then is to configure DNSCrypt, first edit `/etc/dnscrypt-proxy/dnscrypt-proxy.toml`
and modify the `listen_address` line to be `[]`.

Now to add your preferred DNS providers:

-  `server_names = ['doh-eastas-pi-dns', 'doh.tiarap.org', 'quad9-dnscrypt-ipv4-filter-pri', 'quad9-doh-ipv4-filter-pri', 'doh-eastau-pi-dns', 'adguard-dns-doh']`

This allows for ad blocking DNS servers to be selected and deduced based on their ping. You could also just use regular DNS servers by using the following:

-  `server_names = ['deffer-dns.au', 'publicarray-au', 'publicarray-au2', 'publicarray-au2-doh', 'publicarray-au-doh', 'cloudfare']`

Now you're wondering where am I getting these DNS names from? Well you can make your own list from https://dnscrypt.info/public-servers/.

The rest of the recommended settings after `server_names` entry:
```
ipv4_servers = true
ipv6_servers = false
dnscrypt_servers = true
doh_servers = true
require_dnssec = true
require_nolog = true
require_nofilter = false
fallback_resolvers = ['9.9.9.9:53', '8.8.8.8:53']
```

Obviously these settings are not everything, but this is what I recommend you change/add from the default.

Now download the relay and public resolvers files, because apparently DNSCrypt does not do it for you:
-  `wget https://download.dnscrypt.info/dnscrypt-resolvers/v3/relays.md -P /etc/dnscrypt-proxy/`
-  `wget https://download.dnscrypt.info/dnscrypt-resolvers/v3/public-resolvers.md -P /etc/dnscrypt-proxy/`

Now you need to modify the before mentioned `dnscrypt-proxy.socket` service to point to the correct IP address, so edit `/lib/systemd/system/dnscrypt-proxy.socket`
and modify `ListenStream` and `ListenDatagram` to be `0.0.0.0:53`. Having it at 0.0.0.0 rather than 127.0.0.1 means that other interface like `wg0`
will be able to access it. You can point your clients to use this DNS by changing their configuration to point to the DNS to the server.

At this stage  you need to modify your systems default resolve file, but first back it up  
with `cp /etc/resolv.conf /etc/resolv.conf.backup` then edit it and change to 
```
nameserver 127.0.0.1
ptions edns0
```
You are no longer using your default DNS!

Your system will likely try and revert these settings so lock the file with 
`chattr +i /etc/resolv.conf`, note that the `-i` switch will unlock it. 
You should also modify `/etc/systemd/resolved.conf` and uncomment or add
`DNSStubListener=No` this may help prevent port clashing in the future.

Once this is all done, you should restart the service daemon with `systemctl daemon-reload`.
Now you can restart the DNSCrypt service with `systemctl restart dnscrypt-proxy`.

Check if its running with `systemctl status dnscrypt-proxy`. If it all went well it should look like this:

![](media/dnscryptgood.jpg)

Now you should reboot.

### Testing ###

Now run `dnscrypt-proxy -resolve google.com -config /etc/dnscrypt-proxy/dnscrypt-proxy.toml`.
If this succeeded you are good to go! If it didn't you probably didn't listen to me when I said restart, so go ahead and `sudo reboot`.

Feel free to make a final confirmation test of the DNS by running
`nslookup -q=A whoami.akamai.net` and looking at the respondant IP, thats your DNS.
Once you have wireguard setup can also go to `www.dnsleaktest.com` on your client device to see which server/s you're using.

Don't have nslookup? `apt install dnstools`, you will likely need this in the future anyhow.

Another test you can do for the client side is to simply stop the DNSCrypt service 
via `systemctl stop dnscrypt-proxy.service` and `systemctl stop dnscrypt-proxy.socket`, 
if websites timeout and you can still access `https://1.1.1.1/` then all is well!

## WireGuard ##

*UFW*: Run `ufw allow proto udp from <wireguard-network> port 53 comment wireguard to dns` and `ufw route allow in on wg0 comment wireguard`
Also you need to allow the communication to wireguard service itself `ufw allow <wireguard-port>/udp comment wireguard`

Now to finally install WireGuard, this is achieved by issuing `apt-get
install wireguard`. Ensure the service is installed and running by
issuing `modprobe wireguard` and `lsmod | grep wireguard`. 

For the latter command you should see something along the lines of:
```
wireguard             225280  0
ip6_udp_tunnel         16384  1 wireguard
udp_tunnel             16384  1 wireguard
```

You can use [wireguardconfig.com](https://www.wireguardconfig.com/) to generate a all config files.
Place the server config to `/etc/wireguard/wg0.conf`. 
Then save it and modify its permissions with `chmod 600 /etc/wireguard/wg0.conf`. 

Subsequent clients are added below each other with the same formatting, to then remove a user you issue 
`wg set wg0 peer CLIENTPUBLICKEY remove` or modify the wg0.conf manually. To load a
configuration (to add another client for example) without resetting the
service run `wg addconf wg0 <(wg-quick strip wg0)`.

Now ensure that your system can accommodate IP forwarding by editing
`/etc/sysctl.conf` and adding `net.ipv4.ip_forward=1` and `net.ipv6.conf.all.forwarding=1`. 
Once this is done run `sudo sysctl -p` to load your newly edited configuration. 
Now you can finally start WireGuard with `sudo wg-quick up wg0` and confirm its running with 
`wg show`.

![](media/image19.jpeg)

Connect to the server via WireGuard to finally confirm that you are indeed
part of the server's LAN, this is important for a final security measure. If all is well make
WireGuard start at boot with `systemctl enable wg-quick@wg0`. You can confirm that there is indeed
encrypting traffic by issuing `tcpdump -n -X -I eth0 host YOURSERVERIP` 
and looking for WireGuard's magic header identifier in each packet `0400 0000`.

![](media/image20.jpeg)

### Windows Client Side Setup ###

Running the official WireGuard client for Windows, you are able to
create a new tunnel with a few clicks. Within the client click on the
down arrow next to `Add Tunnel` and create a new tunnel. You will be
presented with a form with part of a configuration along with a
pre-generated public and private key, as seen below.

![](media/image21.png)

Use this information to build your client configuration as per the
second screenshot above. Once this is done, click `Activate` and you
will be connected!

### Linux Client Side Setup ###

From the client's perspective setting up WireGuard is very similar, it
starts with the following commands 'apt install wireguard resolvconf' to
install the service and to ensure the DNS functionality, then to confirm
its running run `lsmod | grep wireguard`. Now you have to generate a
private and public key, the private key stays on your system and the
public key is given to the server so you can become a peer. This is done
with `wg genkey | tee privatekey | wg pubkey > publickey`. Then
change the permission of the `/etc/wireguard/` directory with `umask
077`. Create now your own WireGuard configuration file in
`/etc/wireguard/wg0.conf` and insert the following.

```
[Interface]
PrivateKey = CLIENT PRIVATE KEY
ListenPort = 123
Address = 10.0.0.4/24
DNS = 10.0.0.1
[Peer]
PublicKey = SERVER PUBLIC KEY
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = SERVERIP:123
```

The AllowedIPs setting can be changed to permit LAN access or can
restrict to a specific IP address. Once this is done change the
permissions of the file with `chmod 600 /etc/wireguard/wg0.conf` and
then ensure that your system can accommodate IP forwarding by editing
`/etc/sysctl.conf` and adding `net.ipv4.ip_forwarding=1` and
`net.ipv6.conf.all.forwarding=1`. Now you can start the connection with
`wg-quick up wg0` and confirm its running with `wg show` and finally, to
ensure it starts at boot run `systemctl enable wg-quick@wg0`.

### Adding New People Post-Installation ###

Well if you've figured it out by now new clients need a matching public and private key.
This can be generated by the server (or any linux machine running wireguard) with the command
`wg genkey | tee privatekey | wg pubkey > publickey`. If you're using the mobile app you can
do the same, just click `Create from Scratch` and in the interface section click the refresh arrows
and it will generate your private and public key. Then fill out the interface name (whatever you wish)
the IP address of the server, its port, DNS server and MTU. Then you're done. On the server side the public 
key must be added to your `wg0.conf`. So simply add another `[Peer]`, add their public key and assign their IP address.

Another way of doing this for mobile users is to generate the public and private keys for them and essentially create a configuration
and place it into a QR code for them to read. Then you add the public key youve generated into the `wg0.conf` as above.

For Windows users it is the same, but you have to write out the client configuration from scratch, you surely know how to by know!

Once this is all done you need to refresh the configuration (albeit without resetting the interface) with the command
`wg addconf wg0 <(wg-quick strip wg0)`.

## Installing NoMachine ##

*UFW*: Run `ufw allow proto udp from <wireguard-network> port 4000 comment nomachine` 
and `ufw allow proto tcp from <wireguard-network> port 4000 comment nomachine`

I really like to see the desktop environment on my remote server. That is why I use [nomachine](https://www.nomachine.com/).
First download the Debian package via `wget https://download.nomachine.com/download/8.11/Linux/nomachine_8.11.3_4_amd64.deb`.
Then you can install it by `sudo dpkg -i nomachine_8.11.3_4_amd64.deb`.
Now the service is started and listening on port 4000. We have to edit `/usr/NX/etc/node.cfg` to set the right desktop session we want.
Therefore we have to find the property `DefaultDesktopCommand` and change it accordingly.

- CINNAMON>	DefaultDesktopCommand "/usr/bin/cinnamon-session --session cinnamon"
- MATE>	DefaultDesktopCommand "/usr/bin/mate-session"
- LXDE>	DefaultDesktopCommand "/usr/bin/startlxde"
- XFCE>	DefaultDesktopCommand "/usr/bin/startxfce4"
- UNITY>	DefaultDesktopCommand "/etc/X11/Xsession 'gnome-session -session=ubuntu' "

Save the file and restart the nxserver via `/etc/NX/nxserver --restart`.

### Use Key based Authentication ###

Create a file for containing public key via `touch .nx/config/authorized.crt`. Then you can past you public key in there.
After trying out that it works (don't forget to change the auth method on client side). You can disable the password authentication by
ediit the `/usr/NX/etc/server.cfg`. There you have to set `AcceptedAuthenticationMethods NX-private-key`.
Then restart the service via `/etc/NX/nxserver --restart`.

### Use Virtual Display ###

If you want to use the virtual display instead, have to stop the lightdm service by `systemctl stop lightdm` first.
Then you have to restart the nxserver via `/etc/NX/nxserver --restart`.

### Enable Sound ###

If you want to transfer the sound also, you have to run `/usr/NX/scripts/setup/nxnode --audiosetup`.

## Installing Podman ###

To install [podman](https://podman.io/) you only need to run `apt install podman podman-compose`
The `podman-compose` plugin allows you to use a `docker-compose.yaml` to manage all your container.

If you want to export standard ports without need to start podman as root, you have to change the
`/etc/sysctl.conf` and set `net.ipv4.ip_unprivileged_port_start=80`.
To apply this setting run `sudo sysctl -p`.
Then you can run your `docker-compose.yaml`via `podman-compose up -d`.

*UFW*: Don't forget to open the specific port you need with `ufw allow 80/tcp comment my-app`.

## Additional Security Post Installation ##

### Connection Profiling ###

Websites detect your Maximum Transmission Units (MTU) and use that against you, 
this is a known method to fingerprint connections. Wireguard uses the default MTU of `1420`
which is also the default of IPSec. So if a website does not allow VPN connections, 
this is their point of call. Now what would be your new MTU? It is important to get this value right.
You can use websites like http://www.letmecheck.it/mtu-test.php to determine the best one for
a particular website/service but I found that `1480` seems best. It will get fingerprinted as a IPIP/SIT tunnel,
so a Linux virtual interface - which is better than IPSec. 

To change the MTU on the fly you can simply run `sudo ifconfig wg0 mtu 1480 up`. 
Confirm it has been changed with `netstat -i` or with `ifconfig`.
From here you can make the value permanent by adding it to the `wg0.conf` file
by simply adding `MTU = 1480` within the interface section. 
Note you must change the MTU in your client also, else you will experience dropouts. 

![](media/image24.jpeg)

### Logs ###
I highly suggest installing `lnav` for log aggregation. But prior to doing this remember
to change your time-zone with `sudo timedatectl set-timezone your_time_zone`.

If for some reason you do not want logs I suggest running the following commands
or at least setting them up to run on a schedule in the background (see Cron Jobs):

-   `cat /dev/null > ~/.bash_history`
-   `for logs in ``find /var/log -type f``; do > $logs; done`
-   `sudo service rsyslog restart`

### Cron Jobs ###
Cron jobs are tasks that can be set automatically by the system. One that we can make that
is relevant to this project is the apparent need to make the knockd service restart itself
after the system reboots (due to wireguard starting too late) and fixing the wg0 interface MTU.

For this example start with creating a simple bash script, `nano hello.sh`.

-   `#!/bin/bash`
-   `logger "=== Boot Sequence Quick-Fix ==="`
-   `sudo wg-quick up wg0`
-   `sudo ifconfig wg0 mtu 1480 up`
-   `sudo sysctl net.ipv4.ip_default_ttl=30`
-   `sudo systemctl restart dnscrypt-proxy`
-   `sudo systemctl restart knockd.service`
-   `logger "=== Quick-Fix is Done! ==="`

Now just save the file and change its attribute to executable with `chmod 700 hello.sh`.
To have your script run when the system starts up you have to open the crontab editor.
Do this with `crontab -e`, it will prompt you to choose an editor, go with nano, or option 1.
Within this page type in `@reboot sleep 60 && /home/wherever/your/script/is/hello.sh`.
Save and close. 

Now you have to enable the cron service with `systemctl enable cron.service`.
You can see your user cron jobs with `crontab -l` and you can see the history of your cronjobs 
with `systemctl status cron.service` or ideally `sudo grep CRON /var/log/syslog`. And thats that!

Be careful with what you put in these scripts, because they are run as root (depending on your permissions) they are very powerful.

## Troubleshooting ##

During my time with this setup I have found and discovered various small issues, 
here are my quick fixes for them.

## Blocked by Autonomous System Number (ASN) ##

You must be aware of the Autonomous System Number (ASN) that is assigned
to your server IP address when you buy your server. Mine for example is
AS16276, belonging to OVH SAS in Canada, its purpose is for paid VPN,
hosting and 'good' bots. It has however 40,382 active spam IP addresses
out of a total of 381,412 -- that's 10.5% of the entire network
consisting of spammers, this is not good and was likely the reason
behind having multiple DDoS attacks when my IP was still new.

![](media/image27.jpeg)

What this means is that should you indeed use your server as a VPN for
daily use, you may find that you have been banned from websites you have
never visited, this is simply because the website has chosen to ban not
your IP address but your ASN entirely. Luckily the IP address I have is
unique to my account and is not shared, this situation would be worse on
a commercial VPN provider, where allocated shared IPs can be banned in
global black lists due to spamming and other illegal activity. My ASN is
also not part of the Spamhaus Project ASN-DROP list; if it were then I
would certainly not continue using my hosting provider.

![](media/1005.png)

Your ASN could easily just be banned from websites you're trying to visit, 
this is though more common with public VPN's as their anonymity allows for
abuse and thus for a service to block one IP would be meaningless, 
so they ban the entire VPN provider, or at least a large range of their servers.

### Unable to Locate Package: Wireguard? ###

Add these to the bottom of your `/etc/apt/sources.list`

-   `deb  http://deb.debian.org/debian  stretch main`
-   `deb-src  http://deb.debian.org/debian  stretch main`
-   `deb http://ftp.debian.org/debian buster-backports main`
-   `deb-src http://ftp.debian.org/debian buster-backports main`

Then run `apt update`.

### Wireguard Cannot Compile ###

In `/usr/src/wireguard-1.0.20200623/socket.c` add these two lines after the #include's:

-   `#undef ipv6_dst_lookup_flow`
-   `#define ipv6_dst_lookup_flow(a, b, c, d) ipv6_dst_lookup(a, b, &dst, c) + (void *)0 ?: dst`

Now as root run `/usr/lib/dkms/dkms_autoinstaller start`

If this does not work read the following and follow it `https://www.wireguard.com/compilation/`

Now remove lines 95, 96, 97 and 99 from `compat.h`
Compile and install as per the official guide

### Wireguard Won't Start ###

![](media/wgfail.png)

If you're getting an error like `RTNETLINK Operation Not Supported` when trying to start `wg-quick up wg0` 
you need to input `sudo modprobe wireguard`. If the resultant answer is something along the lines of 
`Badprobe: FATAL: Module wireguard not found in directory ...`  then your solution is to run 
`apt-get install wireguard-dkms wireguard-tools linux-headers-$(uname -r)`.

### DNSCrypt Not Starting at Boot (Or at all) ###
Confirm if port 53 is not already in use by something else with `lsof -i -P -n | grep LISTEN`
Kill the PID of whatever is already using that port.
If it is Avahi you can disable it from booting with the following commands:

-   `systemctl stop avahi-daemon.socket`
-   `systemctl stop asystemvahi-daemon.service`
-   `systemctl disable avahi-daemon`

Still not working? 
If it says `can't bind socket` or `could not open ports` try running `netstat -patuln | grep 53`.
If you see `1/init` using port 53 then you need to run `systemctl stop dnscrypt-proxy.socket`
and then restart dnscrypt again. This should fix it.

Still not working? Well I guess systemd is using port 53. You can disable it by running
`systemctl stop systemd-resolved` and `systemctl disable systemd-resolved`.
You don't really need it given you're using DNSCrypt.

### Some Websites Timeout/Cannot Resolve After Reboot ###
If you have changed the MTU it likely went back to the default and thus your client side
settings are not matching, causing dropouts.

If this is not the issue change the wg0 MTU to `1480`.
Again, ensure that your client matches the MTU settings of the wg0 interface.

