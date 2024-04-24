# A Comprehensive Guide On WireGuard/DNSCrypt/SSH/Honeypot Implementation on OVH (Or any other Debian server) #

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

### Firewall Setup ###

I like to use the [ufw](https://wiki.ubuntuusers.de/ufw/) (uncomplicated firewall) programm
for it. It handles all iptables settings we have to make in order to secure our server.

If you want to see the current status of ufw just type `sudo ufw status verbose`.

Here are our first rules we need before we start ufw:
-  `sudo ufw default deny incoming` - denys all incoming traffic by default
-  `sudo ufw allow 22/tcp` - allow ssh connection

Now we can start ufw by `sudo ufw enable`.

## DNSCrypt ##

*UFW*: Run `ufw allow proto udp from 127.0.0.1 port 53`

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

*UFW*: Run `ufw allow proto udp from 10.0.0.1 port 53`

Now to finally install WireGuard, this is achieved by issuing `apt-get
install wireguard`. Ensure the service is installed and running by
issuing `modprobe wireguard` and `lsmod | grep wireguard`. 

For the latter command you should see something along the lines of:
```
wireguard             225280  0
ip6_udp_tunnel         16384  1 wireguard
udp_tunnel             16384  1 wireguard
```

The next step is to generate the key pair, but first change the permission of the
`/etc/wireguard/` directory with `umask 077`. This will ensure that only
the owner is able to read or execute newly-created files. Now the actual
key generation, issue `wg genkey | tee privatekey | wg pubkey > publickey`.

From this point on you can cheat by going to [wireguardconfig.com](https://www.wireguardconfig.com/) and
using a generated configuration. But is not that difficult to set it up yourself, 
start with creating the following file `/etc/wireguard/wg0.conf` and adding your 
own private key and a client's public key to the following configuration in the 
image below (It is best to do both the client and server steps at the same time). 
Then save it and modify its permissions with `chmod 600 /etc/wireguard/wg0.conf`. 

Now create the `wg0.conf`:
```
[Interface]
PrivateKey = INSERT YOUR PRIVATE KEY HERE
Address = 10.0.0.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820

[Peer]
PublicKey = INSERT CLIENT PUBLIC KEY HERE
AllowedIPs = 10.0.0.2/32
```

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
ListenPort = 51820
Address = 10.0.0.4/24
DNS = 10.0.0.1
[Peer]
PublicKey = SERVER PUBLIC KEY
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = SERVERIP:51820
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

## Additional Security Post Installation ##

### SSH via WireGuard (With Knocking) ###

Port knocking is great, but why allow anybody from any IP address to
knock at all? Why not limit the knocks to those already on the WireGuard
network, this way you can ensure that only those you can trust can even
begin the knocking process. This is achieved by simply modifying the
knockd configuration file, as per above, but changing the interface to wg0.
This is done by simply adding `interface = wg0` in the options section.
If you wanted to be even more paranoid, you could set up an additional
WireGuard interface specifically to access SSH and use that as the
knocking interface, this would allow sharing of the WireGuard VPN access
but also ensuring your own secure access on a different interface and IP
address, solely for SSH.

### Connection Profiling ###

Websites detect your Maximum Transmission Units (MTU) and use that against you, this is a known method to fingerprint connections. Wireguard uses the default MTU of `1420` which is also the default of IPSec. So if a website does not allow VPN connections, this is their point of call. Now what would be your new MTU? It is important to get this value right. You can use websites like http://www.letmecheck.it/mtu-test.php to determine the best one for a particular website/service but I found that `1480` seems best. It will get fingerprinted as a IPIP/SIT tunnel, so a Linux virtual interface - which is better than IPSec. 

To change the MTU on the fly you can simply run `sudo ifconfig wg0 mtu 1480 up`. Confirm it has been changed with `netstat -i` or with `ifconfig`. From here you can make the value permanent by adding it to the `wg0.conf` file by simply adding `MTU = 1480` within the interface section. Note you must change the MTU in your client also, else you will experience dropouts. 

![](media/image24.jpeg)

## Other Considerations ##

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

Your ASN could easily just be banned from websites you're trying to visit, this is though more common with public VPN's as their anonymity allows for abuse and thus for a service to block one IP would be meaningless, so they ban the entire VPN provider, or at least a large range of their servers.

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

## Tunneling aka VPN Chain aka Double-VPN ##

OK, so you're confident with this guide and you want to step it up a bit? You want to connect through two VPNs before accessing the internet?
Well first step is to literally do this guide TWICE, it's surely not that hard at this point, right?

Now you have two server servers we will call them server1 and server2, the first is what you will connect to, the second is what ultimately will be your external IP.
Thus it will be: `You ---> server1 ---> server2 ---> Internet`.

On server1, create a second wireguard interface, `wg1` and generate a new set of public and private keys with `wg genkey | tee privatekey | wg pubkey > publickey`. 
You do this by making `nano /etc/wireguard/wg1.conf`. Within this interface file insert the following (Note, the IP address does not have to be on a different subnet, remember, this interface is essentially just another client):

-   `[Interface]`
-   `Address = 10.0.0.4/24`
-   `PrivateKey = theprivatekeyyoujustmade`
-   `FwMark = 51280`

-   `[Peer]`
-   `PublicKey = thepublickeyofserver2`
-   `AllowedIPS = 0.0.0.0/0`
-   `Endpoint = server2IPAddress:51820`
-   `PersistentKeepalive = 21`

Now edit your server1 `wg0` configuration file and add `FwMark = 51820` much like `wg1` has. Save and close, server1 configuation complete!

Now you must add these routes using the following commands:

-   `echo "1 wg1" >> /etc/iproute2/rt_tables`
-   `ip route add 0.0.0.0/0 dev wg0 table wg1`
-   `ip rule add from 10.0.0.0/24 lookup wg1`

Now go into your server2 and edit its `wg0` file and add a new peer using server1's public key (that you made earlier) and give it the IP address of `10.0.0.4/32` (or whatever matches the configuration you just made).

Restart server2's `wg0` either fully with `wg-quick down wg0` and then `wg-quick up wg0` or just simply run `wg addconf wg0 <(wg-quick strip wg0)`.
server2 will now be ready to recieve server1, so go back into server1 and do the same thing to its `wg0`. At this point you can finally start `wg1`.

If all went well server1's external IP address will now be that of server2. Try it out with `curl whatismyip.akamai.com`.

The client can now connect to server1, which will connect through server2! Mission Complete! You can also obviously connect to either of them individually still.
Disabling `wg1` on server1 will not cause any issue, the connection will just be as it was before. So you can essentially turn this ability on and off at your will.

On the topic of turning it on and off at your will, here is a nice and clever method that does not involve you logging into SSH. Just use the `knockd` service!
Start by editing `/etc/knockd.conf` and adding another item much like how SSH is done, check out this example - note that each and every port is different, if you use the same port the sequence will get broken and it wont work. Don't forget to restart the service when you're finished editing the config file, you can do that with `systemctl restart knockd.service`.

![](media/wg1knockdconf.png)

Basically `knockd` can do more than just enable and disable your SSH port, you can make it do whatever command you wish, so why not use it to toggle your double-vpn!
Once the ports are knocked you can see in the log `/var/log/knockd.log` the sequence that it did and the command it ran. You can use your imagination for what other scripts or commands you could run.

![](media/wg1knockd.png)

### Port Knocking ###

Now it's time to setup port knocking. This will ensure that along with a
different SSH port number, it will remain blocked in the iptables (to be
setup next) unless a specific sequence of ports are 'knocked'. Only then
the iptables will allow the SSH port to be open to the IP address of the
knocker. So the first step is to simply install the service required
with the command `apt-get install knockd`. Now before its run its
important to modify the default settings, the service even has a
starting flag hidden away in a different file that must be changed prior
to being started for the first time. The configuration file is in
`/etc/knockd.conf` and this is my recommended bespoke settings:

![](media/image10.jpeg)

What these settings achieve is the need to knock in the sequence 1337,
8888 and 1200 within 5 seconds of each other (Can be any sequence and
amount of ports you wish). When this is done the IP address of the one
who knocked will be added to the iptables, allowing exclusive access to
port 88 (or whatever port you set SSH to). It will also only accept TCP
SYN packets. After 15 seconds the IP that was added to the iptables will
be removed -- this won't log you out of the SSH session but it will
prevent you from logging back in without knocking again, this is easier
than the default settings which require the user to knock the SSH port
shut, which can be easily forgotten.

Now access the following file `/etc/default/knockd` and change
`START_KNOCKD` to `1`. Then start the knockd service by running
`systemctl start knockd`. From this point onwards you will not be able
to access the SSH normally (Well after you've finished setting up IP Tables). 
Note that according to the rules, the IP that knocked is the IP that will be 
added to the iptables -- remember to be mindful of this. 
This will be modified again after WireGuard is installed. 

Once this is set up install knockd on another Linux machine
and issue the command `knock -v IP PORT1 PORT2` to open SSH or for
Windows download the application `BwE Port Knocker` (available on my
GitHub) which I developed for this very write-up. This will be vital to
getting back into the server. Again, should there be a situation where you
cannot login you still can via the KVM console.

![](media/image11.jpeg)

## Troubleshooting ##

During my time with this setup I have found and discovered various small issues, 
here are my quick fixes for them.

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

### Knockd Not Opening Port ###

Well if you've not modified the interface for knockd, it could simply be because you're on the VPN.
Its default settings are to accept knocks from eth0, if you're on the VPN you're on wg0. It won't work.
If it works when you disconnect from the VPN you should add the wg0 interface into knockd if you prefer accessing it this way.

### Knockd Not Starting at Boot ###

Confirm the issue with `systemctl is-enabled knockd.service`, if it comes up as `static`
then edit `/lib/systemd/system/knockd.service` and add this to the bottom:

-   `[Install]`
-   `WantedBy=multi-user.target`
-   `Alias=knockd.service`

Run the following: `systemctl enable knockd.service` and `systemctl is-enabled knockd.service`
it should now come up as `enabled`.

It may still not start due to the sequence of how things start up, so it may error out saying
that there is no wg0 interface. If this is the case you need to make a cron job that restarts it
after wireguard starts.

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

