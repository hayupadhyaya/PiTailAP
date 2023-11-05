# PiTailAP

## Instructions on using a Raspberry Pi - Tailscale Exit node on non tailscale devices with Wireless Hotspot
Use https://www.christopherlouvet.com/posts/apple-tv-tailscale/ for reference

### Requirements:
1. Raspberry Pi (With Ethernet and Wi-Fi)
2. SD Card

#### Step 1 - SSH to Raspberry pi

ssh pi@raspberrypi.local (default password is raspberry)

#### Step 2 - Perform an Update

sudo apt update && sudo apt upgrade -y

#### Step 3 - Verify WLAN0 exists and set correct country-code

ifconfig (to see network interfaces)

sudo raspi-config (To enter raspberry pi configuration)

go to - 5 Localization options > L4 WLAN country > Set country

#### Step 4 - Install required programs and configure AP

sudo apt install -y hostapd dnsmasq iptables

sudo nano /etc/dhcpcd.conf

Enter following information at the end of file

interface wlan0
    static ip_address=10.0.1.1/24  #Make sure this is diffrent from your local IP
    static routers=10.0.1.1        #Make sure this is same as above without "/24" 
    nohook wpa_supplicant
    metric 500

crtl + x to save Y to confirm

sudo systemctl restart dhcpcd

sudo nano /etc/hostapd/hostapd.conf

interface=wlan0

hw_mode=g
channel=6
ieee80211n=1
wmm_enabled=0
macaddr_acl=0
ignore_broadcast_ssid=0

auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP

#This is the name of the network
ssid=Tailscale

#The network passphrase
wpa_passphrase=12345678

ctrl + x to save Y to confirm

sudo sed -i 's|#DAEMON_CONF=\"\"|DAEMON_CONF=\"/etc/hostapd/hostapd.conf\"|g' /etc/default/hostapd

sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak

sudo nano /etc/dnsmasq.conf

interface=wlan0      # Use interface wlan0
server=1.1.1.1       # Use Cloudflare DNS
dhcp-range=10.0.1.100,10.0.1.110,12h # IP range and lease time #Change this range if interface ip is different

ctrl + x to save Y to confirm

sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
sudo sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/g' /etc/sysctl.conf

sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"

sudo nano /etc/rc.local

Enter the following line before exit 0

iptables-restore < /etc/iptables.ipv4.nat

sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl restart hostapd
sudo service dnsmasq restart

#### Step 5 - Install Tailscale

Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

Login to your account
sudo tailscale up

See the exit-nodes in your account
sudo tailscale status

Use IP from you exit node you wish to connect with the command below
sudo tailscale up --exit-node=YOUR-TAILSCALE-EXIT-NODE-IP --exit-node-allow-lan-access

To disconnect from the exitnode
sudo tailscale up --reset

or if you are not using --exit-node-allow-lan-access flag, use
sudo tailscale up --exit-node

sudo iptables -t nat -A POSTROUTING -o tailscale0 -j MASQUERADE

sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"

sudo reboot now

#### Additional Step (May be required) 

sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
