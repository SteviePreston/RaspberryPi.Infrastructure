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

## Docker

#### 1. Remove conflicting packages:

```sh
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```


#### 2. Add Docker's official GPG key:

```sh
sudo apt-get update
```
```sh
sudo apt-get install ca-certificates curl
```
```sh
sudo install -m 0755 -d /etc/apt/keyrings
```
```sh
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```
```sh
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository to Apt sources:

```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```sh
sudo apt-get update
```

#### 3. Install the LTS of Docker CLI:

```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### 4. Add Docker Group to SUDO Permissions:

Create the docker group:

```sh
sudo groupadd docker
```

Add your user to the docker group:

```sh
sudo usermod -aG docker $USER
```

Activate the changes to group memberships:

```sh
 newgrp docker
```

#### 5. Check Docker can run without `sudo`:

```sh
docker --version
```

```sh
docker ps
```

## GitHub Actions

#### 1. Setup Port forwarding for SSH:
- Login into your router
- In WAN settings enable port forwarding for SSH
- Enter the following:
  
  ```
  "Service Name": "GitHub Actions"
  "Protocol": "TCP"
  "External Port": 22
  "Internal Port": 22
  "Internal IP": <Your Pi Private IP>
  ```
  
#### 2. Create Encryption keys:
- Generate Private and Public keys on your local machine:
```sh
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github-actions
```
```sh
ssh-copy-id -i ~/.ssh/github-actions.pub <UserName>@<Your-Pi's-IP>
```

#### 3. Add GitHub Secrets:
- Add the following Secrets (<repo> -> Settings -> Security -> Secrets and Variables -> Actions):
- SSH-Private-Key = cat .ssh/github-actions
  
```
  "DOCKER_USERNAME": <UserName>
  "DOCKER_PASSWORD": <PassWord>
  "DOCKER_PROJECT": <Docker-Hub-Project-Name>
  "PI_K3_SSH_PRIVATE_KEY": <SSH-Private-Key>
  "SSH_IP": <Your Public IP>
  "SSH_USERNAME": <Your Pi's Username>
```

#### 4. Test SSH Connection:
- Make the following:

```sh
mkdir .github/workflows
touch deploy.yml
```
- In `deploy.yml` Add the following:

```yml
name: Build, Push, and Deploy Docker Image

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
  DOCKER_PROJECT: ${{ secrets.DOCKER_PROJECT }}
  PI_K3_SSH_PRIVATE_KEY: ${{ secrets.PI_K3_SSH_PRIVATE_KEY }}
  SSH_USERNAME: ${{ secrets.SSH_USERNAME }}
  SSH_IP: ${{ secrets.SSH_IP }}

jobs:
  docker_build_and_push:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v4

    - name: Log in to Docker Hub
      run: echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

    - name: Build Docker Image
      run: |
        docker build -t $DOCKER_USERNAME/$DOCKER_PROJECT:latest .

    - name: Push Docker Image
      run: |
        docker push $DOCKER_USERNAME/$DOCKER_PROJECT:latest

  rpi_deploy:
    runs-on: ubuntu-latest
    needs: docker_build_and_push

    steps:
    - name: Set up SSH
      uses: webfactory/ssh-agent@v0.9.0
      with:
        ssh-private-key: $PI_K3_SSH_PRIVATE_KEY

    - name: Pull and Run Docker Container on Raspberry Pi
      run: |
        ssh -o StrictHostKeyChecking=no "$SSH_USERNAME"@"$SSH_IP" "
          docker stop $DOCKER_PROJECT || true && \
          docker rm $DOCKER_PROJECT || true && \
          docker pull $DOCKER_USERNAME/$DOCKER_PROJECT:latest && \
          docker run -d -p 4200:4200 --name $DOCKER_PROJECT $DOCKER_USERNAME/$DOCKER_PROJECT:latest
        "
```

## TODO: USE A DEPLOYMENT KEY AND TEST SSH CONNECTION
