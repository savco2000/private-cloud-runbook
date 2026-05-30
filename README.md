# Sovereign Architecture: Bare-Metal to Virtual Lab Runbook

An automated, hyper-efficient blueprint for transforming standard consumer hardware into a deterministic, containerized local cloud infrastructure. By adhering to the **"Thin Host"** paradigm, this architecture decouples the primary development runtime environments from the physical operating system, establishing absolute workspace isolation, configuration reproducibility, and hardware performance efficiency.

## 🪐 Core Architecture Highlights

* **The Thin Host Configuration:** Automates the complete provisioning of an Ubuntu 26.04 LTS (Resolute Raccoon) hypervisor host using declarative `autoinstall` directives, enforcing pre-configured GDM custom configurations, dark mode system defaults, and an integrated secrets layout right out of the box.

* **Orchestration Engine (`deploy-vm.sh`):** A single unified control script providing dynamic virtualization command mappings via `virsh` and `qemu-img` with built-in parameter injection for memory allocations (`-m`), vCPU assignments (`-c`), and physical storage resizing (`-s`).

* **Storage Optimization via Linked Clones:** Implements QEMU copy-on-write (COW) backing layers to instantly spin up disposable guest machines without duplicating the underlying 40GB root disk space footprint.

* **Deterministic Configuration Lifecycle:** Leverages local NoCloud cloud-init data-source wrappers to automatically wire up container engines, system packages, global development environments, network configurations, and drop-proof terminal sessions (`byobu`) synchronously before terminating the host thread.

---

## Phase 0: The "Off-Laptop" Preparation

Before you wipe your drive, prepare these three "External Keys" on other devices.

1. **Plug-in host's Ethernet cable:** Make sure your machine is physically plugged in. Otherwise, the installation will fail.

2. **The Installer USB:** Flash the Ubuntu 26.04 LTS (Resolute Raccoon) Desktop ISO to a USB drive.

