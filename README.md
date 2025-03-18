
# Documentation: Automatically Connecting to a Wi-Fi Network and Configuring a Network Bridge on Proxmox

## Introduction
This documentation covers the steps to configure a Proxmox server to automatically connect to a Wi-Fi network at startup and to configure a network bridge `( vmbr0)` in the same network as the Wi-Fi interface `( wlp3s0)`. This guide includes creating a script to automate the necessary commands and integrating this script into the system startup process using two methods: `/etc/rc.local` and `systemd`.

## 1. Prerequisites
Before you begin, make sure that:
- Your Proxmox server is equipped with a compatible Wi-Fi card.
- You have access to your Wi-Fi network (SSID and password).
- The necessary tools, like `wpa_supplicant` and `dhclient`, are installed on your system.
```bash
apt update
apt install wpasupplicant
```
- From now on, Ethernet is no longer necessary!
## 2. Manual Wi-Fi connection

### 2.1. Configure wpa_supplicant
`wpa_supplicantis` used to manage Wi-Fi connections in Linux. You need to create a configuration file for 
wpa_supplicantit that contains the details of your Wi-Fi network.


1. Create the configuration file:
    ```bash
    sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
    ```
2. Add network configuration:
    ```bash
    network={
        ssid="votre_ssid"
        psk="votre_mot_de_passe"
    }
    ```
3. Save and close the file with `Ctrl+O` to save `Ctrl+X`.

### 2.2. Connecting to the Wi-Fi network
To manually connect your server to the Wi-Fi network, use the following commands:

1. To start up `wpa_supplicant` :
    ```bash
    sudo wpa_supplicant -B -i wlp3s0 -c /etc/wpa_supplicant/wpa_supplicant.conf
    ```
    > Note: Replace <name> `wlp3s0` with the name of your Wi-Fi interface, which you can get with the command `ip a`.

2. Get an IP address:
    ```bash
    sudo dhclient wlp3s0
    ```

## 3. Automation at startup
To have the server automatically connect to Wi-Fi on startup, we will create a shell script that will run the above commands, and then configure that script to run automatically.

### 3.1. Creating the shell script

1. Create the script:
    ```bash
    sudo nano /usr/local/bin/connect_wifi.sh
    ```
2. Add the commands to the script:
    ```bash
    #!/bin/bash
    sudo wpa_supplicant -B -i wlp3s0 -c /etc/wpa_supplicant/wpa_supplicant.conf
    sudo dhclient wlp3s0
    ```
3. Make the script executable:
    ```bash
    sudo chmod +x /usr/local/bin/connect_wifi.sh
    ```

### 3.2. Integration of the script at startup
There are two main ways to run this script automatically when your server starts: using `/etc/rc.local` or creating a service `systemd`.

#### Option 1: Use `/etc/rc.local`

1. To modify `/etc/rc.local` :
    - If the File `/etc/rc.local` does not exist, create it:
    ```bash
    sudo nano /etc/rc.local
    ```
2. Add the script to the file:
    Add the following line before `exit 0` :
    ```bash
    /usr/local/bin/connect_wifi.sh
    ```
3. Make `/etc/rc.local` executable (if not already done):
    ```bash
    sudo chmod +x /etc/rc.local
    ```

#### Option 2 : Create a service `systemd`

1. Create a service file `systemd` :
    ```bash
    sudo nano /etc/systemd/system/connect_wifi.service
    ```
2. Add the following configuration to the file:
    ```bash
    [Unit]
    Description=Connect to Wi-Fi at startup
    After=network.target

    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/connect_wifi.sh
    RemainAfterExit=true

    [Install]
    WantedBy=multi-user.target
    ```
3. Enable the service:
    To have the service run at startup:
    ```bash
    sudo systemctl enable connect_wifi.service
    ```
4. Test the service:
    To start the service manually and verify its operation
    ```bash
    sudo systemctl start connect_wifi.service
    ```

## 4. Network Bridge Configuration `( vmbr0)` with Wi-Fi Interface (currently under testing)
Proxmox uses `vmbr0` to connect virtual machines to the network. If you want it to `vmbr0` be in the same network as `wlp3s0`(your Wi-Fi interface), you must configure the network bridge to use the Wi-Fi interface.
### 4.1. Configure `/etc/network/interfaces`

1. To Modify `/etc/network/interfaces` :
    ```bash
    sudo nano /etc/network/interfaces
    ```
2. Add or modify the following configuration:
    ```bash
    auto wlp3s0
    iface wlp3s0 inet manual

    auto vmbr0
    iface vmbr0 inet dhcp
        bridge_ports wlp3s0
        bridge_stp off
        bridge_fd 0
    ```
    > Note: If the Wi-Fi card does not support bridge mode, this configuration may not work. In this case, you will need to use another method, such as configuring NAT with `iptables`.
    
3. Restart network interfaces:
    ```bash
    sudo systemctl restart networking
    ```

## 5. Summary
You have configured your Proxmox server to automatically connect to a Wi-Fi network at startup using a shell script, and you have integrated this script into the startup process using either `/etc/rc.local`, or a service `systemd`. Additionally, you have configured the network bridge `vmbr0` to operate in the same network as `wlp3s0`.

## 6. Troubleshooting

- **Check service status** : Si vous utilisez `systemd`, vous pouvez vérifier le statut du service pour diagnostiquer des problèmes :
    ```bash
    sudo systemctl status connect_wifi.service
    ```
- **Logs of `wpa_supplicant`** : Errors related to `wpa_supplicant` can be examined int the system logs:
    ```bash
    sudo journalctl -u wpa_supplicant
    ```
- **Manually restart services** : If something isn't working as expected, manually restart the services or the server to see if the problem persists.

---

This documentation should help you configure and automate Wi-Fi on your Proxmox server, as well as configure a network bridge for your virtual machines. If you encounter any additional difficulties, please feel free to revisit these guidelines or ask for additional assistance.
