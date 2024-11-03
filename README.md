# Setup

## Raspberry Pi

#### 1. Flash SD Cards with Raspberry Pi OS Using Pi Imager
- Open [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
  - Choose the 'OS' you want to install from the list. I used the LTS of Ubuntu Server
  - Insert your SD card/M.2 nvme and select it in the 'Storage' section.
  - Before writing, click on the cog icon for advanced settings.
    - Set the hostname to your desired value, e.g., `RP1`.
    - Enable SSH and select the "allow public-key authorization only" option.
  - Click on 'Write' to begin the flashing process.
  
#### 2. Initial Boot and Setup
- Insert the flashed SD card/M.2 nvme into the Raspberry Pi and power it on.
- On the first boot, ssh into the Pi to perform initial configuration
  
#### 3. Update and Upgrade
- Run the following commands to update the package list and upgrade the installed packages:

```bash
sudo apt update && sudo apt upgrade
```

#### 4. Disable Wi-Fi

```sh
sudo vim /etc/wpa_supplicant/wpa_supplicant.conf
```

Add the following lines to the file:

```sh
network={
    ssid=""
    key_mgmt=NONE
}
```

Disable the Wi-Fi interface:

```sh
sudo apt install net-tools
```
```sh
sudo ifconfig wlan0 down
```

Block the Wi-Fi module using `rfkill`:

```sh
sudo apt install rfkill
```
```sh
sudo rfkill block wifi
```

Prevent the Wi-Fi module from loading at boot:

```sh
sudo vim /etc/modprobe.d/raspi-blacklist.conf
```

Add the following line:

```sh
blacklist brcmfmac
```

Reboot your Raspberry Pi:

```sh
sudo reboot
```