3. **The CIDATA USB:** Format a second small USB drive as **FAT32** with the volume label exactly `CIDATA`.

    - Create a directory named `Lifeboat` under `~/Downloads/Lifeboat/`.

      ```bash
      mkdir $HOME/Downloads/Lifeboat
      ```

    - Create a file named `meta-data` at `~/Downloads/Lifeboat/` and leave it empty.

      ```bash
      touch $HOME/Downloads/Lifeboat/meta-data
      ```

    - Create a file named `user-data` at `~/Downloads/Lifeboat`.

      ```bash
      cat << 'OUTER_EOF' >> $HOME/Downloads/Lifeboat/user-data
      #cloud-config
      autoinstall:
        version: 1
        locale: en_US.UTF-8
        keyboard: {layout: us}
        timezone: America/New_York
        identity:
          hostname: ubuntu-host
          realname: "Primary Developer"
          username: devuser
          # NOTE: You MUST replace this with a real hashed password.
          # Generate one in your terminal using: mkpasswd -m sha-512
          password: "$6$EXAMPLERANDOM$HashedPassword..."
        ssh:
          install-server: true
          # NOTE: You MUST replace this with the contents of your id_ed25519.pub
          # Generate one in your terminal using: ssh-keygen -t ed25519 -C "devuser@ubuntu-host"
          authorized-keys:
            - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...your_key... 
        storage:
          layout:
            name: direct
            match:
              path: /dev/nvme0n1
        packages:
          - qemu-system-x86
          - qemu-utils
          - cloud-image-utils
          - libvirt-daemon-system
          - libvirt-clients
          - virt-manager
          - gimp
          - tlp
          - gnupg2
          - pass
          - stow
          - pinentry-gnome3
          - xclip
          - curl
          - jq
          - whois
          - usb-creator-gtk
        snaps:
          - name: code
            classic: true
          - name: slack
            classic: true
          - name: zoom-client
        late-commands:
          - curtin in-target -- wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb -O /tmp/chrome.deb
          - curtin in-target -- apt-get install -y /tmp/chrome.deb
          - curtin in-target -- ubuntu-report send yes
        user-data:
          runcmd:
            # 1. Remove Firefox now that the system is live
            - snap remove --purge firefox

            # 2. SET SYSTEM-WIDE DESKTOP DEFAULTS VIA DCONF
            - |
              mkdir -p /etc/dconf/db/local.d/
              cat <<EOF > /etc/dconf/db/local.d/00-sovereign-desktop
              [org/gnome/shell]
              favorite-apps=['google-chrome.desktop', 'code_code.desktop', 'org.gnome.Ptyxis.desktop', 'slack_slack.desktop', 'zoom-client_zoom-client.desktop', 'virt-manager.desktop', 'org.gnome.Nautilus.desktop']

              [org/gnome/shell/extensions/dash-to-dock]
              show-trash=true
              trash-at-the-end=true

              [org/gnome/desktop/interface]
              color-scheme='prefer-dark'
              EOF
              
              mkdir -p /etc/dconf/profile/
              cat <<EOF > /etc/dconf/profile/user
              user-db:user
              system-db:local
              EOF
              
              dconf update

            # 3. DISABLE GNOME INITIAL SETUP (The "Welcome" Screen & Popup Fix)
            - |
              sed -i '/\[daemon\]/a InitialSetupEnable=false' /etc/gdm3/custom.conf
              for dir in /etc/skel /home/devuser; do
                mkdir -p "$dir/.config"
                echo "yes" > "$dir/.config/gnome-initial-setup-done"
              done
              chown -R devuser:devuser /home/devuser/.config 2>/dev/null || true
              apt-get purge -y gnome-initial-setup

            # 4. Modular SSH Configuration Setup for devuser
            - |
              mkdir -p /home/devuser/.ssh/conf.d
              touch /home/devuser/.ssh/config
              if ! grep -q "^Include conf.d/\*" /home/devuser/.ssh/config; then
                echo -e "Include conf.d/*\n$(cat /home/devuser/.ssh/config)" > /home/devuser/.ssh/config
              fi
              chown -R devuser:devuser /home/devuser/.ssh
              chmod 700 /home/devuser/.ssh
              chmod 600 /home/devuser/.ssh/config

            # 5. Install VS Code Extensions
            - sudo -u devuser code --install-extension ms-vscode-remote.remote-ssh
            - sudo -u devuser code --install-extension ms-vscode-remote.remote-containers

            # 6. Configure Git for devuser context
            - [ sudo, -u, devuser, git, config, --global, user.name, "Your Name" ]
            - [ sudo, -u, devuser, git, config, --global, user.email, "your@email.com" ]
            - [ sudo, -u, devuser, git, config, --global, init.defaultBranch, main ]

            # 7. PERFORMANCE OPTIMIZATIONS (Systemd Services)
            # Turn off non-essential blocking network checks and debugging overhead
            - |
              systemctl disable NetworkManager-wait-online.service
              systemctl disable fwupd-refresh.service
              systemctl disable kdump-tools.service
              systemctl disable apport.service
              systemctl disable ModemManager.service
              systemctl disable fstrim.service
              systemctl enable fstrim.timer

            # 8. SNAP REVISION MANAGEMENT
            # Mitigate virtual loop device inflation by enforcing strict storage retention limits
            - snap set system refresh.retain=2

            # 9. KERNEL SERIAL PROBE DISABLING
            # Patch the GRUB default configuration to bypass legacy UART infrastructure scans
            - |
              if [ -f /etc/default/grub ]; then
                sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash 8250.nr_uarts=0"/' /etc/default/grub
                update-grub
              fi
      OUTER_EOF
      ```

4. **Generate random salted hashed password**

    - Run the following command to generate a random salted hashed password.

      ```bash
      mkpasswd -m sha-512 | tee $HOME/Downloads/Lifeboat/hashed_password.txt
      ```

    - It will output a single line that looks something like this:

      `$6$EXAMPLERANDOM$HashedPassword...`

    - Update your **user-data**

      Copy that entire line and paste it into your host's `user-data` file under the `identity` section:

      ```yaml
      identity:
        hostname: ubuntu-host
        realname: "Primary Developer"
        username: devuser
        password: "$6$EXAMPLERANDOM$HashedPassword..."
      ```

