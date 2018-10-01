# Docker on FreeBSD using `vm-bhyve`

Software:

 - FreeBSD
 - ZFS (ZVOL)
 - [`bhyve`](https://www.freebsd.org/cgi/man.cgi?query=bhyve&sektion=8) ([`bhyve`](http://bhyve.org/) + [`grub2-bhyve`](https://www.freshports.org/sysutils/grub2-bhyve/) orchestrated by [`vm-bhyve`](https://www.freshports.org/sysutils/vm-bhyve/))
 - pf
 - [`dnsmasq`](https://www.freshports.org/dns/dnsmasq/)
 - [`docker`](https://www.freshports.org/sysutils/docker/) (client) + [`docker-machine`](https://www.freshports.org/sysutils/docker-machine/)
 - [`boot2docker`](https://github.com/boot2docker/boot2docker)

## Configure host

Commands performed on the host system by `root` user:

```
# Install required sotfware
portmaster sysutils/vm-bhyve sysutils/grub2-bhyve dns/dnsmasq sysutils/docker sysutils/docker-machine

# Configure DHCP and DNS services used by bhyve guests
sysrc dnsmasq_enable="YES"
cat <<EOF > /usr/local/etc/dnsmasq.conf
domain-needed
expand-hosts
domain=`hostname`
except-interface=lo0
bind-interfaces
local-service
dhcp-authoritative
interface=bridge0
dhcp-range=172.16.0.10,172.16.0.254
EOF

# Use prepared DHCP and DNS services on the host as well
sed -i '' -e '1s/^/nameserver 172.16.0.1\\
/' /etc/resolv.conf

# Create ZFS dataset for Docker
zfs create -o mountpoint=/usr/docker zroot/docker

# Configure vm-bhyve
sysrc vm_enable="YES"
sysrc vm_dir="zfs:zroot/docker"
vm init
cp /usr/local/share/examples/vm-bhyve/* /usr/docker/.templates/

# Configure NATing
vm switch create -a 172.16.0.1/24 docker-network
sysrc gateway_enable="YES"
sysctl net.inet.ip.forwarding=1 # used for enable forwarding without rebooting the server which is required by gateway_enable="YES"
sysrc pf_enable="YES"
cat <<EOF > /etc/pf.conf
# Docker
nat on re0 from {172.16.0.0/24} to any -> (re0)
EOF

# Turn on NATing
service pf start

# Turn on DHCP and DNS services used by bhyve guests
service dnsmasq start

# Add vm-bhyve configuration template for running Docker VM
cat <<EOF > /usr/docker/.templates/docker.conf
loader="grub"
cpu=1
memory=2G
network0_type="virtio-net"
network0_switch="docker-network"
disk0_type="virtio-blk"
disk0_name="/dev/zvol/zroot/docker/vm1"
disk0_dev="custom"
grub_install0="linux (cd0)/boot/vmlinuz64 loglevel=3 user=docker nomodeset norestore base"
grub_install1="initrd (cd0)/boot/initrd.img"
EOF

# Fetch boot2docker.iso
vm iso https://github.com/boot2docker/boot2docker/releases/download/v18.06.0-ce/boot2docker.iso

# Create ZVOL storage for Docker VM1
zfs create -o volmode=dev -o primarycache=none -o secondarycache=none -o compression=off -V 20G zroot/docker/vm1

# Create Docker VM1
vm create -t docker dockervm1
```

## Run Docker VM

```
# Turn on Docker VM1
vm install dockervm1 boot2docker.iso

# Connect to Docker VM1
vm console dockervm1
```

## Configure Docker VM1 guest

Default `boot2docker` credentials:

```
login: docker
pass: tcuser
```

Commands performed inside `Docker VM1` guest by `root` user:

```
# Format ZVOL storage so it will be used by boot2docker to persist data
sudo su -
fdisk /dev/vda # fdisk cmds: o -> n -> p -> 1 -> (enter) -> (enter) -> w
mkfs.ext4 /dev/vda1

# Go to host system, turn Docker VM1 off and back on and connect to Docker VM1

sudo su -

# Set Docker VM1 hostname to be `dockervm1`
echo dockervm1 > /var/lib/boot2docker/etc/hostname

# Configure Docker daemon
cat <<EOF > /var/lib/boot2docker/etc/docker/daemon.json
{
    "default-runtime": "runc-nopivot",
    "runtimes": {
        "runc-nopivot": {
            "path": "/var/lib/boot2docker/etc/docker/runc-nopivot.sh"
        }
    }
}
EOF

cat <<EOF > /var/lib/boot2docker/etc/docker/runc-nopivot.sh
#!/bin/sh
args=\`echo "\$@" | sed 's/create/create --no-pivot/'\`
/usr/local/bin/docker-runc \$args
EOF

chmod +x /var/lib/boot2docker/etc/docker/runc-nopivot.sh

# Go to host system, turn Docker VM1 off and back on
```

## Configure Docker to connect to Docker VM1 guest

Commands performed on the host system by regualar user:

```
# Copy SSH public key to Docker VM1
ssh-copy-id -i ~/.ssh/id_rsa.pub docker@dockervm1.`hostname`

# Create Docker Machine which will represent Docker VM1
docker-machine create \
--driver generic \
--generic-ip-address dockervm1.`hostname` \
--generic-ssh-user docker \
dockervm1

# Set environment variables for `docker` so it connects to created Docker Machine
eval $(docker-machine env dockervm1)

# Test if Docker works
docker run --rm hello-world
```

## Other usefull commands

Disconnect from `vm console`:

```
~ + Ctrl+d
```

Turn off `Docker VM1`:

```
vm stop dockervm1
```

## Sources

 - [https://www.freebsd.org/doc/handbook/virtualization-host-bhyve.html](https://www.freebsd.org/doc/handbook/virtualization-host-bhyve.html)
 - [https://matiasaguirre.com/posts/docker-on-freebsd/](https://matiasaguirre.com/posts/docker-on-freebsd/)
 - [https://github.com/churchers/vm-bhyve/blob/master/README.md](https://github.com/churchers/vm-bhyve/blob/master/README.md)
 - [https://github.com/churchers/vm-bhyve/wiki](https://github.com/churchers/vm-bhyve/wiki)

