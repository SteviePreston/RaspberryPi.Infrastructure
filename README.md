# Raspberry Pi Home Cluster

This is an experimental home project, Trying to build infrasturture for home project, Down the line I plan to add 2 more 8gb Raspberry pi-5 and create a small cluster with persistant storage.
I will add more if nessary dependant on the scale of my personal project. The techstack I plan to use is: Anisble, Docker, K3's, GitLab Actions, Ubuntu Server LTS, Grafana, nginx, PostgresDB Prometheus and Harbor.
Will add more if scaling issues occur. Will document Hardware if needed as the next stage will be a switch and two raspberry pi's to add as worker nodes.

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

## UFW (Uncomplicated Fire Wall)

- Check the status of ufw

```sh
sudo ufw status
```

- Allow SSH, HTTP & HTTPS:

```sh
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
```

- Enable ufw

```sh
sudo ufw enable
```

- Recheck the status if ufw:

```sh
sudo ufw status verbose
```

- Reload ufw

```sh
sudo ufw reload
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
      run: echo "${{ env.DOCKER_PASSWORD }}" | docker login -u "${{ env.DOCKER_USERNAME }}" --password-stdin

    - name: Build Docker Image
      run: |
        docker build -t ${{ env.DOCKER_USERNAME }}/${{ env.DOCKER_PROJECT }}:latest .

    - name: Push Docker Image
      run: |
        docker push ${{ env.DOCKER_USERNAME }}/${{ env.DOCKER_PROJECT }}:latest

  rpi_deploy:
    runs-on: ubuntu-latest
    needs: docker_build_and_push

    steps:
    - name: Set up SSH
      uses: webfactory/ssh-agent@v0.9.0
      with:
        ssh-private-key: ${{ env.PI_K3_SSH_PRIVATE_KEY }}

    - name: Pull and Run Docker Container on Raspberry Pi
      run: |
        ssh -o StrictHostKeyChecking=no ${{ env.SSH_USERNAME }}@${{ env.SSH_IP }} "
          docker stop ${{ env.DOCKER_PROJECT }} || true && \
          docker rm ${{ env.DOCKER_PROJECT }} || true && \
          docker pull ${{ env.DOCKER_USERNAME }}/${{ env.DOCKER_PROJECT }}:latest
          "

```
## Pi-Hole

#### 1. Install pihole:

```sh
curl -sSL https://install.pi-hole.net | bash
```
#### 2. Wizard Questions:

```
Question 1: OK
Question 2: OK
Question 3: Continue
Question 4: eth0 -> Select
Question 5: Yes -> Continue
Question 6: OK
Question 7: Google DNS -> Yes
Question 8: Yes
Question 9: Yes
Question 10: Yes
Question 11: 0 -> Continue
Question 12 Take down the information -> OK
```

#### 3. Change Password:

```sh
sudo pihole -a -p
```
#### 4. Enter and Confirm Password

#### 5. Change Pi-Hole Port:

```sh
sudo vim /etc/lighttpd/lighttpd.conf
```
```sh
server.port = 8080
```

#### 6. Add Pi-Hole Custom Port to ufw:

```sh
sudo ufw allow 8080
```

#### 7. Restart Lighttpd:

```sh
sudo systemctl restart lighttpd
```

#### 8. Access Pi-Hole:

```sh
http://<Raspberry-Pi_IP>:8080/admin/index.php
```

#### 9. Enable IPv6 DNS:

- Settings -> DNS -> Tick the IPv6 Google DNS -> Save

## NginX

#### 1. Install NginX:

```sh
sudo apt install nginx
```

#### 2. Start and Enable Nginx:

```sh
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### 3. Check Nginx Status:

```sh
sudo systemctl status nginx
```

#### 4. Create Subdomains on DNS Provider:
- Create A class and CNAMES on DNS Provider

```
A Class = {
Type : A
Name : <subdomain>
Data : <raspberry-pi-ip>
TTL  : 600
}
```
```
CNAME Class = {
Type : CNAME
Name : www.<subdomain>
Data : <subdomain>.<raspberry-pi-ip>
TTL  : 600
}
```

#### 5. Add Host Config:

```sh
sudo vim /etc/hosts
```

- Add IP and Host name to the `/etc/hosts` file.

```sh
<raspberry-pi-ip> domain.name.com
```

#### 6. Set Up an Nginx Reverse Proxy for Pi-hole

- Create a configuration file:
  
```sh
vim /etc/nginx/sites-available/<subdomain>.<domain.com>
```
- Configure the Reverse Proxy:
```
server {
    listen 80;
    server_name subdomain.domain.com www.subdomain.domain.com;

    location / {
        proxy_pass http://<raspberry-pi-ip>:<application_port>;  
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Optional: Set up basic authentication if you want to restrict access
    # auth_basic "Restricted Access";
    # auth_basic_user_file /etc/nginx/.pihole_htpasswd;
}
```

#### 7. Link the configuration file to Nginx’s sites-enabled directory:

```sh
sudo ln -s /etc/nginx/sites-available/<subdomain>.<domain.com> /etc/nginx/sites-enabled/
```

#### 8. Test and Reload Nginx:

```sh
sudo nginx -t
sudo systemctl reload nginx
```

#### 9. Access the site

```
http://www.<subdomain>.<domain>.com
```

## k3s

#### 1. Install k3s:
- On the Master Node (Primary Pi), run:

```sh
curl -sfL https://get.k3s.io | sh -
```

#### 2. Retrieve the token, which you’ll need to connect worker nodes:

```sh
sudo cat /var/lib/rancher/k3s/server/node-token
```

#### 3. Verify k3s on the Master Node:

```sh
sudo kubectl get nodes
```

#### 4. Add Worker Nodes to the Master Node:

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<master-ip>:6443 K3S_TOKEN=<node-token> sh -
```

#### 5. Verify k3s on the worker Node:

```sh
sudo kubectl get nodes
```

#### 6. Deploy a Docker Container to the k3s Cluster:
-Create a Kubernetes Deployment YAML: `.deployment/deployment.yml` for deploying your Docker container to the cluster:

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: your-docker-username/your-image-name:tag
        ports:
        - containerPort: 80
```

- Apply the Deployment:

```sh
kubectl apply -f .deployment/deployment.yml
```

#### 7. Expose the Deployment within Your Local Network :
- To access the deployed container within your Pi network, expose it using `.deployment/service.yml` file for a NodePort service:

```yml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 123
      nodePort: 30000  # Optional, any available port in the 30000-32767 range
```
This way, external requests to your Raspberry Pi on port 30000 (or whichever nodePort you choose) will be correctly routed to port 123 in your app container.


- Apply the Service:

```sh
kubectl apply -f .deployment/service.yml
```

#### 8. Edit k3s Permissions :

- Start k3s with --write-kubeconfig-mode:
- Edit k3s Service and Add the `--write-kubeconfig-mode` Flag

```sh
sudo vim /etc/systemd/system/k3s.service
ExecStart=/usr/local/bin/k3s server --write-kubeconfig-mode 644
```
- Reload and Restart k3s:

```sh
sudo systemctl daemon-reload
sudo systemctl restart k3s
```
- Verfify Changes:

```sh
kubectl get nodes
```

#### 9. Update CICD to include k3s config:

```yml
**** WILL UPDATE ONCE FIXED ****
```

## ============================================================
## TODO: 
### - k3s is a WiP
### - USE A DEPLOYMENT KEY
### - FIX STRICT HOST KEY CHECKING
### - INSTALL ANSIBLE AND GRAFANA
