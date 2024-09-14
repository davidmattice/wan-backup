
**NOTE: This file has been relocated to [this repo](https://github.com/davidmattice/Usefull_Documentation)**



# Building a "Bridge" using a Raspberry Pi (for WAN backup)

These steps can be used to setup a Raspberry Pi to act as a bridge between a Mobile Wireless Hotspot (or USB Tether) and an Ethernet connected device.  This can be a single device or a Router Wan connection.


It will be necessary to connect either wirelessly or using a wired ethernet connection to complete the configuration of the **Bridge**.


## The environment for this use case is:
* Samsung S21 on T-Mobile
* Raspberry Pi 3+ - Raspberry Pi OS 11 (bullseye) - Lite edition (no desktop needed)
* Ubiquiti USG-3P (4.4.56) AND Asus RT-AX3000 (3.0.0.4)

## Steps to configure the Pi:

1.  Burn a new SD card using the [Raspberry Pi Imager](https://www.raspberrypi.com/software/).  Use the optional settings to configure the **hostname**, the **pi** users password and SSH access.   Optionally configure the initial Wireless settings for the local Wireless network.

2.  Boot the Raspberry Pi with the SD card.

3.  Connect via SSH using the **pi** user and passwrod set above.  The connection can be over WiFi (if configured) or via a connected Ethernet cable (DHCP is required).

4.  Update the OS
    ```
    sudo apt update
    sudo apt upgrade -y
    sudo reboot
    ```
    A reboot is required because of the Kernel updates.  It could be skipped if there are no Kernel updates.

5.  Add required software
    ```
    sudo apt-get install -y dnsmasq iptables
    ```

6.  Update Wireless confiiguration to your Hotspot Wireless SSID and Password.  **Warning: Do not reboot from this point until all steps are completed!**

    Update the file: /etc/wpa_supplicant/wpa_supplicant.conf
    ```
    network={
        ssid="networkname"
        psk="networkpassword"
    }
    ```
    The SSID and Password should be available from your wireless device, Hotspot settings

7.  Update the wired Ethernet inferface to have static IP and Routers entries

    Update the file: /etc/dhcpcd.conf
    ```
    interface eth0
    static ip_address=192.168.220.1/24  # Alternate address option: 172.31.0.1/24
    static routers=192.168.220.0        # Alternate address option: 172.31.0.0
    ```
    Search for **eth0** as there should already be a commented out section.  This should be a subnet that is not in use on your local netowrk or by the Wireless Provider for the Hotspot.  Setting this to a static address will allow you to SSH to the Raspberry Pi from the connected device to make any required updates.

8.  Update the priority of the Wireless interface to be preferred over the wired connection

    Update the file: /etc/dhcpcd.conf
    ```
    # Set a low metric for wlan0 to make it the primary interface
    interface wlan0
    metric 100
    ```
    This can be added near the top of the file.  It should be the only uncommented reference to wlan0

9.  Move the default dnsmasq configuration to a backup, as only a few specific lines are required
    ```
    sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    ```

10. Create a new dnsmasq configuration file with just the specific lines needed

    Create the file: /etc/dnsmasq.conf
    ```
    sudo cat >/etc/dnsmasq.conf <<EOF
    interface=eth0                 # Use interface eth0  
    listen-address=192.168.220.1   # Specify the address to listen on - Alternate address option: 172.31.0.1 (same as eth0 IP above)
    bind-dynamic                   # Bind to the interface
    server=8.8.8.8                 # Use Google DNS or alterativly 1.1.1.1
    domain-needed                  # Don't forward short names  
    bogus-priv                     # Drop the non-routed address spaces.  
    dhcp-range=192.168.220.50,192.168.220.150,12h # IP range and lease time  - Alternate: 172.31.0.50,172.31.0.150,12h
    EOF

    ```

11. Update the firewall to allow forwarding of IPv4 traffic
    ```
    sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
    ```

12. Update the filewall rules to forward all traffic from **eth0** to **wlan0**
    ```
    sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
    sudo iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT  
    sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
    ```

13. Save the firewall rules so they can be reloaded on a reboot, otherwise they will be lost
    ```
    sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
    sudo cat /etc/iptables.ipv4.nat
    ```
    You can review the rules to see they include the ones created above.

14. Configure the rules to be reloaded on a reboot

    Update the file: /etc/rc.local
    ```
    iptables-restore < /etc/iptables.ipv4.nat
    ```
    This should be added just before the **exit 0** statement

15. Shutdown and make final connections or reboot (if already connected)
    ```
    sudo shutdown -h now
    sudo reboot
    ```

16. Once powered up you should be able to connect to the static IP on **eth0** from a lan connected device

    The **wlan0** interface should get an IP from the hotspot device when enabled

### Configuring a Unifi USG to connect on the Failover WAN2 port

1.  Setup a network on the WAN2 port
2.  Add a route to the static routes to get to the Raspberry Pi using SSH
