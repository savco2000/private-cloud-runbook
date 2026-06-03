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
      mkdir -p $HOME/Lifeboat/host
      ```

    - Create a file named `meta-data` at `~/Lifeboat/` and leave it empty.

      ```bash
      touch $HOME/Lifeboat/host/meta-data
      ```

    - Create a file named `user-data` at `~/Downloads/Lifeboat/host`.

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