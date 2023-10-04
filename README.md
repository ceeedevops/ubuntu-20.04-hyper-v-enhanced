
# Setup Hyper-V enhanced session for Ubuntu 20

I couldn't find instructions that were 100% complete, so I put this together.

These instructions worked fine for me.  Follow each step carefully.

## Download Ubuntu 20 desktop

**DO NOT create the VM by choosing Quick Create in Hyper-V Manager. Follow these instructions exactly.**

Download Ubuntu Desktop 20.04 LTS from [here](https://ubuntu.com/#download) - later 20.xx versions will likely work as well.

Direct link as of this writing is [here](https://mirrors.cat.pdx.edu/ubuntu-releases/20.04.2.0/ubuntu-20.04.2.0-desktop-amd64.iso).

## Set up Hyper-V VM

### New > Virtual Machine

Use all defaults aside from:

* Name: Ubuntu 20.04 LTS
* Generation: 2
* Startup memory: 4096
* Connection: Default Switch
* Install from bootable image file: (use iso downloaded above)
* OK

### Set Ubuntu compatible security template for Secure Boot

* Settings > Security
* Change Template to "Microsoft UEFI Certificate Authority"
* OK

### Start VM

* Connect, Start

## Install Ubuntu in VM

Use all defaults.

Do **not** choose "Log in Automatically"

## Install and run linux-vm-tool script

```bash
sudo apt-get update

sudo apt-get install --yes git

git clone https://github.com/Hinara/linux-vm-tools.git

cd linux-vm-tools/ubuntu/20.04

chmod +x ./install.sh

sudo ./install.sh

sudo shutdown -r now
```

For reference, the contents of the install.sh script (as of commit 19ae5b9) is:

```shell
#!/bin/bash

#
# This script is for Ubuntu 20.04 Focal Fossa to download and install XRDP+XORGXRDP via
# source.
#
# Major thanks to: http://c-nergy.be/blog/?p=11336 for the tips.
#

###############################################################################
# Use HWE kernel packages
#
HWE=""
#HWE="-hwe-20.04"

###############################################################################
# Update our machine to the latest code if we need to.
#

if [ "$(id -u)" -ne 0 ]; then
    echo 'This script must be run with root privileges' >&2
    exit 1
fi

apt update && apt upgrade -y

if [ -f /var/run/reboot-required ]; then
    echo "A reboot is required in order to proceed with the install." >&2
    echo "Please reboot and re-run this script to finish the install." >&2
    exit 1
fi

###############################################################################
# XRDP
#

# Install hv_kvp utils
apt install -y linux-tools-virtual${HWE}
apt install -y linux-cloud-tools-virtual${HWE}

# Install the xrdp service so we have the auto start behavior
apt install -y xrdp

systemctl stop xrdp
systemctl stop xrdp-sesman

# Configure the installed XRDP ini files.
# use vsock transport.
sed -i_orig -e 's/port=3389/port=vsock:\/\/-1:3389/g' /etc/xrdp/xrdp.ini
# use rdp security.
sed -i_orig -e 's/security_layer=negotiate/security_layer=rdp/g' /etc/xrdp/xrdp.ini
# remove encryption validation.
sed -i_orig -e 's/crypt_level=high/crypt_level=none/g' /etc/xrdp/xrdp.ini
# disable bitmap compression since its local its much faster
sed -i_orig -e 's/bitmap_compression=true/bitmap_compression=false/g' /etc/xrdp/xrdp.ini

# Add script to setup the ubuntu session properly
if [ ! -e /etc/xrdp/startubuntu.sh ]; then
cat >> /etc/xrdp/startubuntu.sh << EOF
#!/bin/sh
export GNOME_SHELL_SESSION_MODE=ubuntu
export XDG_CURRENT_DESKTOP=ubuntu:GNOME
exec /etc/xrdp/startwm.sh
EOF
chmod a+x /etc/xrdp/startubuntu.sh
fi

# use the script to setup the ubuntu session
sed -i_orig -e 's/startwm/startubuntu/g' /etc/xrdp/sesman.ini

# rename the redirected drives to 'shared-drives'
sed -i -e 's/FuseMountName=thinclient_drives/FuseMountName=shared-drives/g' /etc/xrdp/sesman.ini

# Changed the allowed_users
sed -i_orig -e 's/allowed_users=console/allowed_users=anybody/g' /etc/X11/Xwrapper.config

# Blacklist the vmw module
if [ ! -e /etc/modprobe.d/blacklist-vmw_vsock_vmci_transport.conf ]; then
  echo "blacklist vmw_vsock_vmci_transport" > /etc/modprobe.d/blacklist-vmw_vsock_vmci_transport.conf
fi

#Ensure hv_sock gets loaded
if [ ! -e /etc/modules-load.d/hv_sock.conf ]; then
  echo "hv_sock" > /etc/modules-load.d/hv_sock.conf
fi

# Configure the policy xrdp session
cat > /etc/polkit-1/localauthority/50-local.d/45-allow-colord.pkla <<EOF
[Allow Colord all Users]
Identity=unix-user:*
Action=org.freedesktop.color-manager.create-device;org.freedesktop.color-manager.create-profile;org.freedesktop.color-manager.delete-device;org.freedesktop.color-manager.delete-profile;org.freedesktop.color-manager.modify-device;org.freedesktop.color-manager.modify-profile
ResultAny=no
ResultInactive=no
ResultActive=yes
EOF

# reconfigure the service
systemctl daemon-reload
systemctl start xrdp

#
# End XRDP
###############################################################################

echo "Install is complete."
echo "Reboot your machine to begin using XRDP."
```

### Start VM (again!)

* Connect, Start

## Run the script again

Yes, you must runt the script again after rebooting. This is a step that many online instructions missed.

**Note:** This step takes a while to complete.

```bash
cd linux-vm-tools/ubuntu/20.04

sudo ./install.sh

sudo shutdown -h now
```

## Set Hyper-V transport type

In PowerShell, run the following:

(note that if you named your VM something different, replace the VMName parameter accordingly.  Use `Get-VM` to see the name of all VMs)

```PowerShell
Set-VM -VMName 'Ubuntu 20.04 LTS' -EnhancedSessionTransportType HvSocket
```

start and connect to the VM.  You should now see a prompt for resolution when the VM starts up.

You now have enabled enhanced session, which allows for copy and paste between host and VM etc.

## References

[Setup Hyper-V enhanced session for Ubuntu 20](https://gist.github.com/milnak/54e662f88fa47a5d3a317edb712f957e#setup-hyper-v-enhanced-session-for-ubuntu-20)

[How to install Ubuntu 20.04 on Hyper-V with enhanced session](https://francescotonini.medium.com/how-to-install-ubuntu-20-04-on-hyper-v-with-enhanced-session-b20a269a5fa7)

[Ubuntu 20.04 on Hyper-V](https://medium.com/@labappengineering/ubuntu-20-04-on-hyper-v-8888fe3ced64)
