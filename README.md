# Private Cloud Runbook

## 1. Setup Host Machine

- Pre-install steps
  - Run this command to make sure you’ve enabled virtualization in on your computer.

    ```bash
    egrep -c '(vmx|svm)' /proc/cpuinfo
    ```

    The result should be above `0`.

- Create and run `create_dev_host.sh` script below:

    ```bash
    #!/usr/bin/env bash
    # Make executable: chmod +x create_dev_host.sh
    # Run: sudo ./create_dev_host.sh

    # Unofficial Bash Strict Mode
    set -o errexit
    set -o nounset
    set -o pipefail

    if [[ "${TRACE-0}" == "1" ]]; then
        set -o xtrace
    fi

    # Help Menu
    if [[ "${1-}" =~ ^-*h(elp)?$ ]]; then
        echo 'Usage: sudo ./create_dev_host.sh'
        echo 'Bootstraps a vanilla Ubuntu 26.04 environment with development tools.'
        exit
    fi

    # Ensure script is run with sudo
    if [[ $EUID -ne 0 ]]; then
    echo "This script must be run as root. Please use sudo." 
    exit 1
    fi

    # Ensure SUDO_USER is set so we don't break user-specific commands
    if [[ -z "${SUDO_USER:-}" ]]; then
        echo "Could not detect SUDO_USER. Please run the script using sudo from a standard user account."
        exit 1
    fi

    cd "$(dirname "$0")"

    main() {
        echo -e "\n=== Removing Firefox Snap ==="
        snap remove --purge firefox

        echo -e "\n=== Updating System Packages ==="
        apt-get update
        apt-get upgrade -y

        echo -e "\n=== Installing QEMU and Virtual Machine Manager ==="
        add-apt-repository -y universe

        apt-get install -y qemu-system-x86 \
            qemu-utils \
            libvirt-daemon-system \
            libvirt-clients \
            virt-manager 

        adduser "$SUDO_USER" libvirt
        adduser "$SUDO_USER" kvm

        echo -e "\n=== Installing Startup Disk Creator ==="
        apt-get install -y usb-creator-gtk

        echo -e "\n=== Installing Google Chrome ==="
        wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb -O /tmp/google-chrome.deb
        
        apt-get install -y /tmp/google-chrome.deb
        rm /tmp/google-chrome.deb

        echo -e "\n=== Installing Visual Studio Code ==="
        snap install code --classic

        echo -e "\n=== Installing VS Code extensions for user: $SUDO_USER ==="
        sudo -u "$SUDO_USER" code --install-extension davidanson.vscode-markdownlint

        echo -e "\n=== Installing Slack ==="        
        snap install slack --classic

        echo -e "\n=== Installing GIMP ==="
        apt-get install -y gimp    

        echo -e "\n=== Script execution complete! ==="

        read -p "A system reboot is recommended. Reboot now? (y/N) " -n 1 -r
        echo
        if [[ $REPLY =~ ^[Yy]$ ]]; then
            reboot
        else
            echo "Please remember to reboot your system later."
        fi
    }

    main "$@"
    ```

- Post-install steps
  
  - Verify that Libvirtd service is started (optional)

    ```bash
    sudo systemctl status libvirtd.service
    ```

  - Check virsh status (optional)

    ```bash
    sudo virsh net-list --all
    ```

## 2. Create Base Virtual Machine (VM)

1. Create VM in QEMU
    - File > New Virtual Machine
       - Local install media (ISO image or CDROM)
    - VM Name: `tabiri_base`
    - Memory size: `4096MB`
    - CPUs: 2
    - Hard disk size: `50GB`
2. Install Ubuntu on VM
    - Your name: `Tabiri Analytics`
    - Your computer's name: `dev-vm`
    - Pick a username: `tabiri`
3. Make VM fit the entire screen
    - `View` > `Scale Display` > `Auto resizeVM with window`
4. Update the VM
    - Fetch and install the latest version of the package list from Ubuntu's software repository, and any third-party repositories that may have been configured.

        ```bash
        sudo apt update && sudo apt upgrade -y
        ```

    - Remove unused dependencies.

        ```bash
        sudo apt autoremove -y
        ```

    - Update snap store.

        ```bash
        # sudo killall snap-store
        sudo snap refresh
        ```

