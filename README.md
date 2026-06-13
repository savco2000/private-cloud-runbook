# Sovereign Architecture: Bare-Metal to Virtual Lab Runbook

An automated, hyper-efficient blueprint for transforming standard consumer hardware into a deterministic, containerized local cloud infrastructure. By adhering to the **"Thin Host"** paradigm, this architecture decouples the primary development runtime environments from the physical operating system, establishing absolute workspace isolation, configuration reproducibility, and hardware performance efficiency.

## 🪐 Core Architecture Highlights

* **The Thin Host Configuration:** Automates the complete provisioning of an Ubuntu 26.04 LTS (Resolute Raccoon) hypervisor host using declarative `autoinstall` directives, enforcing pre-configured GDM custom configurations, dark mode system defaults, and an integrated secrets layout right out of the box.

* **Orchestration Engine (`deploy-vm.sh`):** A single unified control script providing dynamic virtualization command mappings via `virsh` and `qemu-img` with built-in parameter injection for memory allocations (`-m`), vCPU assignments (`-c`), and physical storage resizing (`-s`).

* **Storage Optimization via Linked Clones:** Implements QEMU copy-on-write (COW) backing layers to instantly spin up disposable guest machines without duplicating the underlying 40GB root disk space footprint.

* **Deterministic Configuration Lifecycle:** Leverages local NoCloud cloud-init data-source wrappers to automatically wire up container engines, system packages, global development environments, network configurations, and drop-proof terminal sessions (`byobu`) synchronously before terminating the host thread.

## Quick Start

Use this if you want the fastest successful path without reading every detail first.

1. Verify prerequisites in the next section.
2. Run `forge-host.sh` to generate `user-data` and `meta-data`.
3. Prepare two USB drives: Ubuntu installer + `CIDATA`.
4. Copy `user-data` and `meta-data` to the `CIDATA` USB.
5. Optionally export `private_key.asc` and `ownertrust.txt` to your Lifeboat USB.
6. Install the host using the installer + `CIDATA` drives.
7. Restore GPG trust and SSH from Lifeboat on first host login.
8. Create `deploy-vm.sh` and make it executable.
9. Generate VM cloud-init files and deploy your first VM.
10. Verify VM health with `ssh <vm-name>`, `docker ps`, and `virsh list --all`.

## Prerequisites

Confirm these before Phase 0 to avoid mid-run failures.

- Host hardware supports hardware virtualization (VT-x/AMD-V enabled in BIOS/UEFI)

  ```bash
  egrep -c '(vmx|svm)' /proc/cpuinfo
  
  # If the result is > 0 you are good to go.
  ```
- Ubuntu 26.04 installer ISO downloaded
- Two USB drives available (installer + `CIDATA`)
- `pass` store initialized and accessible
- GPG key available for `pass` decryption
- GitHub SSH access working for password-store clone

Recommended pre-flight checks:

```bash
command -v pass gpg ssh-keygen virsh qemu-img cloud-localds >/dev/null && echo "tooling ok"
pass show host >/dev/null && echo "pass entry ok"
```

---

## Phase 0: The "Off-Laptop" Preparation

