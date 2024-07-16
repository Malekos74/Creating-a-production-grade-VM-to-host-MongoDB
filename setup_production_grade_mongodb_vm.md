# Task 298: Creation of a production grade VM that hosts a MongoDB
## This task is part of the EDL project

## I. Creation of the VM and installing MongoDB on it

The creation of the VM was one of the easiest steps. To start, open the Web UI of Proxmox and connect to either the ENI1 or ENI2 client[^1]. After doing that, one can choose to host a VM on `ENI1 (TUEIEWK-LSSIM01)` or `ENI2 (TUEIEWK-LSSIM02)`.

The current VM used for the database is hosted on `ENI2 (TUEIEWK-LSSIM02)`. It is the VM `374 edl-db` and it's hardware is:
- 2 CPU's
- 2 GiB of RAM
- 100 GiB of disk space

The hardware settings can be adjusted as see fit but remember to restart the VM[^2] afterwards. The OS is `ubuntu-24.04-live-server-amd64`[^3] and has no GUI. 

Once the VM is running, you can open the VM console where you will be greeted by a login screen. For safety reasons the username and password cannot be shared here. However if you want to gain access to the VM, you can ask one of the admins to help you with saving an SSH key[^4] for you. Make sure that the keyboard layout matches your current one by changing it on the top right of the login screen.

After that, MongoDB[^5], Docker and Docker-Compose[^6] were installed on the VM.


## II. Installing `QEMU guest agent` on the VM
- **What is QEMU guest agent?**
The QEMU guest agent is a daemon that runs on the virtual machine and passes information to the host about the virtual machine, users, file systems, and secondary networks.
- **Installation**
Connect to the VM using the Proxmox UI or using any commandline. Once connected, run the following command to see infos about the running operating system
```Powershell
cat /etc/os-release
```

1. Start by downloading or refreshing the local packages using:
    ```shell
    sudo apt update
    ```
2. Install the QEMU guest agent:
    ```shell
    sudo apt install qemu-guest-agent
    ```
3. Enable it on system start:
    ```shell
    sudo systemctl enable qemu-guest-agent
    ```
4. Start QEMU guest agent:
    ```shell
    sudo systemctl start qemu-guest-agent
    ```
5. Check status to see if everything worked correctly:
    ```shell
    sudo systemctl status qemu-guest-agent
    ```