## 3. Create Dev Base VM

- Pre-install steps
  - Clone the `tabiri_base` VM created in `Step #2`
  - Rename the VM to `tabiri_dev_base`
  - `View` > `Scale Display` > `Auto resizeVM with window`
  - Download `customize_development_vm.tar.xz` from Tabiri's Google Drive to `~/Downloads/` folder

- Create and run `create_tabiri_dev_base_vm.sh` script below:

    ```bash
    #!/usr/bin/env bash
    # chmod +x ~/Downloads/create_tabiri_dev_base_vm.sh
    # Run: sudo ~/Downloads/./create_tabiri_dev_base_vm.sh
    set -euo pipefail
    
    # Enable debugging if TRACE is set
    [[ "${TRACE-0}" == "1" ]] && set -x
    
    # Function to print usage
    usage() {
        cat <<EOF
    Usage: ${0##*/} [OPTIONS]
    
    This is an awesome bash script to make your life better.
    
    Options:
    -h, --help     Show this help message and exit
    EOF
    }
    
    # Check if help is requested
    if [[ "${1-}" =~ ^-*h(elp)?$ ]]; then
        usage
        exit 0
    fi
    
    cd "$(dirname "$0")"
    
    # Function to handle each installation step
    run_command() {
        local message="$1"
        shift
        echo
        echo "$message"
        echo
        "$@"
    }
    
    main() {
        run_command "Updating guest VM name..." hostnamectl set-hostname dev-guest
    
        run_command "Removing Firefox..." snap remove --purge firefox
    
        run_command "Updating snap store..." snap refresh
    
        run_command "Updating the apt package index..." apt-get update
    
        run_command "Upgrading packages..." apt-get upgrade -y
    
        run_command "Downloading Google Chrome..." wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb -P /tmp/ && \
            apt-get install -y /tmp/google-chrome-stable_current_amd64.deb
    
        run_command "Downloading Slack..." wget https://downloads.slack-edge.com/releases/linux/4.37.94/prod/x64/slack-desktop-4.37.94-amd64.deb -P /tmp/ && \
            apt-get install -y /tmp/slack-desktop-*.deb
    
        run_command "Installing curl..." apt-get install -y curl
        # curl --version
    
        run_command "Installing AWS CLI..." \
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip" && \
            unzip /tmp/awscliv2.zip -d /tmp && \
            /tmp/aws/install
    
        run_command "Creating folder for git projects..." sudo -u "$SUDO_USER" mkdir -p "/home/$SUDO_USER/code"
    
        run_command "Installing .NET 8.0 SDK..." apt-get install -y dotnet-sdk-8.0
        # dotnet --version

        run_command "Installing PostgreSQL..." \
            sh -c "echo 'deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main' > /etc/apt/sources.list.d/pgdg.list" && \
            wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && \
            apt-get update && \
            apt-get install -y postgresql postgresql-client
        # psql --version
        # https://www.postgresql.org/download/linux/ubuntu/
    
        run_command "Installing pgAdmin..." \
            curl https://www.pgadmin.org/static/packages_pgadmin_org.pub | apt-key add && \
            sh -c "echo 'deb https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main' > /etc/apt/sources.list.d/pgadmin4.list" && \
            apt-get update && \
            apt-get install -y pgadmin4
    
        run_command "Installing VS Code..." \
            apt-get install -y wget gpg && \
            wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg && \
            install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg && \
            sh -c "echo 'deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main' > /etc/apt/sources.list.d/vscode.list" && \
            rm -f packages.microsoft.gpg && \
            apt-get install -y apt-transport-https && \
            apt-get update && \
            apt-get install -y code
    
        local extensions=(
            ms-dotnettools.csharp
            ms-ossdata.vscode-pgsql
            amazonwebservices.aws-toolkit-vscode
            ms-azuretools.vscode-docker
            formulahendry.auto-rename-tag
            thekalinga.bootstrap4-vscode
            pranaygp.vscode-css-peek
            davidanson.vscode-markdownlint
            ritwickdey.LiveServer
            bradlc.vscode-tailwindcss
            GitHub.copilot
        )
    
        for ext in "${extensions[@]}"; do
            run_command "Installing VS Code extension $ext..." sudo -u "$SUDO_USER" code --install-extension "$ext"
        done
    
        run_command "Installing Git..." apt-get install -y git
        # git --version
    
        run_command "Installing Docker..." \
            apt-get install -y ca-certificates curl gnupg lsb-release && \
            mkdir -p /etc/apt/keyrings && \
            curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
            sh -c "echo 'deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable' > /etc/apt/sources.list.d/docker.list" && \
            apt-get update && \
            apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
        # docker --version
    
        run_command "Installing Terraform and Packer..." \
            apt-get install -y gnupg software-properties-common && \
            wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg && \
            sh -c "echo 'deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main' > /etc/apt/sources.list.d/hashicorp.list" && \
            apt-get update && \
            apt-get install -y terraform packer
        # terraform --version
        # packer --version
    
        run_command "Updating the apt package index..." apt-get update
    
        run_command "Installing the newest versions of available packages..." apt-get upgrade -y
    
        run_command "Removing packages that are no longer required..." apt-get autoremove -y
    
        run_command "Deleting installation files..." rm -r /tmp/*
    
        echo "The script has been executed successfully."
        echo "Rebooting system..."
        reboot
    }
    
    main "$@"
    ```