1. Create and run `forge-host.sh` which:
    - Creates the host's user-data and meta-data files in the same directory `forge-host.sh` is run from
    - Optionally rotates the host's password in `pass`
    - Optionally rotates the SSH keys in `pass`

    ```bash
    #!/bin/bash
    # Enable strict error handling: fail fast on errors or unset variables
    set -euo pipefail

    # Define a single source of truth for the output paths
    OUTPUT_FILE="$(dirname "$0")/user-data"
    META_DATA_FILE="$(dirname "$OUTPUT_FILE")/meta-data"

    mkdir -p "$(dirname "$OUTPUT_FILE")"
    touch "$META_DATA_FILE" # Ensure the empty meta-data file exists

    # --- 1. Optional Password Rotation ---
    echo "------------------------------------------------"
    read -p "🔄 Do you want to rotate the host password before generating? (y/N): " ROTATE_PWD
    if [[ "$ROTATE_PWD" =~ ^[Yy]$ ]]; then
        read -s -p "   Enter new plaintext password: " RAW_PWD
        echo ""
        read -s -p "   Confirm new plaintext password: " RAW_PWD_CONFIRM
        echo ""
        
        if [ "$RAW_PWD" != "$RAW_PWD_CONFIRM" ]; then
            echo "❌ Passwords do not match! Exiting script."
            exit 1
        fi
        
        echo "⚙️  Hashing password and updating 'host' entry in pass..."
        NEW_HASH=$(echo "$RAW_PWD" | mkpasswd -m sha-512 -s)
        
        # Pull the existing record, replace line 1 (plaintext), and replace the hashed-password line
        EXISTING_SECRETS=$(pass show host)
        UPDATED_SECRETS=$(echo "$EXISTING_SECRETS" | sed "1s|.*|$RAW_PWD|" | sed "s|^hashed-password:.*|hashed-password: $NEW_HASH|")
        
        # Pipe it back into pass (-f forces overwrite without prompting)
        echo "$UPDATED_SECRETS" | pass insert -f -m host > /dev/null
        
        echo "✅ Password successfully rotated and saved to your vault!"
    fi

    # --- 2. Optional SSH Key Rotation ---
    echo "------------------------------------------------"
    read -p "🔑 Do you want to rotate the SSH key pair? (y/N): " ROTATE_SSH
    if [[ "$ROTATE_SSH" =~ ^[Yy]$ ]]; then
        read -s -p "   Enter passphrase for new SSH key (leave blank for none): " SSH_PASS
        echo ""
        
        echo "⚙️  Generating new ed25519 SSH key pair..."
        # Create a secure temporary directory that only your user can access
        TEMP_SSH_DIR=$(mktemp -d)
        
        # Generate the key pair silently
        ssh-keygen -t ed25519 -f "$TEMP_SSH_DIR/id_ed25519" -N "$SSH_PASS" -C "devuser@ubuntu-host" -q
        
        # Read the newly generated keys
        NEW_PUB_KEY=$(cat "$TEMP_SSH_DIR/id_ed25519.pub")
        NEW_PRIV_KEY=$(cat "$TEMP_SSH_DIR/id_ed25519")
        
        echo "⚙️  Updating SSH keys in pass..."
        # Inject the keys into the password manager using the multiline (-m) flag
        echo "$NEW_PUB_KEY" | pass insert -f -m ssh/public-key > /dev/null
        echo "$NEW_PRIV_KEY" | pass insert -f -m ssh/private-key > /dev/null
        
        # Securely remove the temporary directory and its contents
        rm -rf "$TEMP_SSH_DIR"
        echo "✅ SSH keys successfully rotated and saved to your vault!"
        
        echo "⚠️  NOTE: Because you rotated your key, you MUST upload your new public key to GitHub!"
    fi
    echo "------------------------------------------------"

    # --- 3. Stage Local SSH Environment ---
    echo "⚙️  Staging local SSH keys for immediate use..."
    mkdir -p "$HOME/.ssh"
    chmod 700 "$HOME/.ssh"

    # Extract keys directly into the standard OpenSSH paths
    pass show ssh/private-key > "$HOME/.ssh/id_ed25519"
    pass show ssh/public-key > "$HOME/.ssh/id_ed25519.pub"

    # Lock down permissions to prevent "Bad owner or permissions" errors
    chmod 600 "$HOME/.ssh/id_ed25519"
    chmod 644 "$HOME/.ssh/id_ed25519.pub"
    echo "✅ SSH keys are staged and ready in ~/.ssh/"
    echo "------------------------------------------------"

    # --- 4. Dynamically pull identity and SSH keys ---
    # Fetch the host secrets exactly once to save GPG decryption overhead
    HOST_SECRETS=$(pass show host)
    GIT_SECRETS=$(pass show github/personal)

    SSH_PUB_KEY=$(pass show ssh/public-key | tr -d '\n')
    HASHED_PASSWORD=$(echo "$HOST_SECRETS" | grep "^hashed-password:" | cut -d' ' -f2)
    REAL_NAME=$(echo "$HOST_SECRETS" | grep "^realname:" | cut -d' ' -f2-)
    HOST_NAME=$(echo "$HOST_SECRETS" | grep "^hostname:" | cut -d' ' -f2-)
    USERNAME=$(echo "$HOST_SECRETS" | grep "^username:" | cut -d' ' -f2-)
    GIT_NAME=$(echo "$GIT_SECRETS" | grep "^username:" | cut -d' ' -f2-)
    GIT_EMAIL=$(echo "$GIT_SECRETS" | grep "^email:" | cut -d' ' -f2)

    # --- 5. Generate the configuration file with safe placeholders ---
    cat << 'OUTER_EOF' > "$OUTPUT_FILE"
    #cloud-config
    autoinstall:
      version: 1
      locale: en_US.UTF-8
      keyboard: {layout: us}
      timezone: America/New_York
      identity:
        hostname: __HOST_NAME_PLACEHOLDER__
        realname: __REAL_NAME_PLACEHOLDER__
        username: __USERNAME_PLACEHOLDER__
        password: __HASHED_PASSWORD_PLACEHOLDER__
      ssh:
        install-server: true
        authorized-keys:
          - __SSH_PUB_KEY_PLACEHOLDER__ 
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
        - paperkey
      snaps:
        - name: code
          classic: true
        - name: slack
          classic: true
        - name: zoom-client
      late-commands:
        - curtin in-target -- wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb -O /tmp/chrome.deb
        - curtin in-target -- apt-get install -y /tmp/chrome.deb
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
            for dir in /etc/skel /home/__USERNAME_PLACEHOLDER__; do
              mkdir -p "$dir/.config"
              echo "yes" > "$dir/.config/gnome-initial-setup-done"
            done
            chown -R __USERNAME_PLACEHOLDER__:__USERNAME_PLACEHOLDER__ /home/__USERNAME_PLACEHOLDER__/.config 2>/dev/null || true
            apt-get purge -y gnome-initial-setup

          # 4. Modular SSH Configuration Setup for __USERNAME_PLACEHOLDER__
          - |
            mkdir -p /home/__USERNAME_PLACEHOLDER__/.ssh/conf.d
            touch /home/__USERNAME_PLACEHOLDER__/.ssh/config
            if ! grep -q "^Include conf.d/\*" /home/__USERNAME_PLACEHOLDER__/.ssh/config; then
              printf "Include conf.d/*\n%s" "$(cat /home/__USERNAME_PLACEHOLDER__/.ssh/config)" > /home/__USERNAME_PLACEHOLDER__/.ssh/config
            fi
            chown -R __USERNAME_PLACEHOLDER__:__USERNAME_PLACEHOLDER__ /home/__USERNAME_PLACEHOLDER__/.ssh
            chmod 700 /home/__USERNAME_PLACEHOLDER__/.ssh
            chmod 600 /home/__USERNAME_PLACEHOLDER__/.ssh/config

          # 5. Install VS Code Extensions
          - sudo -u __USERNAME_PLACEHOLDER__ code --install-extension ms-vscode-remote.remote-ssh
          - sudo -u __USERNAME_PLACEHOLDER__ code --install-extension ms-vscode-remote.remote-containers

          # 6. Configure Git for __USERNAME_PLACEHOLDER__ context
          - [ sudo, -u, __USERNAME_PLACEHOLDER__, git, config, --global, user.name, "__GIT_NAME_PLACEHOLDER__" ]
          - [ sudo, -u, __USERNAME_PLACEHOLDER__, git, config, --global, user.email, "__GIT_EMAIL_PLACEHOLDER__" ]
          - [ sudo, -u, __USERNAME_PLACEHOLDER__, git, config, --global, init.defaultBranch, main ]

          # 7. PERFORMANCE OPTIMIZATIONS (Systemd Services)
          - |
            systemctl disable NetworkManager-wait-online.service
            systemctl disable fwupd-refresh.service
            systemctl disable kdump-tools.service
            systemctl disable apport.service
            systemctl disable ModemManager.service
            systemctl disable fstrim.service
            systemctl enable fstrim.timer

          # 8. SNAP REVISION MANAGEMENT
          - snap set system refresh.retain=2

          # 9. KERNEL SERIAL PROBE DISABLING
          - |
            if [ -f /etc/default/grub ]; then
              sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash 8250.nr_uarts=0"/' /etc/default/grub
              update-grub
            fi
    OUTER_EOF

    # --- 6. Safely inject all parameters in a single disk I/O operation ---
    sed -i \
      -e "s|__SSH_PUB_KEY_PLACEHOLDER__|$SSH_PUB_KEY|g" \
      -e "s|__HASHED_PASSWORD_PLACEHOLDER__|$HASHED_PASSWORD|g" \
      -e "s|__REAL_NAME_PLACEHOLDER__|$REAL_NAME|g" \
      -e "s|__HOST_NAME_PLACEHOLDER__|$HOST_NAME|g" \
      -e "s|__USERNAME_PLACEHOLDER__|$USERNAME|g" \
      -e "s|__GIT_NAME_PLACEHOLDER__|$GIT_NAME|g" \
      -e "s|__GIT_EMAIL_PLACEHOLDER__|$GIT_EMAIL|g" \
      "$OUTPUT_FILE"

    echo "✨ user-data and meta-data files successfully generated at $(dirname "$OUTPUT_FILE")/"

    # --- 7. Export GPG Private Key & Ownertrust ---
    echo "------------------------------------------------"
    read -p "🔐 Do you want to export your GPG private key and ownertrust for backup? (y/N): " EXPORT_GPG
    if [[ "$EXPORT_GPG" =~ ^[Yy]$ ]]; then
        echo "⚙️  Exporting GPG keys for $GIT_EMAIL..."
        PRIVATE_KEY_FILE="$(dirname "$OUTPUT_FILE")/private_key.asc"
        OWNERTRUST_FILE="$(dirname "$OUTPUT_FILE")/ownertrust.txt"
        
        # We use the $GIT_EMAIL dynamically pulled from pass to identify the correct GPG key
        gpg --armor --export-secret-keys "$GIT_EMAIL" > "$PRIVATE_KEY_FILE"
        gpg --export-ownertrust > "$OWNERTRUST_FILE"
        
        # Secure the exported private key permissions so SSH/Linux doesn't complain later
        chmod 600 "$PRIVATE_KEY_FILE"
        
        echo "✅ GPG private key saved to: $PRIVATE_KEY_FILE"
        echo "✅ GPG ownertrust saved to: $OWNERTRUST_FILE"
    fi
    echo "------------------------------------------------"
    ```
