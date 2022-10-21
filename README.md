# Wireguard QR script
## Automate creating WireGuard peers

This script helps you to add peers in Wireguard server config (wg0.conf) and create configs for peers.
So new Wireguard clients can easily connect with text or QR configs.

Installation:

$ sudo apt install qrencode
$ cd /etc/wireguard/

clone this repository to /etc/wireguard/ or other directory where wireguard installed

$ cd /etc/wireguard/clients

put in file "state" ip adress of the last peer in wg0.conf, or server IP if there is no peers added. For new peers ip addresses will be added as +1 to last peer IP.

There is template of peers config in client.conf, like:

[Interface]
PrivateKey = privatekeytoreplace 
Address = 10.10.0.0/24
DNS = 1.1.1.1, 1.0.0.1
MTU = 1380

[Peer]
PublicKey = SERVER_PUBLIC_KEY
AllowedIPs = 0.0.0.0/0
Endpoint = 123.234.56.78:51820
PersistentKeepalive = 7

You should change SERVER_PUBLIC_KEY and 123.234.56.78:51820 to your real values, and change other settings you need.

Open wg_gencli.sh:

$ nano wg_gencli.sh

and change variables IP_RANGE and WG_CONF_PATH if you need.

Run wg_gencli.sh with name of new peer: 

$ ./wg_gencli.sh newpeer

## WireGuard-go on Ubuntu OpenVZ server 

In result it should create new directory "newpeer" that containes:

client.conf - peer config file
client.key - peer private key
client.key.pub - peer public key
clinet.png - QR-code to connect mobile app

and show new peer text and QR configs in console, so you can instantly connect client app. 

This may help (I think it's the only way) to run Wireguard on cheap VPS with an outdated kernel which you have no access to update (like OpenVZ), incompatible with Wireguard.

apt-get update && apt-get -y upgrade
apt-get -y install nano bash-completion wget git
apt-get -y install software-properties-common
add-apt-repository ppa:wireguard/wireguard
apt-get update && apt-get -y upgrade
apt install wireguard-tools --no-install-recommends

Then, to enable forwarding:

nano /etc/sysctl.conf
And uncomment the following lines:

net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

You could do

sysctl -p

just makes sure all updates/upgrades/changes are in place properly

reboot

Now, we are going to use wireguard-go, so need to install "go". Check it on https://go.dev/dl/, but just change "go1.19.2" in each of the following lines if wish to try a differen version:

cd /tmp
wget https://go.dev/dl/go1.19.2.linux-amd64.tar.gz
tar zvxf go1.19.2.linux-amd64.tar.gz
mv go /opt/go1.19.2
ln -s /opt/go1.19.2/bin/go /usr/local/bin/go
Now, download and install wireguard-go itself

cd /usr/local/src
git clone https://git.zx2c4.com/wireguard-go
cd wireguard-go

If you are on a system with limited RAM (such as a 256 MB or lower "LowEndSpirit" VPS), you will need to do a small tweak to the wireguard-go code to make it use less RAM. Open device/queueconstants_default.go and replace this:

MaxSegmentSize             = (1 << 16) - 1 // largest possible UDP datagram 
PreallocatedBuffersPerPool = 0 // Disable and allow for infinite memory growth

With these values (taken from device/queueconstants_ios.go):

MaxSegmentSize             = 1700 
PreallocatedBuffersPerPool = 1024

Now we can compile it:

make
cp wireguard-go /usr/local/bin
Reboot to ensure everything is clean

reboot
Check to see if working/version

wireguard-go --version

Generate private and public keys

cd /etc/wireguard/
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
#write down the public key:
cat publickey
#write down the private key:
cat privatekey

Define wg0 interface by creating "wg.conf".

cd /etc/wireguard/
nano wg0.conf

And copy the following into the file (changing private key and address as appropriate). An example address would be 10.10.1.1/24. 

[Interface]
PrivateKey = <privatekey>
Address = 10.10.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
SaveConfig = true

One note, you may need to change eth0 to your proper interface, you can find it with:
  
ip a
  
We need to modify the systemd unit to pass the WG_I_PREFER_BUGGY_USERSPACE_TO_POLISHED_KMOD flag to wireguard-go, to allow it to run on Linux. Open 
  
nano /lib/systemd/system/wg-quick@.service
  
find:

Environment=WG_ENDPOINT_RESOLUTION_RETRIES=infinity

and add this line directly below:

Environment=WG_I_PREFER_BUGGY_USERSPACE_TO_POLISHED_KMOD=1 

Enable and start the service
  
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
systemctl status wg-quick@wg0.service

  Assuming your client is set up correctly, all should flow. Depending on the environment, you may need the following to enable and configure the firewall (ufw firewall):

ufw allow 22/tcp
ufw allow 51820/udp
ufw enable
You also may need to add the following if the default firewall policy is to REJECT:

iptables -A INPUT -p udp -m udp --dport 51820 -j ACCEPT
iptables -A OUTPUT -p udp -m udp --sport 51820 -j ACCEPT
  
And finally, if you are running this on OpenVZ, you may need to (at the host level - so need to talk to your service provider):

vzctl set $CTID --netfilter full --save

Sources:
https://wiki.shulepov.com/software/linux/qrnecode_wireguard
 https://www.reddit.com/r/WireGuard/comments/dze220/wireguard_on_ubuntu_1804_openvz/
  
https://d.sb/2019/07/wireguard-on-openvz-lxc
