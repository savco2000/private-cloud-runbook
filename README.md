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
2. Run `forge-host.sh` to generate:
    - `user-data`
    - `meta-data`
    - `private_key.asc`
    - `ownertrust.txt`
3. Prepare three USB drives:
    - Ubuntu 26.04 installer (8 GB to 16 GB)
    - `CIDATA` (1 GB to 8 GB)
    - `LIFEBOAT` (8 GB to 32 GB)
4. Copy `user-data` and `meta-data` to the `CIDATA` USB drive.
5. Export `private_key.asc` and `ownertrust.txt` to your `LIFEBOAT` USB drive.
6. Install the host using the Ubuntu 26.04 installer + `CIDATA` USB drives.
7. Restore GPG trust and SSH from the `LIFEBOAT` USB drive on first host login.
8. Create `deploy-vm.sh` and make it executable by running `chmod +x deploy-vm.sh`.
9. Generate VM cloud-init files (`dev-user-data-yaml` etc ) and deploy your first VM.
10. Verify VM health with `ssh <vm-name>`, `docker ps` and `virsh list --all`.

## Prerequisites

Confirm these before Phase 0 to avoid mid-run failures.

- Host hardware supports hardware virtualization (VT-x/AMD-V enabled in BIOS/UEFI)

  ```bash
  egrep -c '(vmx|svm)' /proc/cpuinfo

  # If the result is > 0 you are good to go.
  ```
- Ubuntu 26.04 installer ISO downloaded
- Three USB drives available:
  - Ubuntu 26.04 installer (8 GB to 16 GB)
  - `CIDATA` (1 GB to 8 GB)
  - `LIFEBOAT` (8 GB to 32 GB)
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

1. Clone `dofiles` into `$HOME` directory if it doesn't already exist.

    ```bash
    cd $HOME
    git clone git@github.com:savco2000/dotfiles.git
    ```

2. Activate symlinks

    ```bash
    cd $HOME/dotfiles
    stow .
    ```

3. Make all the `.sh` files in the `install_scripts` folder executable

    ```bash
    cd $HOME/install_scripts
    find . -type f -name "*.sh" -exec chmod +x {} +
    ```

4. Run `forge-host.sh`:

    ```bash
    $HOME/install_scripts/host/./forge-host.sh    
    ```
    This will:
      - Create:

        - `user-data`
        - `meta-data`
        - `private_key.asc`
        - `ownertrust.txt`

        files in `$HOME/install_scripts/host`
      - Optionally rotates the host's password in `pass`
      - Optionally rotates the SSH keys in `pass`

5. **The Installer USB:** Flash the Ubuntu 26.04 LTS (Resolute Raccoon) Desktop ISO to a USB drive using Startup Disk Creator.

6. Prepare the `CIDATA` and `LIFEBOAT` USB Drives

    Use this workflow for the configuration USB and the backup `LIFEBOAT` USB. These commands erase the entire target drive, so confirm the device name carefully before continuing.

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

7. Copy `user-data` and `meta-data` files created by the `forge-host.sh` script above into the CIDATA USB drive.

8. Copy `private_key.asc` and `ownertrust.txt` to the `LIFEBOAT` USB drive. These files let you import your private key and restore your key trust mappings in the new machine's `pass`, so **keep this USB safe!**

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

1. Once you log in to your fresh host, establish your sovereignty by importing your GPG private key and restore key trust mappings.

    - Plug in the LIFEBOAD USB
        
    - Run the following script

        ```bash
        #!/bin/bash
        # Enable strict error handling: fail fast on errors or unset variables
        set -euo pipefail
        
        SOURCE_DIR="/run/media/$USER/LIFEBOAT"
        TARGET_DIR="$HOME/Lifeboat"
        DOTFILES_DIR="$HOME/dotfiles"
        
        if [ ! -d "$SOURCE_DIR" ]; then
            echo "Error: LIFEBOAT drive is not mounted at $SOURCE_DIR"
            exit 1
        fi
        
        mkdir -p "$TARGET_DIR"
        
        echo "LIFEBOAT detected. Copying files to $TARGET_DIR..."
        rsync -av "$SOURCE_DIR/" "$TARGET_DIR/"
        
        echo "Backup complete!"
        
        # Setup SSH
        mkdir -p "$HOME/.ssh"
        chmod 700 "$HOME/.ssh"
        
        if compgen -G "$TARGET_DIR/id_ed25519*" > /dev/null; then
            mv "$TARGET_DIR"/id_ed25519* "$HOME/.ssh/"
        fi
        
        # Lock down permissions to prevent "Bad owner or permissions" errors
        if [ -f "$HOME/.ssh/id_ed25519" ]; then
            chmod 600 "$HOME/.ssh/id_ed25519"
        fi
        if [ -f "$HOME/.ssh/id_ed25519.pub" ]; then
            chmod 644 "$HOME/.ssh/id_ed25519.pub"
        fi
        
        # Setup Password Store
        # Explicitly bind your terminal window to the GPG subsystem
        export GPG_TTY="$(tty)"
        
        # Execute the import command while forcing loopback mode
        gpg --pinentry-mode loopback --import "$TARGET_DIR/private_key.asc"
        
        # Restore your key trust mappings
        gpg --import-ownertrust "$TARGET_DIR/ownertrust.txt"
        
        # Scan and save the GitHub SSH key
        ssh-keyscan github.com >> ~/.ssh/known_hosts
        
        # Clone the Password Store
        git clone git@github.com:savco2000/the-black-box.git $HOME/.password-store
        
        if [ -f "$HOME/.bashrc" ]; then
            mv "$HOME/.bashrc" "$HOME/.bashrc.bak"
        fi
        
        # Clone dotfiles repository
        git clone git@github.com:savco2000/dotfiles.git $DOTFILES_DIR
        
        # Clean up
        rm -rf "$TARGET_DIR"
        echo "Cleanup complete!"
        ```
      - Activate symlinks
        ```bash
        cd $HOME/dotfiles
        stow .
        ```
    