5. **Generate the SSH key**

    - Run this command on the machine you will be using to access your host (e.g., your current laptop):

      ```bash
      ssh-keygen -t ed25519 -C "devuser@ubuntu-host"
      ```

    - Follow the Prompts
      - **Enter file in which to save the key:** Press Enter to accept the default location (`~/.ssh/id_ed25519`).
      - **Enter passphrase:** It is highly recommended to enter a passphrase. This adds a second layer of security.

    - Identify Your Keys
      
      The command creates two files in your `~/.ssh/` directory:
      - `id_ed25519` **(Private Key):** This stays on your laptop. Never share it.
      - `id_ed25519.pub` **(Public Key):** This is the one we inject.

    - Extract the Public Key for your **user-data**

      To get the string you need for your autoinstall file, run:

      ```bash
      cat ~/.ssh/id_ed25519.pub
      ```

      It will output a single line that looks something like this:

      `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... devuser@ubuntu-host`

      - Update your **user-data**

        Copy that entire line and paste it into your host's `user-data` file under the `ssh` section:

        ```yaml
        ssh:
          install-server: true
          authorized-keys:
            - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... devuser@ubuntu-host
        ```

      - Add your Key to GitHub
        - Go to **GitHub Settings > SSH and GPG keys > New SSH Key**.
        - Paste the key and give it a name like "Thin Host Laptop."

      - Copy your SSH keys to `~/Downloads/Lifeboat`

        ```bash
        cp $HOME/.ssh/id_ed25519.pub $HOME/.ssh/id_ed25519.pub $HOME/Downloads/Lifeboat/
        ```

6. **The "Lifeboat" USB:** Copy `~/Downloads/Lifeboat` to a standard storage USB.

## Phase 1: The "Thin Host" Installation

1. Plug both the **Installer** and **CIDATA** drives into your laptop.

2. Reboot machine and press F12 to bring up the boot menu.

3. Boot from the Installer USB. At the GRUB menu, highlight **"Install Ubuntu"**.

## Phase 2: Host Personalization (Secrets & Dotfiles)

Once you log in to your fresh host, establish your sovereignty.

1. Copy contents of Lifeboat USB to `$HOME/Downloads/Lifeboat`

1. **Restore SSH** 
    - Copy `id_ed25519` and `id_ed25519.pub` to `~/.ssh/`

      ```bash
      mv $HOME/Downloads/Lifeboat/id_ed25519 $HOME/Downloads/Lifeboat/id_ed25519.pub $HOME/.ssh/
      ```

    - Restrict permissions on `id_ed25519`

      ```bash
      chmod 600 ~/.ssh/id_ed25519
      ```

2. **Set up secrets management**
    - Generate GPG key
      ```bash
      gpg --full-generate-key   
      
      # At the "Please select what kind of key you want:" prompt, select "(9) ECC (sign and encrypt) *default*"
      # At the "Please select which elliptic curve you want:" prompt, select "(1) Curver 25519 *default*"
      # At the "Please specify how long the key should be valid>" prompt, select "0 = key does not expire"
      # Follow the rest of the prompts
      ```
    - Backup GPG key
      ```bash
      gpg --export-secret-key <YOUR_KEY_ID> | paperkey --output-type base16 > $HOME/Downloads/Lifeboat/gpg_paper.txt
      ```
    - Initialize Pass
      ```bash
      pass init <YOUR_KEY_ID>
      ```
    - Sync your secrets to your private GitHub repo

      ```bash
      pass git init

      # pass git config --global user.name "Your Name"
      # pass git config --global user.email "your@email.com"
      # pass git config --global init.defaultBranch main

      # Go to GitHub and create a new private repository called "my-passwords"

      pass git remote add origin git@github.com:youruser/my-passwords.git
      pass git push -u origin main
      ```

3. **Restore Dotfiles**

    ```bash
    git clone git@github.com:youruser/private-dotfiles.git ~/.dotfiles

    cd ~/.dotfiles && stow bash nvim git host-only
    ```

## Phase 3: The Virtual Lab

### 1. Create an executable installation script