- Post-Install Steps
  - Install latest NodeJS LTS version (`source` doesn't work in non-interactive shell in Ubuntu)

    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
    source ~/.bashrc
    nvm install --lts
    #node --version
    ```
  
  - Unpin the following programs to the Dock:
    - App Center
    - Help

  - Pin the following programs to the Dock:
    - Google Chrome
    - pgAdmin4
    - Visual Studio Code
    - Terminal
    - Slack

## 4. Create Base AWS Dev VM

- Pre-install steps
  - Clone the `tabiri_dev_base` VM created in `Step #3`
  - Rename the VM to `tabiri_aws`
  - `View` > `Scale Display` > `Auto resizeVM with window`

- Create and run `create_tabiri_aws_dev_vm.sh` script below:

    ```bash
    #!/usr/bin/env bash

    # Give this script execute permission before executing it by running the chmod in the script's directory
    # chmod +x ~/Downloads/create_tabiri_aws_dev_vm.sh
    # Run: sudo ~/Downloads/./create_tabiri_aws_dev_vm.sh

    set -euo pipefail

    # Enable debugging if TRACE is set
    [[ "${TRACE-0}" == "1" ]] && set -x

    # Function to print usage
    usage() {
        cat <<EOF
    Usage: ${0##*/} [OPTIONS]

    This is an awesome bash script to make your life better.

    Options:
    -h, --help     Show this help message and exit
    EOF
    }

    # Check if help is requested
    if [[ "${1-}" =~ ^-*h(elp)?$ ]]; then
        usage
        exit 0
    fi

    cd "$(dirname "$0")"

    # Function to handle each installation step
    run_command() {
        local message="$1"
        shift
        echo
        echo "$message"
        echo
        "$@"
    }

    main() {
        run_command "Updating snap store..." snap refresh

        run_command "Extracting contents of customize_development_vm.tar.xz..." \
            tar -xf "/home/$SUDO_USER/Downloads/customize_development_vm.tar.xz" -C /tmp/ --strip-components=1

        run_command "Setting up OpenVPN..." \
            mv /tmp/setup_dev_machine/vpn_files/ca.crt /etc/openvpn/ && \
            mv /tmp/setup_dev_machine/vpn_files/aws/* /etc/openvpn/ && \
            chmod 700 /etc/openvpn/skadima003-aws.key && \
            systemctl start openvpn@skadima003-aws.service

        run_command "Setting up and configuring AWS access..." \
            sudo -u "$SUDO_USER" mkdir -p /home/"$SUDO_USER"/.aws && \
            mv /tmp/setup_dev_machine/aws/* /home/"$SUDO_USER"/.aws/

        run_command "Moving the EC2 keys to ~/Tams/Keys/ ..." \
            sudo -u "$SUDO_USER" mkdir -p /home/"$SUDO_USER"/Tams/Keys/ && \
            sudo -u "$SUDO_USER" mv /tmp/setup_dev_machine/tams_keys/*.pem /home/"$SUDO_USER"/Tams/Keys/ && \
            chmod 700 /home/"$SUDO_USER"/Tams/Keys/*.pem

        run_command "Moving tamssettings.json file to /tamssettings.json..." \
            mv /tmp/setup_dev_machine/tamssettings.json /

        run_command "Creating Packer and Terraform environment variables..." \
            mv /tmp/setup_dev_machine/environment /etc/environment && \
            source /etc/environment

        echo "The script has been executed successfully."

        echo "Rebooting system..."
        reboot
    }

    main "$@"
    ```

- Post-Install Steps
    1. Test VPN connection to Tabiri's AWS infrastructure by pinging `TabiriVPN001`

        ```bash
        ping -c 3 192.168.250.12
        ```

    2. Create localhost PostgreSQL database
        - Log into PostgreSQL: `sudo -u postgres psql`
        - Create password for postgres user: `postgres=ALTER USER postgres PASSWORD 'PASSWORD_GOES_HERE';` Get password from `/tamssettings.json` above.
        - Exit PostgreSQL: `postgres=\q`
        - Restart PostgreSQL: `sudo /etc/init.d/postgresql restart`
    3. Configure `pgAdmin`
        - Connect to `local` database
            - Name: `Dev`
            - Connection: `localhost`
            - Port: `5432`
            - Username: `postgres`
            - Password: `Get password from /tamssettings.json`

        - Connect to `Dev` database
            - Name: `Test`
            - Connection: `192.168.251.88`
            - Port: `5432`
            - Username: `postgres`
            - Password: `Get password from /tamssettings.json`

        - Connect to `Prod` database
            - Name: `Prod`
            - Connection: `192.168.251.10`
            - Port: `5432`
            - Username: `postgres`
            - Password: `Get password from /tamssettings.json`
    4. Configure `AWS CodeCommit` credentials (SSH key)
        - Generate `AWS CodeCommit` public key file:

            ```bash
            ssh-keygen -b 4096 -t rsa -f /home/tabiri/.ssh/codecommit_rsa -q -N ""
            ```

        - View and copy `AWS CodeCommit` public key file by running:

            ```bash
            cat /home/tabiri/.ssh/codecommit_rsa.pub
            ```

        - Sign in to the `AWS Management Console` and open the IAM console at <https://console.aws.amazon.com/iam/home#/users>
        - On the `IAM console`, in the navigation pane, choose `Users`, and from the list of users, choose your `IAM user`
        - On the user details page, choose the `Security Credentials` tab
        - Under `SSH public keys for AWS CodeCommit` choose `Upload SSH public key`
        - Paste the contents of your SSH public key into the field, and then choose `Upload SSH public key`
        - Copy or save the information in `SSH Key ID` (for example, `APKAEIBAERJR2EXAMPLE`)
        - Generating `AWS CodeCommit` config file: (Replace `APKAEIBAERJR2EXAMPLE` with the SSH Key ID from above)

            ```bash
            sudo cat << EOF >> /home/tabiri/.ssh/config
            Host git-codecommit.*.amazonaws.com
                User APKAEIBAERJR2EX-webAMPLE
                IdentityFile ~/.ssh/codecommit_rsa
            EOF
            ```

        - Run the following command to test your SSH configuration:

            ```bash
            ssh git-codecommit.us-east-1.amazonaws.com
            ```

        - To avoid "an application wants to access the private key but it is locked" error, add this command:

            ```bash
            ssh-add ~/.ssh/codecommit_rsa
            ```

        - Read full instructions [here](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html#setting-up-ssh-unixes-connect-console)

    5. Clone `.dotfiles` repo
        - Clone the `.dotfiles` repo

            ```bash
            git clone --bare ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/.dotfiles $HOME/.dotfiles
            ```

        - Create an alias `dotfiles` so you don't need to type it all over again

            ```bash
            alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
            ```

        - Set git status to hide untracked files

            ```bash
            dotfiles config --local status.showUntrackedFiles no
            ```

        - Remove the files that are to be replaced

            ```bash
            rm -f README.md \
                .bashrc \
                .gitconfig \
                .gitmessage \
                .config/Code/User/settings.json \
                .aws/config
            ```

        - Checkout the actual content from the git repository to your `$HOME`

            ```bash
            dotfiles checkout
            ```

        - Set local branch to track remote branch

            ```bash
            dotfiles push --set-upstream origin master
            ```

        - Reload `.bashrc` settings

            ```bash
            source ~/.bashrc
            ```

    6. Clone `AWS CodeCommit` repos
        - Clone `Tams.Web` repo

            ```bash
            git clone ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/Tams.Web /home/tabiri/git/Tams.Web
            ```

        - Clone `Tams.Backend` repo

            ```bash
            git clone ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/Tams.Backend /home/tabiri/git/Tams.Backend
            ```

        - Clone `Tams.Tailwind` repo

            ```bash
            git clone ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/Tams.Tailwind /home/tabiri/git/Tams.Tailwind
            ```

        - Clone `DevOps` repo

            ```bash
            git clone ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/DevOps /home/tabiri/git/DevOps
            ```

        - Read full instructions [here](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html#setting-up-ssh-unixes-connect-console)

## 5. Create GCP Dev VM

- Pre-install steps
  - Clone the `tabiri_dev_base` VM created in `Step #3`
  - Rename the VM to `tabiri_gcp`

- Create and run `create_tabiri_gcp_dev_vm.sh` script below:

    ```bash
    #!/usr/bin/env bash
    
    # You can access this bash template here: https://sharats.me/posts/shell-script-best-practices/
    # Give this script execute permission before executing it by running the chmod in the script's directory
    # chmod +x ~/Downloads/create_tabiri_gcp_dev_vm.sh
    # Run: sudo ~/Downloads/./create_tabiri_gcp_dev_vm.sh
    
    # PRE-INSTALL STEPS
    # NONE
    
    set -euo pipefail
    
    # Enable debugging if TRACE is set
    [[ "${TRACE-0}" == "1" ]] && set -x
    
    # Function to print usage
    usage() {
        cat <<EOF
    Usage: ${0##*/} [OPTIONS]
    
    This is an awesome bash script to make your life better.
    
    Options:
      -h, --help     Show this help message and exit
    EOF
    }
    
    # Check if help is requested
    if [[ "${1-}" =~ ^-*h(elp)?$ ]]; then
        usage
        exit 0
    fi
    
    cd "$(dirname "$0")"
    
    # Function to handle each installation step
    run_command() {
        local message="$1"
        shift
        echo
        echo "$message"
        echo
        "$@"
    }
    
    main() {
        run_command "Updating snap store..." snap refresh
    
        run_command "Extracting contents of customize_development_vm.tar.xz..." \
            tar -xf "/home/$SUDO_USER/Downloads/customize_development_vm.tar.xz" -C /tmp/ --strip-components=1
    
        run_command "Setting up OpenVPN..." \
            mv /tmp/setup_dev_machine/vpn_files/ca.crt /etc/openvpn/ && \
            mv /tmp/setup_dev_machine/vpn_files/gcp/* /etc/openvpn/ && \
            chmod 700 /etc/openvpn/skadima002-gcp.key && \
            systemctl start openvpn@skadima002-gcp.service
    
        # Test VPN connection to Tabiri's GCP infrastructure by pinging tabirivpn002
        # ping -c 3 192.168.250.101
    
        echo
        echo "The script has been executed successfully."
    }
    
    main "$@"
    ```

- Post-Install Steps
  1. `View` > `Scale Display` > `Auto resizeVM with window`
  2. Test VPN connection to Tabiri's GCP infrastructure by pinging `tabirivpn002`

        ```bash
        ping -c 3 192.168.250.101
        ```