More documentation [here](https://docs.openshift.com/container-platform/4.8/virt/virtual_machines/virt-installing-qemu-guest-agent.html#:~:text=The%20QEMU%20guest%20agent%20is,file%20systems%2C%20and%20secondary%20networks.).

## III. Setting up an IP
### 1. Fresh setup
When freshly setting up the VM, the OS will automatically prompt you to change the IP settings if you desire to.
This will take place on the `Network configuration` screen.
Move the cursor to the following line and press enter
```
    [ ens18     eth  -              ▶]       
```
After that, move to `Edit IPv4` and confirm your choice. Here select the `IPv4 Method: Manual`. A new window will appear where you will need to fill in Subnet, Address, Gateway and Name Servers field. For that one should keep the following table in mind:

| Field        | Value(s)                           | Comment                                                                                                               |
| ------------ | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Subnet       | `x.x.x.0/24`                   | This means that the next IP address you specify will be in the range of `x.x.x.0` to `x.x.x.255` (exclusive). |
| Address      | IP Range | This is the static IP address the VM will get. Please make sure that it is not used before assigning[^7].             |
| Gateway      | `x.x.x.254`                    | Default route of the router on the subnetwork.                                                                        |
| Name servers |`10.156.33.53` AND/OR `129.187.5.1` | For routing on the global internet.                                                                                   |

### 2. Cloned VM
If for some reason you decide to setup a new database by Cloning the current VM, please make sure to change the IP address as soon as the cloning is done to avoid conflicts.
This is a step by step tutorial for that:

1. **Open the console:**
    Open the console on the Proxmox webinterface

2. **Set a new IPv4 Address:**
    Use the following command to set a new IPv4 Address:
    ```shell
    sudo nmcli connection modify "Wired connection 1" ipv4.addresses x.x.x.[...]/24
    ```
    Replace the `[...]` with a number between 1 and 254 (Make sure it is unused)

3. **Modify the IPv4 method:**
    Use the following command to modify the IPv4 method from DHCP to manual:
    ```shell
    sudo nmcli connection modify "Wired connection 1" ipv4.method manual
    ```
4. **Reboot the VM**


## IV. Setting up an SSH-Key

- **What is an SSH Key?**
An SSH-key is a cryptographic key used for authenticating and securing network connections between systems. It is commonly used to securely connect to a remote server without needing to provide a password every time. SSH keys come in pairs: a private key, which is kept secure on your local machine, and a public key, which is placed on the server you wish to access. When you try to connect to the server, it uses the public key to verify the corresponding private key.

### 1. Generating an SSH-Key Pair

To generate a new SSH-key pair, follow these steps:

1. **Open a Terminal:**

   On Linux or macOS, you can open a terminal window. On Windows, you can use Git Bash or PowerShell.

2. **Generate the Key Pair:**

   Run the following command:

   ```sh
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
   ```
    - `-t rsa` specifies the type of key to create, in this case, RSA.
    - `-b 4096` specifies the number of bits in the key, 4096 bits is a good security choice.
    - `-C "your_email@example.com"` adds a label to the key with your email address.

3. **Follow the Prompts:**

    You will be asked where to save the new SSH key. The default path is usually fine:
    ```sh
    Enter file in which to save the key (/home/you/.ssh/id_rsa):
    ```
    Next, you will be prompted to enter a passphrase. A passphrase adds an extra layer of security, but you can press Enter to leave it empty if you prefer not to use one:
    ```sh
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
    ```

4. **Completion Message:**

    After you enter the passphrase and if everything worked correctly, you will see a message confirming that your SSH key pair has been generated, along with the key fingerprint and a randomart image.

### 2. Adding an SSH-Key to the VM

To do this, you either would have generated an SSH-key pair as discussed before, or got an SSH-key from someone. Please make sure that the key that you are using from now onwards is the public key. If you ever use the private key for this step, the whole key pair should be scrapped and regenerated.
After having acquired a public key, please follow these steps:

1. **Copy the public key**

    If you generated the SSH-key pair as discussed earlier, you will find the public key under the path that you chose saved as `<key_file_name>.pub`. Per default, the `<key_file_name>` is `id_rsa` if you used RSA encryption.
    
    All you need to do in this step is open this file with a text editor, and copy it's contents.

2. **Open a Terminal:**

   On Linux or macOS, you can open a terminal window. On Windows, you can use Git Bash or PowerShell.

3. **SSH into the VM**

   Run the following command in order to SSH into the VM:
   ```sh
   ssh <username>@<ip_address>
   ```
   There are two scenarios now:
   - Either you will need to enter the password.
   - Or you have a key already saved on the VM, which will grant you direct access without the need of inputting a password.

   If none of the cases apply to you, contact an admin to get access.

4. **Navigate to Proper File:**

    Run the following command to navigate to the `.ssh` file
    ```sh
    cd .ssh
    ```
    After that open the `authorized_keys` file using any text editor of your choice
    ```sh
    nano authorized_keys
    ```

5. **Paste the Key:**

    Now that you are in the right text file, you will need to add two new lines

    The first one is the 
    ```sh
    # First_name Last_name
    ```
    For the second one, paste the public key in. Save and exit.


6. **Test:**

    Now that you saved your key on the VM, retry to SSH into it. If it works without you having to input a password, then everything worked as intended

---

[^1]: Available online under [ENI1 Proxmox](https://10.152.14.63:8006) and [ENI2 Proxmox](https://10.152.14.64:8006). Both have recently been merged, so either link will show the same interface.
[^2]: This is done by right clicking on the VM on the left taskbar and selecting `Reboot`.
[^3]: Official download link for Ubuntu 24.04 LTS [here](https://ubuntu.com/download/server).
[^4]: This will be explained further in section [IV].
[^5]: Official documentation for installing MongoDB on an Ubuntu system [here](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/).
[^6]: Official documentation for installing Docker on an Ubuntu system [here](https://docs.docker.com/engine/install/ubuntu/).
[^7]: [Here](https://gitlab.lrz.de/edge/infrastructure/-/blob/main/docs/general/server-infra.md) you can check which IP addresses are used and which are free. This link may not be up to date with the current state of the ENI server.
[^10]: ENI1 IP-Adress: `10.152.14.63`, ENI2 IP-Adress: `10.152.14.64`.