1. Create `deploy-vm.sh` at `~/Downloads/`

    ```bash
    #!/bin/bash
    # Make executable: chmod +x deploy-vm.sh
    # Usage: sudo ./deploy-vm.sh <vm-name> [-m memory_mib] [-c vcpus] [-s disk_gib] [-d] [-f]
    # Default: 4096 MiB RAM, 4 vCPUs, 40 GiB Disk (Linked Clone)

    # --- 1. Dynamic Variables & Defaults ---
    VM_NAME=$1
    if [ -z "$VM_NAME" ]; then
        echo "❌ Error: No VM name specified."
        echo "Usage: $0 <vm-name> [-m memory] [-c vcpus] [-s size_gb] [-d] [-f]"
        exit 1
    fi

    MEM=4096
    CPUS=4
    DISK=40

    shift

    # --- 2. Flag Handling ---
    while getopts "dfm:c:s:" opt; do
      case $opt in
        d)
          echo "🔥 Self-Destruct Initiated for $VM_NAME..."
          sudo virsh destroy "$VM_NAME" 2>/dev/null
          sudo virsh undefine "$VM_NAME" --remove-all-storage 2>/dev/null
          sudo rm -f "/var/lib/libvirt/images/$VM_NAME-meta-data" "/var/lib/libvirt/images/$VM_NAME-seed.iso"
          echo "✨ Environment cleared."
          exit 0
          ;;
        f)
          echo "♻️ Force flag detected. Removing local base image..."
          sudo rm -f "/var/lib/libvirt/images/resolute-base.img"
          ;;
        m) MEM=$OPTARG ;;
        c) CPUS=$OPTARG ;;
        s) DISK=$OPTARG ;;
        \?) exit 1 ;;
      esac
    done

    # --- 3. Internal Paths ---
    PREFIX=$(echo "$VM_NAME" | sed 's/-vm$//')
    USER_DATA="./${PREFIX}-user-data.yaml"
    LIBVIRT_DIR="/var/lib/libvirt/images"
    BASE_IMG="$LIBVIRT_DIR/resolute-base.img"
    VM_DISK="$LIBVIRT_DIR/$VM_NAME.qcow2"
    META_DATA="$LIBVIRT_DIR/$VM_NAME-meta-data"
    SEED_ISO="$LIBVIRT_DIR/$VM_NAME-seed.iso"
    BASE_IMG_URL="https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64v3.img"
    SHA_URL="https://cloud-images.ubuntu.com/resolute/current/SHA256SUMS"

    # --- 4. Pre-Flight Checks ---
    if [ ! -f "$USER_DATA" ]; then
        echo "❌ Error: Configuration file '$USER_DATA' not found."
        exit 1
    fi

    if [ "$EUID" -ne 0 ]; then 
      echo "Please run as root (sudo) for deployment"
      exit
    fi

    # Detect the real user who invoked sudo and discover their real home directory
    REAL_USER=${SUDO_USER:-$USER}
    REAL_HOME=$(getent passwd "$REAL_USER" | cut -d: -f6)

    # --- 5. Synchronize & Verify Base Image ---
    if [ ! -f "$BASE_IMG" ]; then
        echo "📡 Downloading Ubuntu Cloud Image..."
        wget -c --no-verbose -P "$LIBVIRT_DIR" "$BASE_IMG_URL"
        mv "$LIBVIRT_DIR/resolute-server-cloudimg-amd64v3.img" "$BASE_IMG" 2>/dev/null
    fi

    echo "🛡️ Verifying integrity..."
    curl -s "$SHA_URL" | grep "resolute-server-cloudimg-amd64v3.img" > /tmp/sha256
    sed -i "s/resolute-server-cloudimg-amd64v3.img/resolute-base.img/" /tmp/sha256
    (cd "$LIBVIRT_DIR" && sha256sum --check --status /tmp/sha256) || { echo "❌ Checksum failed!"; exit 1; }

    # --- 6. Provision & Seed ---
    echo "🧹 Cleaning up existing $VM_NAME resources..."
    sudo virsh destroy "$VM_NAME" 2>/dev/null
    sudo virsh undefine "$VM_NAME" --remove-all-storage 2>/dev/null

    echo "🌱 Generating Cloud-Init seed..."
    cat <<EOF > "$META_DATA"
    instance-id: $VM_NAME-$(date +%s)
    local-hostname: $VM_NAME
    EOF
    cloud-localds "$SEED_ISO" "$USER_DATA" "$META_DATA"

    echo "💾 Creating linked clone: $VM_DISK (${DISK}GB)..."
    sudo qemu-img create -f qcow2 -b "$BASE_IMG" -F qcow2 "$VM_DISK" "${DISK}G"

    # --- 7. Launch ---
    echo "🚀 Launching $VM_NAME ($MEM MiB RAM, $CPUS vCPUs)..."
    virt-install \
      --name "$VM_NAME" \
      --osinfo ubuntu-lts-latest \
      --cpu host-model \
      --memory "$MEM" \
      --vcpus "$CPUS" \
      --import \
      --disk path="$VM_DISK" \
      --disk path="$SEED_ISO",device=cdrom \
      --network network=default \
      --graphics none \
      --noautoconsole

    # --- 8. Post-Flight: Monitor the Black Box ---
    echo "⏳ Waiting for VM to claim an IP..."
    MAX_RETRIES=30
    COUNT=0
    while [ -z "$VM_IP" ] && [ $COUNT -lt $MAX_RETRIES ]; do
        VM_IP=$(virsh domifaddr "$VM_NAME" | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b")
        sleep 2
        ((COUNT++))
    done

    if [ -z "$VM_IP" ]; then
        echo "⚠️ IP detection timed out. Check manually with 'virsh domifaddr $VM_NAME'."
        exit 1
    fi

    echo "🚀 $VM_NAME is live at $VM_IP. Waiting for configuration to finish..."

    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
        -o ConnectTimeout=5 devuser@$VM_IP "cloud-init status --wait"

    # --- 9. Automated SSH Configuration ---
    echo "⚙️  Wiring up SSH configuration for $VM_NAME..."

    # Ensure the modular directory exists in the true invoking user's home folder
    mkdir -p "$REAL_HOME/.ssh/conf.d"

    # Create or overwrite the VM's specific config file path dynamically
    cat << EOF > "$REAL_HOME/.ssh/conf.d/$VM_NAME"
    Host $VM_NAME
      HostName $VM_IP
      User devuser
      IdentityFile ~/.ssh/id_ed25519
      ForwardAgent yes
      IdentitiesOnly yes
      StrictHostKeyChecking no
      UserKnownHostsFile /dev/null
    EOF

    # Ensure permissions and system ownership are handed back to the normal user context
    chown -R "$REAL_USER:$REAL_USER" "$REAL_HOME/.ssh/conf.d"
    chmod 700 "$REAL_HOME/.ssh/conf.d"
    chmod 600 "$REAL_HOME/.ssh/conf.d/$VM_NAME"

    echo "------------------------------------------------"
    echo "✅ $VM_NAME is FULLY PROVISIONED and ready!"
    echo "------------------------------------------------"
    echo "Please wait a couple of minutes before connecting by typing:"
    echo "ssh $VM_NAME"
    echo "as the installation may still be completing."
    echo "------------------------------------------------"
    ```