## Phase 3: The Virtual Lab

### 1. Create the Virtual Machines

#### 1. Create the `dev-vm` virtual machine

  - Run `create-dev-user-data.sh` to generate `dev-user-data.yaml` at `$HOME/install_scripts/vm/`

    ```bash
    $HOME/install_scripts/vm/./create-dev-user-data.sh
    ```
  - Run the following command to create the `dev-vm` virtual machine

    ```bash
    sudo $HOME/install_scripts/vm/./deploy-vm.sh $HOME/install_scripts/vm/dev-user-data.yaml -m 4096 -c 4 -s 40 -f
    ```

  - SSH into `dev-vm`

    ```bash
    ssh dev-vm
    ```

  - Verify installation

    - Check Docker Permissions:

      ```bash
      docker ps
      ```
    - Check Git Identity:

      ```bash
      git config --global -l
      ```
    - Automate the launching and exiting of DevContainers

      - Manually connect to the DevContainers using Visual Studio Code so that Visual Studio Code generates the DevContainer URIs.

        ```bash
        virsh start dev-vm
        ```
        - In Visual Studio Code, click on the "Remote Explorer" icon and click on "Connect in Current Window". 
        - Once the remote repostory opens, it will ask you to "Reopen in DevContainer", select "Yes". Close the remote rository by clicking on "File" > "Close Remote Connection"

      - Extract DevContainer URIs from Visual Studio Code's storage database by running this `get-vs-code-uris.sh` script.

        ```bash
        $HOME/install_scripts/vm/./get-vs-code-uris.sh
        ```
      - Edit this entry to your `~/.bashrc` file with the retrieved HEX_STRINGS.

        ```bash
        # --- TABIRI CONTAINER ENGINE ---
        _dev_tabiri() {
            if ! virsh domstate dev-vm 2>/dev/null | grep -q "running"; then
                echo "🔌 dev-vm is powered off. Booting KVM domain..."
                virsh start dev-vm
            fi

            echo -n "⏳ Waiting for network and SSH to wake up"
            until ssh -o ConnectTimeout=1 -o BatchMode=yes dev-vm true 2>/dev/null; do
                printf "."
                sleep 1
            done
            echo -e "\n🚀 VM is awake! Launching Tabiri DevContainer..."

            code --folder-uri "vscode-remote://dev-container+NEW_HEX_STRING_HERE@ssh-remote+dev-vm/workspaces/tabiri-website"
        }
        alias dev-tabiri=_dev_tabiri
        ```
    - To disable **Workspace Restoration** in Visual Studio Code:
        - Open the Command Palette (`Ctrl + Shift + P`).
        - Type and select: `Preferences: Open User Settings (JSON)`.
        - Add the following line to your settings object (make sure to include a comma on the preceding line if needed):

          ```json
          "window.restoreWindows": "none"
          ```

#### 2. Create the `openclaw-vm` virtual machine

  - Run `create-openclaw-user-data.sh` to generate `openclaw-user-data.yaml` at `$HOME/install_scripts/vm/`

    ```bash
    $HOME/install_scripts/vm/./create-openclaw-user-data.sh
    ```
  - Run the following command to create the `dev-vm` virtual machine

    ```bash
    sudo $HOME/install_scripts/vm/./deploy-vm.sh $HOME/install_scripts/vm/openclaw-user-data.yaml -m 4096 -c 4 -s 40 -f
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
  ssh dev-vm "cloud-init status"
  ```

## Phase 4: Managing Host and Virtual Machines

### 1. Updating Host's Firmware

Update host's firmware once a month.

```bash
fwupdmgr update
```

### 2. Listing VMs

To see what is running or what has been created on your host, use the `list` command.

- List only ACTIVE (running) VMs

  ```bash
  virsh list
  ```

- List ALL VMs (running, paused, or stopped)

  ```bash
  virsh list --all
  ```

### 3. Starting a VM

To power on a virtual machine that is currently shut down:

```bash
virsh start <vm-name>
# Example: virsh start dev-vm
```

### 4. Stopping a VM

There are two ways to turn off a VM, depending on whether you want a polite request or an instant pull of the virtual power cord.

- Graceful Shutdown (Polite Request)

  This sends an ACPI shutdown signal to the guest OS, letting Ubuntu close its processes cleanly.

  ```bash
  virsh shutdown <vm-name>
  # Example: virsh shutdown openclaw-vm
  ```
- Forceful Stop (The Virtual Power Cord)

  If a VM is frozen or if you are about to run your destroy script, this instantly kills the power to the virtual instance.

  ```bash
  virsh destroy <vm-name>
  # Example: virsh destroy dev-vm
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
