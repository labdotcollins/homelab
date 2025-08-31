# Proxmox Repos

After installing Proxmox, it defaults to using enterprise repositories. These are intended for users with a valid subscriptions. This is how you switch to the "No-Subscription" repositories:

Open the Proxmox Web Interface and log into your Proxmox Web UI. In the left-hand sidebar, select your Proxmox node.

In the left-hand menu, Underneath Updates you should see a Repositories option. You’ll see a list of repositories. Look for and disable the following:

https://enterprise.proxmox.com/debian/ceph-quincy

https://enterprise.proxmox.com/debian/pve

After disabling those, click the Add button at the top. In the dialog that appears, set the Repository option to No-Subscription. Confirm to add the new repository.

Now navigate to Updates in the left-hand menu. Click Refresh to update the repository list. Click Upgrade to install the latest updates from the new repository.

# GPU Passthrough

If you plan on using GPU passthrough for any of the services on your machine, be sure to enable it. I'm using an i5-12600H, and from my research, it's worth noting that Alder Lake CPUs can introduce some complications in Proxmox. The hybrid architecture, combining hyperthreaded P-cores with non-hyperthreaded E-cores, has been known to cause issues with scheduling, virtualization, and IOMMU grouping, particularly when setting up GPU passthrough. That said, I haven’t personally encountered any of these problems. In fact, everything has worked flawlessly, and configuring GPU passthrough was a breeze.

In order to enable GPU Passthrough, you'll want to start on the CLI of your Proxmox node and using:

nano /etc/default/grub

Locate the following line:

GRUB_CMDLINE_LINUX_DEFAULT="quiet"

And change it to:

GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"

After saving the file, run the following commands to apply the changes:

update-grub

update-initramfs -u

and Reboot your system.

We ran update-initramfs -u just to be safe. I haven’t confirmed through my own testing whether it’s strictly necessary, but updating it ensures we avoid any potential issues with modules that load early during boot.

Once the system is back up, verify that IOMMU is enabled with:

dmesg | grep -e DMAR -e IOMMU

This command will confirm IOMMU is active and can also reveal the raw device IDs of your PCI devices.

I should mention some Motherboards you might have to go into the BIOS to enable IOMMU

# ZFS

The first thing you should decide when setting up your home server is which RAID configuration best fits your needs. In my case, cost was the biggest factor, so I chose RAID 5. I used four 3TB drives, and with RAID 5, one of those drives is reserved for redundancy. This gives me a total of 9TB of usable storage, 

HOWEVER,

One thing I learned too late about RAID 5, or RAIDZ, which is Proxmox's equivalent. It turns out RAIDZ1 is generally not recommended for drives larger than 2TB, and I had four 3TB drives in my pool. This is because in the event of drive failure, the resilvering process takes much longer on a larger drive. During that time, the chance of encountering an issue increases significantly, which would then cause the rebuild to fail and result in data loss. In hindsight, it would have been much wiser to go with RAIDZ2, which provides an extra layer of redundancy by allowing two drives to fail before the pool is at risk.

Creating a ZFS pool is straightforward. First, open the Proxmox Web Interface and log in. In the left-hand sidebar, select your Proxmox node.

Near the bottom of the left-hand menu, under the Disks category, you’ll find ZFS, click on it, then select Create: ZFS.

From there, choose the drives you want to include, give your pool a name, and select the RAID level. For my setup, I selected all four disks and set the RAID level to RAIDZ.

## Network Share

With your ZFS pool set up, you can mount and use it however you'd like. I chose to containerize a network share and mount it as needed in other VMs or containers that need access to it.

To start, I selected the storage pool from the left-hand sidebar where I wanted to store the image. in this case, the internal M.2 drive of my node. 

From the storage volume you selected, go to the left-hand menu and click CT Templates, then click Templates. The choice of container OS is up to you. I had previously run an SMB share on Ubuntu, so I stuck with that. When it comes to Ubuntu, I've heard Ubuntu 22.04 tends to work better with Proxmox, so that’s what I went with.

Once the template is downloaded, click Create CT at the top of the interface. Here, you’ll configure the basic settings: choose the node, assign a container ID, set a hostname, and create a password. I named mine share and matched the container ID to the last octet of its IP address for easier organization.

On the next screen, select the Ubuntu template you just downloaded.

Under the Disks section, I chose to use the internal M.2 storage, which I go into more detail about in the next section. I allocated 32GB of space to the container.

For CPU and Memory, I assigned 2 cores, 2048MB of RAM, and 1028MB of swap.

Since I handle all DHCP and DNS settings through my router, I left the network and DNS options in Proxmox at their defaults.

Now, from the container we just created, navigate to the Resources tab and click Add > Mount Point.

In the Create: Mount Point menu, select your ZFS pool on the hard drive array, allocate 8TB of storage, and set the mount path to /data. The mount point is up to you, though; this is just what I went with.

Once in the console of the container, you'll want to create a new user:

adduser [your-user]
adduser [your-user] sudo

Switch to the new user:
su [your-user]

Set ownership of the mount point (replace /data with your actual mount location):
sudo chown -R [your-user]:[your-user] /data

Install Samba:
sudo apt install samba -y

Back up the default Samba config:
cd /etc/samba
sudo mv smb.conf smb.conf.backup

Create a new Samba config, this is a simple one:
sudo nano smb.conf

```
[global]
   server string = [network-share-name]
   workgroup = WORKGROUP
   security = user
   map to guest = Bad User
   name resolve order = bcast host
   hosts allow = [your-local-subnet]/0
   hosts deny = 0.0.0.0/0
[data]
   path = /data
   force user = [your-user]
   force group = [your-user]
   create mask = 0774
   force create mode = 0774
   directory mask = 0775
   force directory mode = 0775
   browseable = yes
   writable = yes
   read only = no
   guest ok = no
```

Set a Samba password for your user:
sudo smbpasswd -a [your-user]

Enable and start the Samba services:
sudo systemctl enable smbd
sudo systemctl enable nmbd
sudo systemctl restart smbd
sudo systemctl restart nmbd

To make the share discoverable by Windows systems, install WSDD:
sudo apt install wsdd -y

## Mounting the Network Share

# Containers vs VMs

I recommend running containers and VMs on a faster drive like an SSD or NVMe for better performance, while keeping the bulk of your data on the hard drive-based ZFS pool.

# Backups

## Cron