2. **The Installer USB:** Flash the Ubuntu 26.04 LTS (Resolute Raccoon) Desktop ISO to a USB drive using Startup Disk Creator.

3. Prepare the **CIDATA** and **LIFEBOAT** USB Drives

    Use this workflow for the configuration USB and the backup "Lifeboat" USB. These commands erase the entire target drive, so confirm the device name carefully before continuing.

    a. List attached drives and identify the correct USB device by its size.

      ```bash
      lsblk
      ```

      Look for the base device name, such as `/dev/sda`. If the drive already has partitions, you may also see entries such as `/dev/sda1`.

    b. Unmount any mounted partition on that USB drive before wiping it.

      ```bash
      sudo umount /dev/sda1
      ```

    c. Remove the old filesystem signatures so the drive is a clean slate.

      ```bash
      sudo wipefs --all --force /dev/sda
      ```

    d. Format the entire USB drive as FAT32 with the correct label.

      - For the cloud-init drive:

          ```bash
          sudo mkfs.vfat -F 32 -I -n CIDATA /dev/sda
          ```

      - For the backup drive:

          ```bash
          sudo mkfs.vfat -F 32 -I -n LIFEBOAT /dev/sda
          ```

      Command breakdown:

      - `mkfs.vfat`: creates a FAT filesystem.
      - `-F 32`: forces FAT32 for broad compatibility.
      - `-I`: formats the whole base device instead of requiring a partition table.
      - `-n CIDATA` or `-n LIFEBOAT`: sets the volume label the host will look for.
      - `/dev/sda`: the target USB drive. Replace this with the correct base device from `lsblk`.

