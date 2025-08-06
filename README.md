# Troubleshooting Simulated Injected System Faults on Oracle Linux
This homelab project simulates system faults on Oracle Linux VM to practice troubleshooting skills. It is optimised for low-resource environments, ideal for single-laptop homelabs and virtual setups.
The idea is to run the provided script to randomise system issues that will happen to provide a black-box environment, and troubleshoot those issues in a strategical approach.


1. [Preparation and Initial Setup](#preparation-and-initial-setup)
2. [Injecting Random System Faults](#injecting-random-system-faults)
3. [Searching and Troubleshooting Issues](#searching-and-troubleshooting-issues)
4. [Confirming Issues Found](#confirming-issues-found)


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

- Ensure that the Bash script is executable. Use the following command to change the permissions
  ```
  sudo chmod 777 faultinjector.sh
  ```
  <img width="1137" height="674" alt="VirtualBox_Oracle Linux_06_08_2025_11_00_18" src="https://github.com/user-attachments/assets/f4d8b239-2053-4a68-9b5a-a535ccfede6e" />


- After running the script, the file can be read and is located in
  ```
  cat /root/.injected_faults.txt
  ```
  <img width="662" height="212" alt="image" src="https://github.com/user-attachments/assets/b157d89f-4d4b-46ca-a218-eca1023be8b2" />


## Searching and Troubleshooting Issues
- Since the 3 faults injected were random, a step-by-step approach to search for the issues will be implemented
- Firstly, a general system overview will be checked. Each of the following commands are used and their results provided
  ```
  uptime
  ```
  <img width="531" height="67" alt="image" src="https://github.com/user-attachments/assets/ed4fcc00-0dad-4d74-9c12-998f3765c4ea" />

  ```
  top
  ```
  <img width="903" height="868" alt="image" src="https://github.com/user-attachments/assets/e22addc2-939f-4341-99b6-0297550aef2c" />


  ```
  df -h
  ```
  <img width="647" height="183" alt="image" src="https://github.com/user-attachments/assets/d55705db-fdb3-4f1a-b063-908b1e707d5a" />


  ```
  free -m
  ```
  <img width="670" height="98" alt="image" src="https://github.com/user-attachments/assets/2bd6b4be-5e52-4207-bc0a-e1786c66b3f0" />


- Next, look for failed services. Use the command
  ```
  systemctl --failed
  ```
  <img width="605" height="165" alt="image" src="https://github.com/user-attachments/assets/f48ee120-9759-4b72-94c0-7671df3e70c1" />


- If something shows up like `badunit.service`, get the details like the following
  ```
  systemctl status badunit.service
  journalctl -xe -u badunit.service
  ```
  <img width="944" height="421" alt="image" src="https://github.com/user-attachments/assets/32299d07-307e-4946-a085-a65d96a1e811" /> <br />
  <img width="503" height="939" alt="image" src="https://github.com/user-attachments/assets/6e8faa7c-9dd7-4b4c-af1f-dad1e70267c9" />

- Check also for disk or filesystem issues. Use
  ```
  mount -a
  ```
  <img width="554" height="105" alt="image" src="https://github.com/user-attachments/assets/ba227e41-938a-41ba-97ab-339346c656ba" /> <br />
  From here we can see a fake mount point that does not exist


- Check for bad mounts. If `/mnt/fake` or `/dev/fakevolume` appears, it is a deliberate fault
  ```
  cat /etc/fstab
  lsblk
  ```
  <img width="776" height="443" alt="image" src="https://github.com/user-attachments/assets/6de5de39-2f78-4116-bd7f-8d83eee0f9bc" />


- Sometimes the issue could be sudo or root access problems. Use the following command to check
  ```
  sudo ls /root
  ```
  <img width="307" height="74" alt="image" src="https://github.com/user-attachments/assets/65d7cf5e-577e-43ca-a96e-3605587457f0" />


- Check for DNS and network issues. A simple ping to google.com and to Google's DNS server should suffice
  ```
  ping -c 4 google.com
  ping -c 4 8.8.8.8
  ```
  <img width="522" height="235" alt="image" src="https://github.com/user-attachments/assets/c7780cb5-55c2-4132-93fa-8fefb153f519" />

  
- If 8.8.8.8 works but google.com does not work, DNS is broken. Open the `resolv.conf` file and check if the nameserver is faulty
  ```
  cat /etc/resolv.conf
  ```
  <img width="384" height="65" alt="image" src="https://github.com/user-attachments/assets/39c16981-a779-483f-8ae6-6eaafc1d2cf5" /> <br />
  The nameserver is faulty as it is trying to resolve DNS through itself (localhost). The nameserver should be 8.8.8.8


- Check the SSH service. This is an important step if you would normally SSH into the box
  ```
  ss -tulnp | grep sshd
  systemctl status sshd
  ```
  <img width="704" height="335" alt="image" src="https://github.com/user-attachments/assets/a35bb9bb-f0fe-4dfd-9ca8-227f38808fe1" /> <br />

  If `PermitRootLogin maybe` is visible, it means it is invalid

- Cron jobs may be problematic too. Check the broken cron syntax using
  ```
  ls /etc/cron.d/
  cat /etc/cron.d/brokenjob
  ```
  <img width="422" height="106" alt="image" src="https://github.com/user-attachments/assets/458c21d9-e24f-45ca-9843-be2e0db11dd3" />


  Check the cron logs too
  ```
  grep CRON /var/log/cron
  ```
  <img width="893" height="905" alt="image" src="https://github.com/user-attachments/assets/d353531f-f359-470c-96a0-fca420d9e4d2" /> <br />
  <img width="940" height="906" alt="image" src="https://github.com/user-attachments/assets/1598d8b2-4e7a-4291-940b-264a7da52334" />


- SELinux or permission problems might be an issue too. Check if SELinux is enforcing
  ```
  getenforce
  ```
  <img width="289" height="60" alt="image" src="https://github.com/user-attachments/assets/5a5e314c-3c01-4bd3-bc67-95c362fefc02" />


- Audit the logs for denials
  ```
  ausearch -m avc -ts recent | audit2why
  ```
  <img width="566" height="180" alt="image" src="https://github.com/user-attachments/assets/5b1ff80a-9192-46c4-b4bf-8595b0867f14" />


- Check for mislabeled system files
  ```
  ls -Z /etc/shadow
  ```
  <img width="351" height="70" alt="image" src="https://github.com/user-attachments/assets/35a9436c-fe81-40e6-aed1-f4fd2df0baa9" />

  
## Confirming Issues Found
- After the search is complete, we can check back the hidden generated text file located in the root folder. Since it is in the root folder, `sudo` needs to be included
  ```
  sudo cat /root/.injected_faults.txt
  ```
  <img width="486" height="150" alt="image" src="https://github.com/user-attachments/assets/0cae8da0-3ddd-4665-b1c2-f4ba2b7080fd" />
  