2. Make `deploy-vm.sh` executable

    ```bash
    chmod +x deploy-vm.sh
    ```

### 2. Create the Virtual Machines

1. Create the `dotnet-vm` virtual machine

    - Create `dotnet-user-data.yaml` at `~/Downloads/`

      ```bash
      # 1. Dynamically pull identity and SSH keys from the password manager
      GIT_NAME=$(pass show github/personal | grep "^username:" | cut -d' ' -f2-)
      GIT_EMAIL=$(pass show github/personal | grep "^email:" | cut -d' ' -f2)
      SSH_PUB_KEY=$(pass show ssh | grep "^public_key:" | cut -d' ' -f2-)

      # 2. Generate the configuration file with variables injected
      # (Using > instead of >> to ensure we overwrite cleanly on rebuilds)
      cat << EOF > $HOME/Downloads/dotnet-user-data.yaml
      #cloud-config
      users:
        - name: devuser
          groups: [sudo]
          shell: /bin/bash
          sudo: ALL=(ALL) NOPASSWD:ALL # Explicitly grant devuser passwordless sudo
          lock_passwd: true # 🔒 Locks password authentication entirely
          ssh_authorized_keys:
            - $SSH_PUB_KEY

      packages:
        - docker.io
        - docker-buildx
        - git
        - postgresql-client
        - curl
        - jq
        - htop
        - ncdu
        - byobu

      runcmd:
        # 1. Adding the user to the docker group safely after package installation
        - usermod -aG docker devuser
        
        # 2. Configuring Git identity for devuser
        - [ sudo, -u, devuser, git, config, --global, user.name, "$GIT_NAME" ]
        - [ sudo, -u, devuser, git, config, --global, user.email, "$GIT_EMAIL" ]
        - [ sudo, -u, devuser, git, config, --global, init.defaultBranch, main ]

        # 3. Enable Byobu auto-launch on login for devuser
        - [ sudo, -u, devuser, byobu-enable ]
      EOF
      ```

    - Create the `dotnet-vm` virtual machine

      Run the following command to create `dotnet-vm`

      ```bash
      sudo ./deploy-vm.sh dotnet-vm -m 4096 -c 4 -s 40 -f
      ```

    - SSH into `dotnet-vm`

      ```bash
      ssh dotnet-vm
      ```

    - Verify installation

      - Check Docker Permissions:

        ```bash
        docker ps
        ```

      - Check .NET 10 (via Docker):

        ```bash
        docker run --rm mcr.microsoft.com/dotnet/sdk:10.0-preview dotnet --version
        ```
      - Check Git Identity:

        ```bash
        git config --global -l
        ```

