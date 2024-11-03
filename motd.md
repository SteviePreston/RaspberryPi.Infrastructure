# Message of the day Scripts:

To remove the majority of bloat from the logon apply the following permissons:

```sh
cd /etc/update-motd.d
```

```sh
ls -l
```

```sh
sudo chmod 600 <script-name>
```
- N.B *** marks the ones I have kept enabled at login.

```bash
-rwxr-xr-x 1 root root 1220 Apr 22  2024 00-header ***
-rw------- 1 root root 1151 Apr 22  2024 10-help-text
lrwxrwxrwx 1 root root   46 Aug 27 15:29 50-landscape-sysinfo -> /usr/share/landscape/landscape-sysinfo.wrapper ***
-rw------- 1 root root 5023 Apr 22  2024 50-motd-news
-rw------- 1 root root   84 Apr  5  2024 85-fwupd
-rw------- 1 root root   83 Mar  8  2024 90-pemmican
-rwx------ 1 root root  218 Feb 16  2024 90-updates-available ***
-rw------- 1 root root  296 Apr 30  2024 91-contract-ua-esm-status
-rw------- 1 root root  558 Jun 21 21:19 91-release-upgrade
-rw------- 1 root root  165 Feb 12  2024 92-unattended-upgrades
-rw------- 1 root root  379 Feb 22  2024 95-hwe-eol
-rw------- 1 root root  111 Jan 26  2024 97-overlayroot
-rwxr-xr-x 1 root root  142 Feb 16  2024 98-fsck-at-reboot ***
-rw------- 1 root root  144 Feb 16  2024 98-reboot-required
```
