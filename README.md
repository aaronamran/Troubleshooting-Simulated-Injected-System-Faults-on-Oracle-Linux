# Troubleshooting Simulated Injected System Faults on Oracle Linux
This homelab project simulates system faults on Oracle Linux VM to practice troubleshooting skills. It is optimised for low-resource environments, ideal for single-laptop homelabs and virtual setups.
The idea is to run the provided script to randomise system issues that will happen to provide a black-box environment, and troubleshoot those issues in a strategical approach.


1. [Preparation and Initial Setup](#preparation-and-initial-setup)
2. [Injecting Random System Faults](#injecting-random-system-faults)
3. [Searching and Troubleshooting Issues](#searching-and-troubleshooting-issues)


## Preparation and Initial Setup
- Download Oracle Linux ISO version 9.6 from [https://yum.oracle.com/oracle-linux-isos.html](https://yum.oracle.com/oracle-linux-isos.html). Install it in VirtualBox and use 4096MB of RAM, 2 CPUs and 20GB storage
- Power on the VM. Select the suitable language and on the screen shown as below, setup the Installation Destination, Root Password and User Creation <br />
  <img width="1292" height="820" alt="image" src="https://github.com/user-attachments/assets/b8877196-e66a-40df-a456-ff53dd878eeb" />

- In Installation Destination, select the disk where the OS will be installed on. Click Done when ready <br />
  <img width="1287" height="815" alt="image" src="https://github.com/user-attachments/assets/d134aac7-5349-4233-9f89-98c933e53b3e" />

- In Root Password, set the root password and confirm it. A simple password like `sysadmin` for the sake of this homelab is acceptable. Press Done twice since the password used is not strong enough <br />
  <img width="1291" height="812" alt="image" src="https://github.com/user-attachments/assets/f8dc2ac1-c103-40d8-8e52-23d4ff7923d4" />

- In User Creation, create a user account. Simple login credentials are enough <br />
  <img width="1291" height="814" alt="image" src="https://github.com/user-attachments/assets/73921757-1d02-4d6b-8ece-419d62e18050" />

- Begin the installation and let it complete <br />
  <img width="1295" height="821" alt="image" src="https://github.com/user-attachments/assets/3eea70e5-240f-4ed4-b707-e8ad4c3e5a78" />

- 




## Injecting Random System Faults
- In the Oracle Linux VM, run the following script to randomise problems that will occur in the VM
  ```
  #!/bin/bash
  # Simulate and Troubleshoot: Black-box Fault Injector for Oracle Linux
  
  set -e
  
  echo "[*] Starting black-box fault injection..."
  
  # Output file to record what faults were injected
  fault_log="/root/.injected_faults.txt"
  echo "[+] $(date): Fault injection run" > "$fault_log"
  
  # === Fault Definitions ===
  
  inject_dns_failure() {
    echo "nameserver 127.0.0.1" > /etc/resolv.conf
    echo "- DNS failure (resolv.conf overwritten)" >> "$fault_log"
  }
  
  inject_systemd_failure() {
    cat <<EOF > /etc/systemd/system/badunit.service
  [Unit]
  Description=Broken Service
  After=network.target
  
  [Service]
  ExecStart=/nonexistent/script.sh
  Restart=always
  
  [Install]
  WantedBy=multi-user.target
  EOF
    systemctl daemon-reload
    systemctl enable badunit.service
    systemctl start badunit.service || true
    echo "- Systemd failure (badunit.service broken)" >> "$fault_log"
  }
  
  inject_fstab_error() {
    echo "/dev/fakevolume /mnt/fake ext4 defaults 0 0" >> /etc/fstab
    echo "- Fstab error (invalid mount added)" >> "$fault_log"
  }
  
  inject_sudo_broken() {
    echo "badentry" >> /etc/sudoers
    echo "- Sudo failure (invalid sudoers line)" >> "$fault_log"
  }
  
  inject_cron_failure() {
    echo "* * * * * root /bin/bash -c 'echo MissingQuote" > /etc/cron.d/brokenjob
    echo "- Cron job failure (syntax error)" >> "$fault_log"
  }
  
  inject_full_disk() {
    fallocate -l 500M /var/bloatfile
    echo "- Disk fill (500MB junk file created)" >> "$fault_log"
  }
  
  inject_ssh_config_error() {
    echo "PermitRootLogin maybe" >> /etc/ssh/sshd_config
    systemctl restart sshd || true
    echo "- SSH config error (invalid directive)" >> "$fault_log"
  }
  
  inject_selinux_error() {
    if sestatus | grep -q "enabled"; then
      chcon -t httpd_sys_content_t /etc/shadow
      echo "- SELinux mislabel (wrong context on /etc/shadow)" >> "$fault_log"
    fi
  }
  
  # === Fault List ===
  
  problems=(
    inject_dns_failure
    inject_systemd_failure
    inject_fstab_error
    inject_sudo_broken
    inject_cron_failure
    inject_full_disk
    inject_ssh_config_error
    inject_selinux_error
  )
  
  # Shuffle & inject 3–6 random faults
  num_faults=$(( RANDOM % 4 + 3 ))
  shuffled_faults=($(shuf -e "${problems[@]}"))
  selected_faults=("${shuffled_faults[@]:0:$num_faults}")
  
  for fault in "${selected_faults[@]}"; do
    $fault
  done
  
  echo "[+] Fault injection complete. ($num_faults issues injected)"
  echo "[*] You can view the fault log after you're done: $fault_log"
  ```
  <br />
  <img width="632" height="640" alt="image" src="https://github.com/user-attachments/assets/bcc2e5d4-b2ed-4ead-b887-09ee1106c77c" />


- The file can be read and is located in
  ```
  cat /root/.injected_faults.txt
  ```

- 


## Searching and Troubleshooting Issues
- Step	Goal	Example Tools
1. Basic Health Check	Is the system alive, usable, responsive?	uptime, top, free -m, df -h
2. Recent Logs	Any obvious issues reported?	journalctl -xe, dmesg, /var/log/messages
3. System Services	Any failed services?	systemctl --failed, systemctl list-units --state=failed
4. Disk & Mounts	Disk full? Mount broken?	df -h, mount -a, /etc/fstab
5. Network Connectivity	DNS? Gateway? SSH?	ping, dig, ip a, ip r, ss -tuln
6. User Access	Can users sudo? SSH? Login?	id, sudo -l, getent passwd, last
7. Scheduled Jobs	Are cron jobs breaking things?	crontab -l, /etc/cron*, log checks
8. SELinux/Permissions	Is access denied unexpectedly?	getenforce, audit2why, ls -Z
9. Application Layer	Is anything expected missing/broken?	App-specific logs, configs
10. Investigate Deeper	Now dig into suspicious leads	grep logs, check service configs, try restarts, etc






## 
