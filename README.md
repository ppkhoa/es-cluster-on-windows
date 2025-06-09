# es-cluster-on-windows
Create a fully functioning Lustre cluster using DDN EXAScaler on a Windows machine using VirtualBox for testing purposes.

**DO NOT USE FOR PRODUCTION ENVIRONMENT AS THIS IS AN UNSUPPORTED CONFIGURATION, PROVIDED AS-IS, WITHOUT ANY SUPPORT FROM DDN. DO NOT CONTACT DDN SUPPORT FOR ANY ISSUE RELATED TO THIS CONFIGURATION.**

**I AM NOT RESPONSIBLE FOR ANY DATA LOSS WHILE USING THIS CONFIGURATION.**

## Prerequisites
1. [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
2. \>=16GB of RAM
3. 250GB of space
4. EXAScaler VM image (contact your SE for the image, it is not provided in this repo)
5. Windows 10/11 on x86 CPU (not yet tested with ARM cpu)
6. [qemu-img for Windows](https://cloudbase.it/qemu-img-windows/)

Tested with [this](https://valid.x86.fr/m0du7l) configuration

## Installation
### Create VMs
1. Download and extract qemu-img to your preferred directory.
2. Convert EXAScaler image to VirtualBox image using the following command:
```
qemu-img.exe convert <exa_image>.img -O vdi es1.vdi
```
3. Create 3 copy of `es1.vdi` and name it `es2.vdi`, `client1.vdi`, `client2.vdi` (or anything else to your preference, as long as it is different than the original)
4. Create a new `Virtual Machine` in VirtualBox using the wizard, with

- `Type: Linux`, `Subtype: Red Hat`, `Version: 64-bit`.
- Skip `Unattended Install`.
- In `Hardware`, set `Base Memory` to minimum `4GB` and `2 CPUs`.
- In `Hard Disk`, select `Create a Virtual Hard Disk Now` and select `es1.vdi`, set capacity to `64GB` minimum
- Click `Finished`

5. Repeat step 4, create a second, third and fourth VM using the copies created earlier for the virtual hard disk, in this example, `es2.vdi`, `client1.vdi`, `client2.vdi`. You should have 4 VMs created after this step.

### Configure VMs
1. Select a VM from the list on left hand side, click on `Settings`
2. Go to `Network`, click on `Adapter 2` tab
3. Check the box `Enable Network Adapter`, select `Internal Network` from the dropdown for `Attached to:`, the rest can be left as default.
4. Repeat step 3 for `Adapter 3`

![Adapter 3 Settings](https://ptpimg.me/363cvx.png)

5. Make sure `Adapter 1` is also enabled and is set to `Bridged Adapter` or `NAT`
6. Click `OK` to save.

### Configure Lustre shared disks (MGT, MDTs, OSTs)
1. At VirtualBox main screen, click on `Tools`
2. Click on `Properties` to open `Attributes` panel (if not already opened).
3. Click on `Create`, select the location for the Hard Disk File, name it `es-mdt1.vdi`, with `2GB` in size.
4. Check the box `Pre-allocate Full Size` (must be checked in order to set `Shareable` type)
5. Click `Finish`
6. Repeat the same steps for the remaining shared disks as follows:
- `es-mdt2.vdi`, size `2GB`
- `es-mgt.vdi`, size `128GB`
- `es-ost1.vdi`, size `16GB`
- `es-ost2.vdi`, size `16GB`
7. Select each of the shared disks created, change the type to `Shareable`, similar to this:

![Change type to Shareable](https://ptpimg.me/0rlkhp.png)

By the end of step 6, you should have 9 hard disk files total, similar to the screenshot above.

### Configure server VMs
Each of the VMs will have 3 network interfaces as follows:

`enp0s3` -> Management interface, mainly used for SSH and other management tasks

`enp0s8` -> Data interface, Lustre will communicate through this interface, together with

`enp0s9` -> Data interface, another interface Lustre can use to communicate with the `es*` servers

1. Start server VMs (`es` and `es2` in this guide), one by one (otherwise VirtualBox will return an error)
2. On `es1` and `es2` VM, assign an IP address to management interface `enp3s0` (modify the IP address to your network):

```
ifconfig enp0s3 inet 192.168.56.193/24 up
```

3. Set hostname on both `es*` VMs (each VM must have their own hostname), for example, on `es1` VM:

```
hostnamectl set-hostname es1
```

4. Create DDN device tree on each VM (this step must be done on each VM separately):

```
# Find block devices for the shared disks:
lsblk
# Get Serial number for shared disks:
sg_inq --page=0x80 /dev/sdb
sg_inq --page=0x80 /dev/sdc
sg_inq --page=0x80 /dev/sdd
sg_inq --page=0x80 /dev/sde
sg_inq --page=0x80 /dev/sdf

# Put the Serial number for shared disks in /etc/ddn/device_alias.conf (change the serial number to the output from the commands above), output should look like this:
$ cat /etc/ddn/device_alias.conf

VB9a7c2ff9-2529ff28     mgs
VB86176f17-e467ed64     e633fs_mdt0000_s0
VB81c33a6b-b0a9a8d1     e633fs_mdt0001_s0
VBeeadf9b4-fcd05873     e633fs_ost0000
VBee2c12b8-e5289fe0     e633fs_ost0001
```
5. Create a script called `ddn-dev-name.sh` to create softlink to `/dev/ddn` at `/root`
```
#! /bin/sh
/usr/bin/grep $1 /etc/ddn/device_alias.conf | awk '{print $2}'
```

6. Make the script executable
```
chmod +x /root/ddn-dev-name.sh
```

7. Add the following line to the top of `/etc/udev/rules.d/90-tune-ddn.rules`
```
KERNEL=="sd?",  SUBSYSTEM=="block", ATTRS{model}=="VBOX HARDDISK*", PROGRAM="/root/ddn-dev-name.sh $env{ID_SERIAL_SHORT}", SYMLINK+="ddn/%c", SYMLINK+="ddn/by-dev/%c"
```

8. Reboot the VM, then check the output of `ls -lah /dev/ddn`, should look like this
```
[root@es1 ~]# ls -lah /dev/ddn
total 0
drwxr-xr-x  3 root root  160 Jun  9 19:05 .
drwxr-xr-x 23 root root 3.3K Jun  9 19:05 ..
drwxr-xr-x  2 root root  140 Jun  9 19:05 by-dev
lrwxrwxrwx  1 root root    6 Jun  9 19:05 e633fs_mdt0000_s0 -> ../sdc
lrwxrwxrwx  1 root root    6 Jun  9 19:05 e633fs_mdt0001_s0 -> ../sdd
lrwxrwxrwx  1 root root    6 Jun  9 19:05 e633fs_ost0000 -> ../sde
lrwxrwxrwx  1 root root    6 Jun  9 19:05 e633fs_ost0001 -> ../sdf
lrwxrwxrwx  1 root root    6 Jun  9 19:05 mgs -> ../sdb
```

### EXAScaler configuration

1. Copy the example `exascaler.toml` from this repo and put it in `/etc/ddn/`, modify IP addresses to your network if you are not using the example config.
2. Bootstrap EMF
```
emfctl bootstrap
```

3. Bring up localhost interface to trick ES into thinking this is a SFA setup:
```
ifconfig lo:1 10.0.2.15/32 up
```

4. On `es2` VM, create a route back to `es1` via management interface
```
ip route add 10.0.2.15/32 via <es1_VM_mgmt_ip>
```

5. Create softlink to MGS on both `es1` and `es2`
```
ln -s /dev/ddn/mgs /dev/mapper/mgs
```

6. Start EXAScaler configuration
```
emf config load
emf install --steps nics,hosts,restart_network --password DDNSolutions4U
```
Note: The install step will seem to hang at some point while restarting network interfaces. Connect to es2 and add the route back, from step 4. The installation should continue
Once completed, run this command to start Lustre formatting and create cluster:
```
emf install --exclude nics,hosts,restart_network,os,ntp
```
If you are **upgrading**, add `lustre` to the above command

Congratulations! You now have a fully functioning EXAScaler cluster test environment. Proceed to next section for client mounting.

### Upgrading to a new EXAScaler version
Repeat the installation section above, replace the main OS image with the new one then go through the steps again. Note that the last command will need to exclude `lustre` step since the filesystem already exist, otherwise, command will fail.

### Client configuration
The following steps need to be done each client, `client1` and `client2` separately, modify IP address appropriately.
1. Assign IP address to the 3 interfaces manually, using NetworkManager
```
nmcli conn add con-name enp0s3 type ethernet connection.interface-name enp0s3
nmcli conn mod enp0s3 ipv4.address <ip-address>/<netmask> ipv4.gateway <optional-gateway-ip> # should be in the same subnet as management IP on server VMs
nmcli conn add con-name enp0s8 type ethernet connection.interface-name enp0s8 ipv4.address 10.10.10.2/24 ipv4.method manual
nmcli conn add con-name enp0s9 type ethernet connection.interface-name enp0s9 ipv4.address 10.10.10.3/24 ipv4.method manual
nmcli conn up enp0s8 && nmcli conn up enp0s9 && nmcli conn up enp0s3
```

2. Add the following to `/etc/modprobe.d/lustre.conf`
```
options lnet networks="tcp(enp0s8,enp0s9)"
options ko2iblnd peer_credits=32 peer_credits_hiw=16 concurrent_sends=64
options lnet lnet_transaction_timeout=100 lnet_retry_count=2
```

3. Reload Lustre modules
```
lustre_rmmod
modprobe lustre
```

4. Get `mount` command from server VM, log into server VM (either `es1` or `es2`), grab the output of `mount_lustre_client --dry-run`, should look like below
```
mount -t lustre 10.10.10.101@tcp,10.10.10.102@tcp:10.10.10.104@tcp,10.10.10.105@tcp:/e633fs /lustre/e633fs/client
```

5. Create mountpoint on client
```
mkdir /lustre
```

6. Mount Lustre, modify the output from step 4 appropriately
```
mount -t lustre 10.10.10.101@tcp,10.10.10.102@tcp:10.10.10.104@tcp,10.10.10.105@tcp:/e633fs /lustre
```

**Now you should have a Lustre cluster with 2 clients for testing**

## Things to do with this test cluster
1. Nodemaps for multitenancy
2. API (available at `https://<EMF IP>:7443/graphiql`, need to log in via `https://<EMF IP>:7443/` first)
3. Quotas
4. Hot nodes
