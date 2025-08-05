# Troubleshooting Simulated Injected System Faults on Oracle Linux
This homelab project simulates system faults on Oracle Linux VM to practice troubleshooting skills. It is optimised for low-resource environments, ideal for single-laptop homelabs and virtual setups.
The idea is to run the provided script to randomise system issues that will happen to provide a black-box environment, and troubleshoot those issues in a strategical approach.


1. [Preparation and Initial Setup](#preparation-and-initial-setup)
2. [


## Preparation and Initial Setup
- In the Oracle Linux VM, run the following script to randomise problems that will occur in the VM
  ```
  #!/bin/bash
  # Simulate and Troubleshoot: Black-box Fault Injector for Oracle Linux
  
  set -e
  
  echo "[*] Starting black-box fault injection..."
  
  # Log file for your own future reference (optional)
  logfile="/var/log/fault_injection.log"
  mkdir -p /root/backup_configs
  echo "[+] $(date): Starting fault injection" >> "$logfile"
  
  # Define all fault functions
  inject_dns_failure() {
    echo "nameserver 127.0.0.1" > /etc/resolv.conf
    echo "DNS failure injected" >> "$logfile"
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
    echo "Systemd failure injected" >> "$logfile"
  }
  
  inject_fstab_error() {
    echo "/dev/fakevolume /mnt/fake ext4 defaults 0 0" >> /etc/fstab
    echo "Fstab error injected" >> "$logfile"
  }
  
  inject_sudo_broken() {
    echo "badentry" >> /etc/sudoers
    echo "Sudoers syntax error injected" >> "$logfile"
  }
  
  inject_cron_failure() {
    echo "* * * * * root /bin/bash -c 'echo MissingQuote" > /etc/cron.d/brokenjob
    echo "Cron job failure injected" >> "$logfile"
  }
  
  inject_full_disk() {
    fallocate -l 500M /var/bloatfile
    echo "Disk fill simulated" >> "$logfile"
  }
  
  inject_ssh_config_error() {
    echo "PermitRootLogin maybe" >> /etc/ssh/sshd_config
    systemctl restart sshd || true
    echo "SSH misconfiguration injected" >> "$logfile"
  }
  
  inject_selinux_error() {
    if sestatus | grep -q "enabled"; then
      chcon -t httpd_sys_content_t /etc/shadow
      echo "SELinux mislabel injected" >> "$logfile"
    fi
  }
  
  # List of problem functions
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
  
  # Shuffle and inject 3 to 6 random problems
  num_faults=$(( RANDOM % 4 + 3 )) # 3–6
  shuffled_faults=($(shuf -e "${problems[@]}"))
  selected_faults=("${shuffled_faults[@]:0:$num_faults}")
  
  for fault in "${selected_faults[@]}"; do
    $fault
  done
  
  echo "[+] Fault injection complete. Total: $num_faults issues injected."
  ```



## Troubleshooting Approach
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
