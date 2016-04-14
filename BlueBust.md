# BlueBust
BlueBust is a special setup that gets past BlueCoat (and other deep packet inspection) filters. It is cross platform, versatile, and very secure. Use the base install but set the client to listen to the localhost, as well as the server. Stunnel provides the wrapper, which bypasses the BlueCoat Deep Packet Inspection filters, regular OpenVPN traffic will not be able to breach to the outside of the network, which makes this extra encryption a necessity.

## Server
Install stunnel

<code>sudo apt-get install stunnel4</code>

Create self-signed key

<code>openssl req -new -nodes -x509 -days 3650 -out /etc/stunnel/stunnel.pem -keyout /etc/stunnel/stunnel.pem</code>
Next configure stunnel to listen and connect on respective ports by editing */etc/stunnel/stunnel.conf*
```
cert = /etc/stunnel/stunnel.pem
 
[openvpn]
accept = 443
connect = 1194
```
Then edit the ENABLED variable to 1 in */etc/default/stunnel4*
<code>sudo vim /etc/default/stunnel4</code>
Then restart stunnel
<code>sudo service stunnel4 restart</code>

## Client
Install stunnel also on the client [Windows Download](https://www.stunnel.org/downloads.html)
<code>sudo apt-get install stunnel4</code>
Then use this config
```
[openvpn]
client = yes
accept = localhost:1194
connect = {ip}:443
```
Then restart stunnel