4. Copy `user-data` and `meta-data` files created by the `forge-host.sh` script above into the CIDATA USB drive.

5. Copy `private_key.asc` and `ownertrust.txt` to the LIFEBOAT USB drive. These files let you import your private key and restore your key trust mappings in the new machine's `pass`, so **keep this USB safe!**

**Phase 0 checkpoint:**

```bash
ls -l user-data meta-data
```

## Phase 1: The "Thin Host" Installation

1. **Plug-in host's Ethernet cable:** Make sure your machine is physically plugged in. Otherwise, the installation will fail.

2. Plug both the **Installer** and **CIDATA** drives into your laptop.

3. Reboot machine and press **F12** to bring up the boot menu.

4. Boot from the Installer USB. At the GRUB menu, highlight **"Install Ubuntu"**.

**Phase 1 checkpoint:**

- You can log into the newly installed host.
- `virsh --version` returns successfully.
- SSH directory exists at `$HOME/.ssh/`.

## Phase 2: Host Personalization (Secrets & SSH Recovery)

Once you log in to your fresh host, establish your sovereignty by importing your GPG private key and restore key trust mappings.

1. Create the Lifeboat folder
    
    ```bash
    mkdir $HOME/Lifeboat
    ```

2. Copy contents of Lifeboat USB from **Phase 0** to `$HOME/Lifeboat`