2. Create the `openclaw-vm` virtual machine

    - Create `openclaw-user-data.yaml` file at `~/Downloads/`

      ```bash
      # 1. Dynamically pull identity and SSH keys from the password manager
      GIT_NAME=$(pass show github/personal | grep "^username:" | cut -d' ' -f2-)
      GIT_EMAIL=$(pass show github/personal | grep "^email:" | cut -d' ' -f2)
      SSH_PUB_KEY=$(pass show ssh | grep "^public_key:" | cut -d' ' -f2-)

      # 2. Generate the configuration file with variables injected
      # (Using > instead of >> to ensure we overwrite cleanly on rebuilds)
      cat << EOF > $HOME/Downloads/openclaw-user-data.yaml
      #cloud-config
      users:
        - name: devuser
          groups: [sudo]
          shell: /bin/bash
          sudo: ALL=(ALL) NOPASSWD:ALL # Explicitly grant devuser passwordless sudo access:
          lock_passwd: true # 🔒 Locks password authentication entirely
          ssh_authorized_keys:
            - $SSH_PUB_KEY

      packages:
        - docker.io
        - docker-buildx
        - nodejs
        - npm
        - git
        - curl
        - jq
        - tcpdump
        - auditd
        - byobu

      runcmd:
        # 1. Safely attach devuser to docker now that the package is installed
        - [ usermod, -aG, docker, devuser ]
        
        # 2. System-wide global installation of the OpenClaw CLI
        - [ npm, install, -g, openclaw@latest ]
        
        # 3. Provision the workspace securely using devuser's explicit context
        - [ sudo, -u, devuser, mkdir, -p, /home/devuser/claw-workspace ]

        # 4. Configuring Git identity for devuser
        - [ sudo, -u, devuser, git, config, --global, user.name, "$GIT_NAME" ]
        - [ sudo, -u, devuser, git, config, --global, user.email, "$GIT_EMAIL" ]
        - [ sudo, -u, devuser, git, config, --global, init.defaultBranch, main ]

        # 5. Enable Byobu auto-launch on login for devuser
        - [ sudo, -u, devuser, byobu-enable ]
      EOF
      ```

    - Create the `openclaw-vm` virtual machine

      Run the following command to create `openclaw-vm`

      ```bash
      sudo ./deploy-vm.sh openclaw-vm -m 4096 -c 4 -s 40 -f
      ```

    - SSH into `openclaw-vm`

      ```bash
      ssh openclaw-vm
      ```

    - Verify installation
      - Run `openclaw gateway start`
      - Open an SSH tunnel from your host: `ssh -L 18789:localhost:18789 devuser@openclaw-ip`
      - Access the UI at `http://localhost:18789`
      - Before running a risky AI experiment, freeze the target image: `virsh snapshot-create-as openclaw-vm pre-experiment`

## Phase 4: Managing Host and Virtual Machines

## 1. Updating Host's Firmware

Update host's firmware once a month.

```bash
fwupdmgr update
```

## 2. Listing VMs

To see what is running or what has been created on your host, use the `list` command.

### a. List only ACTIVE (running) VMs:

```bash
virsh list
```

### b. List ALL VMs (running, paused, or stopped):

```bash
virsh list --all
```

## 3. Starting a VM

To power on a virtual machine that is currently shut down:

```bash
virsh start <vm-name>
# Example: virsh start dotnet-vm
```

## 4. Stopping a VM

There are two ways to turn off a VM, depending on whether you want a polite request or an instant pull of the virtual power cord.

### a. Graceful Shutdown (Polite Request):

This sends an ACPI shutdown signal to the guest OS, letting Ubuntu close its processes cleanly.

```bash
virsh shutdown <vm-name>
# Example: virsh shutdown openclaw-vm
```
### b. Forceful Stop (The Virtual Power Cord):

If a VM is frozen or if you are about to run your destroy script, this instantly kills the power to the virtual instance.

```bash
virsh destroy <vm-name>
# Example: virsh destroy dotnet-vm
```

## 5. Creating a Snapshot of a VM

```bash
virsh snapshot-create-as <vm-name> pre-experiment
# Example: virsh snapshot-create-as openclaw-vm pre-experiment
```

## 6. Checking IP Addresses

You can grab a running VM's network information without using the full console anytime by running

```bash
virsh domifaddr <vm-name>
# Example: virsh domifaddr openclaw-vm
```