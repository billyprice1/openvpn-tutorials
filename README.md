# OpenVPN
These files are tutorials for various OpenVPN setups, the main README being a regular tunneling setup using a Debian 8 server. These setups were pieced together from many sources but mainly the [ArchWiki](https://wiki.archlinux.org/index.php/OpenVPN) and [Debian Wiki](https://wiki.debian.org/OpenVPN) tutorials.

## Setting up the server
To install on debian simply do <code>sudo apt-get install openvpn</code>To install OpenVPN on the server (there is no specific server package, OpenVPN is either configured to run as a client or server). If you want to get a basic generic start copy over all the files from */usr/share/doc/openvpn/examples/sample-config-files* to */etc/openvpn*. ***Do not run this as-is.*** First of all it may not work and second of all if you have edited the files configured to the correct IP's it will connect to itself since it has both the client and server files within the config folder. **Never have both client and server configs within the same folder!** A more specific config file for both are provided below.

Now for Debian 8 you also need to install easy-rsa separately <code>sudo apt-get install easy-rsa</code>Then instead of copying over from the sample configuration file where easy-rsa was before copy from */usr/share/easy-rsa*. <code>cp -r /usr/share/easy-rsa/ /etc/openvpn</code> From there on continue as normal.

### Sample server config
This file should be called something like server.conf within the */etc/openvpn* folder.
```
port 1194 # port which it runs over
proto udp # specifies which IP protocol to use
dev tun # specifies this as a tunneling server

ca         /etc/openvpn/easy-rsa/keys/ca.crt    # generated keys
cert       /etc/openvpn/easy-rsa/keys/server.crt
key        /etc/openvpn/easy-rsa/keys/server.key  # keep secret
dh         /etc/openvpn/easy-rsa/keys/dh2048.pem
crl-verify /etc/openvpn/easy-rsa/keys/crl.pem  # prevents revoked certificates from connecting

server 10.5.0.0 255.255.255.0 # specifies the internal IP scheme
ifconfig 10.5.0.1 10.5.0.2

ifconfig-pool-persist ipp.txt # tries to make internal IPs persistent
keepalive 10 120 # sends pings to client to keep connection open on NAT firewalls

comp-lzo # Compression - must be turned on at both ends
persist-key # make sure the key persists if permissions are downgraded
persist-tun # same thing with tunnel
push "redirect-gateway def1 bypass-dhcp" # redirect all packets except for local gateway
push "dhcp-option DNS 10.5.0.1" # push the VPN server's choice for DNS

status status.log

verb 3  # verbose mode
#client-to-client # allow a hamachi like internal network
```

#### Create a Hamachi-like network

Simply take the above configuration but remove the "push" lines and then it will not force all traffic though the VPN. Make sure to have the client-to-client line that is at the bottom as that is what provides the ability for all devices to communicate using the internal 10.x.x.x addresses. [Original Source](https://forums.openvpn.net/topic11344.html)

### Creating keys
OpenVPN has a series of scripts called easy-rsa which helps manage the creation of keys for the VPN. To use this scripts copy them over from */usr/share/doc/openvpn/examples/easy-rsa/2.0* to */etc/openvpn*. For the exact commands on how to do this:

**Creating the easy-rsa folder** 
<code>sudo mkdir /etc/openvpn/easy-rsa</code>

**Copying over the contents from the documentation** 
<code>sudo cp -r /usr/share/doc/openvpn/examples/easy-rsa/2.0/* /etc/openvpn/easy-rsa/</code>

Then going into root using the command <code>sudo su</code> is probably the least painful way to do the next part. This part we will go through the process of creating the keys for the CA, server, and clients so they can be distributed to where ever they need to be. Points to remember:
  * Each client device should have it's own key
  * .key files should be kept secret
  * .csr files don't really matter, don't move them
  * .crt files can be submitted over plaintext
**First time setup commands**
  - edit the vars file to configure keys (set key size to 2048)
  - source vars
  - ./build-ca
  - ./build-key-server server
  - ./build-dh
**For each client**

*This name should be descriptive of the device so that later it makes it easier for revocation in case it gets stolen or lost.*

<code>./build-key clientName</code>

## Setting up routing
This part is CRUCIAL to the ability of the tunnel to be able to access the internet. On the other hand it could be useful to omit this in the event you are trying to make a network which is for purely p2p VPN such as a Hamachi replacement. To enable IP forwarding on the current running server: 

<code>echo 1 > /proc/sys/net/ipv4/ip_forward</code> 

And to make sure the ip forwarding stays persist uncomment the folling line in */etc/sysctl.conf*

<code>net.ipv4.ip_forward = 1</code> 

Then issue the following commands so that iptables will route the traffic through the internet
```
sudo iptables -A FORWARD -i eth0 -o tun0 -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A FORWARD -s 10.5.0.0/24 -o eth0 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 10.5.0.0/24 -o eth0 -j MASQUERADE
```
These rules will not be saved on their own. Only after issuing all those commands successfully then run this command to install a program which makes all the current rules persistent. <code>sudo apt-get install iptables-persistent</code>


## Setting up the client
Install OpenVPN for the client (or OpenVPN Connect for mobile devices) and then insert the sample configuration file making sure that the IP, port, protocol, and keys are all correctly specified within the config file. In Windows and Android the file must have the extension .ovpn in order to be accepted by the program. **NOTE: OpenVPN must be run as sudo or administrator in order to redirect all traffic through the VPN.** Otherwise the only traffic that gets routed through is if anything is configured to work with the OpenVPN SOCKS proxy which is not recommended.

### Sample client config
All places marked with a double pound needs some value placed there, make sure the file has all configurations set correctly anyways.
```
## place description of device here for identification
client
port 1194 # what port to use
dev tun # makes it a VPN tunnel and not bridge
proto udp # what IP protocol to use (UDP is recommended for VPN)
remote-cert-tls server # verifies server cert to prevent MiTM, extremely important

remote SERVER_IP 1194 # specifies the server IP and port to connect to
nobind

persist-key
persist-tun
comp-lzo # makes sure compression is on

verb 3
mute 20
# certificate authority cert
<ca>
## place CA cert here ##
</ca>
# client certificate
<cert>
## place client cert here ##
</cert>
# client key
<key>
## place client key here ##
</key>
```