3. Setup SSH

    ```bash
    mkdir -p "$HOME/.ssh"
    chmod 700 "$HOME/.ssh"

    # Import the private key
    gpg --import $HOME/Lifeboat/private_key.asc

    # Restore your key trust mappings
    gpg --import-ownertrust $HOME/Lifeboat/ownertrust.txt

    # Clone the Password Store
    git clone git@github.com:savco2000/the-black-box.git $HOME/.password-store

    # Extract keys directly into the standard OpenSSH paths
    pass show ssh/private-key > "$HOME/.ssh/id_ed25519"
    pass show ssh/public-key > "$HOME/.ssh/id_ed25519.pub"

    # Lock down permissions to prevent "Bad owner or permissions" errors
    chmod 600 "$HOME/.ssh/id_ed25519"
    chmod 644 "$HOME/.ssh/id_ed25519.pub"

    # Optional: remove key export artifacts after successful import
    rm -f "$HOME/Lifeboat/private_key.asc" "$HOME/Lifeboat/ownertrust.txt"
    ```

  **Phase 2 checkpoint:**

  ```bash
  pass show ssh/public-key >/dev/null && echo "pass recovered"
  ssh -T git@github.com
  ```

## Phase 3: The Virtual Lab

### 1. Create an executable installation script

1. Create `deploy-vm.sh` at `~/Downloads/`

    ```bash
    #!/bin/bash
    # Make executable: chmod +x deploy-vm.sh
    # Usage: sudo ./deploy-vm.sh <config.yaml | vm-name> [-m memory_mib] [-c vcpus] [-s disk_gib] [-d] [-f]
    # Default: 4096 MiB RAM, 4 vCPUs, 40 GiB Disk (Linked Clone)

    # --- 1. Dynamic Variables & Defaults ---
    TARGET=$1
    if [ -z "$TARGET" ]; then
        echo "❌ Error: No target specified."
        echo "Usage: $0 <user-data.yaml | vm-name> [-m memory] [-c vcpus] [-s size_gb] [-d] [-f]"
        exit 1
    fi

    # Smart Input Parsing: Determine if target is a file or a VM name
    if [[ "$TARGET" =~ \.(yaml|yml)$ ]]; then
        # Input is a file path
        USER_DATA="$TARGET"
        FILENAME=$(basename "$TARGET")
        PREFIX=$(echo "$FILENAME" | sed -E 's/-user-data\.(yaml|yml)$//')
        VM_NAME="${PREFIX}-vm"
    else
        # Input is a string (VM name or prefix)
        if [[ "$TARGET" == *-vm ]]; then
            VM_NAME="$TARGET"
            PREFIX="${TARGET%-vm}"
        else
            PREFIX="$TARGET"
            VM_NAME="${PREFIX}-vm"
        fi
        # Guess the local user-data path in case they are creating rather than destroying
        USER_DATA="./${PREFIX}-user-data.yaml"
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
    while [ -z "${VM_IP:-}" ] && [ $COUNT -lt $MAX_RETRIES ]; do
        VM_IP=$(virsh domifaddr "$VM_NAME" | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" || true)
        sleep 2
        ((COUNT++))
    done

    if [ -z "${VM_IP:-}" ]; then
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
      #!/bin/bash
      # Enable strict error handling: fail fast on errors or unset variables
      set -euo pipefail

      # Dynamically set the output path to the exact directory where this script resides
      OUTPUT_FILE="$(dirname "$0")/dotnet-user-data.yaml"

      # 1. Dynamically pull identity and SSH keys from the password manager
      # Fetch git secrets once to halve the GPG decryption overhead
      GIT_SECRETS=$(pass show github/personal)
      GIT_NAME=$(echo "$GIT_SECRETS" | grep "^username:" | cut -d' ' -f2-)
      GIT_EMAIL=$(echo "$GIT_SECRETS" | grep "^email:" | cut -d' ' -f2)

      SSH_PUB_KEY=$(pass show ssh/public-key | tr -d '\n')
      USERNAME=$(pass show virtual-machine | grep "^username:" | cut -d' ' -f2-)

      # 2. Generate the configuration file with variables injected directly
      # (Unquoted EOF allows Bash to inject variables directly without needing sed)
      cat << EOF > "$OUTPUT_FILE"
      #cloud-config
      users:
        - name: $USERNAME
          groups: [sudo]
          shell: /bin/bash
          sudo: ALL=(ALL) NOPASSWD:ALL # Explicitly grant $USERNAME passwordless sudo
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
        - usermod -aG docker $USERNAME
        
        # 2. Configuring Git identity for $USERNAME
        - [ sudo, -u, $USERNAME, git, config, --global, user.name, "$GIT_NAME" ]
        - [ sudo, -u, $USERNAME, git, config, --global, user.email, "$GIT_EMAIL" ]
        - [ sudo, -u, $USERNAME, git, config, --global, init.defaultBranch, main ]

        # 3. Enable Byobu auto-launch on login for $USERNAME
        - [ sudo, -u, $USERNAME, byobu-enable ]
      EOF

      echo "✨ VM user-data file successfully generated at $OUTPUT_FILE"
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
      #!/bin/bash
      # Enable strict error handling: fail fast on errors or unset variables
      set -euo pipefail

      # Dynamically set the output path to the exact directory where this script resides
      OUTPUT_FILE="$(dirname "$0")/openclaw-user-data.yaml"

      # 1. Dynamically pull identity and SSH keys from the password manager
      # Fetch git secrets once to halve the GPG decryption overhead
      GIT_SECRETS=$(pass show github/personal)
      GIT_NAME=$(echo "$GIT_SECRETS" | grep "^username:" | cut -d' ' -f2-)
      GIT_EMAIL=$(echo "$GIT_SECRETS" | grep "^email:" | cut -d' ' -f2)

      SSH_PUB_KEY=$(pass show ssh/public-key | tr -d '\n')
      USERNAME=$(pass show virtual-machine | grep "^username:" | cut -d' ' -f2-)

      # 2. Generate the configuration file with variables injected
      # (Unquoted EOF allows Bash to inject variables directly without needing sed)
      cat << EOF > "$OUTPUT_FILE"
      #cloud-config
      users:
        - name: $USERNAME
          groups: [sudo]
          shell: /bin/bash
          sudo: ALL=(ALL) NOPASSWD:ALL # Explicitly grant $USERNAME passwordless sudo access:
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
        # 1. Safely attach $USERNAME to docker now that the package is installed
        - [ usermod, -aG, docker, $USERNAME ]
        
        # 2. System-wide global installation of the OpenClaw CLI
        - [ npm, install, -g, openclaw@latest ]
        
        # 3. Provision the workspace securely using $USERNAME's explicit context
        - [ sudo, -u, $USERNAME, mkdir, -p, /home/$USERNAME/claw-workspace ]

        # 4. Configuring Git identity for $USERNAME
        - [ sudo, -u, $USERNAME, git, config, --global, user.name, "$GIT_NAME" ]
        - [ sudo, -u, $USERNAME, git, config, --global, user.email, "$GIT_EMAIL" ]
        - [ sudo, -u, $USERNAME, git, config, --global, init.defaultBranch, main ]

        # 5. Enable Byobu auto-launch on login for $USERNAME
        - [ sudo, -u, $USERNAME, byobu-enable ]
      EOF

      echo "✨ OpenClaw user-data file successfully generated at $OUTPUT_FILE"
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
      - Open an SSH tunnel from your host: `ssh -L 18789:localhost:18789 openclaw-vm`
      - Access the UI at `http://localhost:18789`
      - Before running a risky AI experiment, freeze the target image: `virsh snapshot-create-as openclaw-vm pre-experiment`

    Phase 3 checkpoint:

    ```bash
    virsh list --all
    ssh dotnet-vm "cloud-init status"
    ```

## Phase 4: Managing Host and Virtual Machines

### 1. Updating Host's Firmware

Update host's firmware once a month.

```bash
fwupdmgr update
```

### 2. Listing VMs

To see what is running or what has been created on your host, use the `list` command.

### a. List only ACTIVE (running) VMs:

```bash
virsh list
```

### b. List ALL VMs (running, paused, or stopped):

```bash
virsh list --all
```

### 3. Starting a VM

To power on a virtual machine that is currently shut down:

```bash
virsh start <vm-name>
# Example: virsh start dotnet-vm
```

### 4. Stopping a VM

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

### 5. Creating a Snapshot of a VM

```bash
virsh snapshot-create-as <vm-name> pre-experiment
# Example: virsh snapshot-create-as openclaw-vm pre-experiment
```

### 6. Checking IP Addresses

You can grab a running VM's network information without using the full console anytime by running

```bash
virsh domifaddr <vm-name>
# Example: virsh domifaddr openclaw-vm
```

## Common Issues and Fast Fixes

1. `cloud-init status --wait` hangs longer than expected
  - Check from host: `ssh <vm-name> "sudo tail -n 100 /var/log/cloud-init-output.log"`
2. VM has no IP address in `virsh domifaddr`
  - Confirm default network is up: `virsh net-start default && virsh net-autostart default`
3. SSH says bad permissions on private key
  - Fix with: `chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_ed25519 && chmod 644 ~/.ssh/id_ed25519.pub`
4. `pass` fails to decrypt
  - Re-import key and trust: `gpg --import private_key.asc && gpg --import-ownertrust ownertrust.txt`