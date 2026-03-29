# Commands history
## what i did before start reading Phase 1
```bash
sudo dnf install -y qemu-kvm podman virt-install
mkdir ~/Mino/data
```


## Phase 1
```bash
sudo mkdir -p /data/oci
sudo chown -R rocky: /data/oci

sudo mkdir -p /opt/workdir
sudo chown -R rocky: /opt/workdir
cd /opt/workdir

sudo sysctl -w net.ipv4.ip_forward=1

cat <<EOF > openchami-net.xml
<network>
  <name>openchami-net</name>
  <bridge name="virbr-openchami" />
  <forward mode='route'/>
   <ip address="172.16.0.254" netmask="255.255.255.0">
   </ip>
</network>
EOF

# on this get error: Operation not supported: Cannot use direct socket mode if no URI is set
sudo virsh net-define openchami-net.xml
sudo virsh net-start openchami-net
sudo virsh net-autostart openchami-net

# possible solution
sudo dnf install -y libvirt
systemctl start virtqemud.socket

# then get: error: Failed to connect socket to '/var/run/libvirt/virtnetworkd-sock': No such file or directory
# what did
sudo systemctl enable libvirtd
sudo systemctl start libvirtd

# and get success
sudo virsh net-define openchami-net.xml
# out: Network openchami-net defined from openchami-net.xml
# and success on other 2 commandd
sudo virsh net-start openchami-net
sudo virsh net-autostart openchami-net

echo "172.16.0.254 demo.openchami.cluster" | sudo tee -a /etc/hosts > /dev/null

sudo vim /etc/systemd/containers/minio.container
"""
[Unit]
Description=Minio S3
After=local-fs.target network-online.target
Wants=local-fs.target network-online.target

[Container]
ContainerName=minio-server
Image=docker.io/minio/minio:latest
# Volumes
Volume=/home/danil/Minio/data:/data:Z

# Ports
PublishPort=9000:9000
PublishPort=9090:9090

# Environemnt Variables
Environment=MINIO_ROOT_USER=admindanil
Environment=MINIO_ROOT_PASSWORD=admindanil

# Command to run in container
Exec=server /data --console-address ":9090"

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
"""

sudo systemctl daemon-reload && sudo systemctl start minio.service

sudo vim /etc/containers/systemd/registry.container
"""
[Unit]
Description=Image OCI Registry
After=network-online.target
Requires=network-online.target

[Container]
ContainerName=registry
HostName=registry
Image=docker.io/library/registry:latest
Volume=/data/oci:/var/lib/registry:Z
PublishPort=5000:5000

[Service]
TimeoutStartSec=0
Restart=always

[Install]
WantedBy=multi-user.target
"""

sudo systemctl daemon-reload && sudo systemctl start registry.service

# checkpoint
for s in minio registry; do echo -n "$s: "; systemctl is-failed $s; done

# install openchami
# Set repository details
OWNER="openchami"
REPO="release"

# Identify the latest release RPM
API_URL="https://api.github.com/repos/${OWNER}/${REPO}/releases/latest"
release_json=$(curl -s "$API_URL")
rpm_url=$(echo "$release_json" | jq -r '.assets[] | select(.name | endswith(".rpm")) | .browser_download_url' | head -n 1)
rpm_name=$(echo "$release_json" | jq -r '.assets[] | select(.name | endswith(".rpm")) | .name' | head -n 1)

# Download the RPM
curl -L -o "$rpm_name" "$rpm_url"

# Install the RPM
sudo rpm -Uvh "$rpm_name"

cat <<EOF | sudo tee /etc/openchami/configs/coredhcp.yaml
server4:
  # You can configure the specific interfaces that you want OpenCHAMI to listen on by
  # uncommenting the lines below and setting the interface
  listen:
    - "%virbr-openchami"
  plugins:
    # You are able to set the IP address of the system in server_id as the place to look for a DHCP server
    # DNS is able to be set to whatever you want but it is much easier if you keep it set to the server IP
    # Router is also able to be set to whatever you network router address is
    - server_id: 172.16.0.254
    - dns: 172.16.0.254
    - router: 172.16.0.254
    - netmask: 255.255.255.0
    # The lines below define where the system should assign ip addresses for systems that do not have
    # mac addresses stored in SMD
    - coresmd: https://demo.openchami.cluster:8443 http://172.16.0.254:8081 /root_ca/root_ca.crt 30s 1h false
    - bootloop: /tmp/coredhcp.db default 5m 172.16.0.200 172.16.0.250
EOF

sudo openchami-certificate-update update demo.openchami.cluster

grep -RnE 'demo|openchami\.cluster' /etc/openchami/configs/openchami.env /etc/containers/systemd/
#out:
"""
/etc/openchami/configs/openchami.env:2:SYSTEM_NAME=demo
/etc/openchami/configs/openchami.env:3:SYSTEM_DOMAIN=openchami.cluster
/etc/openchami/configs/openchami.env:5:SYSTEM_URL=demo.openchami.cluster
/etc/openchami/configs/openchami.env:8:URLS_SELF_ISSUER=https://demo.openchami.cluster
/etc/openchami/configs/openchami.env:9:URLS_SELF_PUBLIC=https://demo.openchami.cluster
/etc/openchami/configs/openchami.env:10:URLS_LOGIN=https://demo.openchami.cluster/login
/etc/openchami/configs/openchami.env:11:URLS_CONSENT=https://demo.openchami.cluster/consent
/etc/openchami/configs/openchami.env:12:URLS_LOGOUT=https://demo.openchami.cluster/logout
/etc/openchami/configs/openchami.env:42:DOCKER_STEPCA_INIT_DNS_NAMES=step-ca,step-ca.openchami.cluster
/etc/containers/systemd/acme-deploy.container:30:  -d demo.openchami.cluster \
/etc/containers/systemd/acme-register.container:8:ContainerName=demo.openchami.cluster
/etc/containers/systemd/acme-register.container:9:HostName=demo.openchami.cluster
/etc/containers/systemd/acme-register.container:26:  -d demo.openchami.cluster \
/etc/containers/systemd/opaal.container:26:PodmanArgs=--add-host='demo.openchami.cluster:192.168.1.111'
"""

sudo systemctl start openchami.target
systemctl list-dependencies openchami.target
#out:
"""
openchami.target
● ├─acme-deploy.service
● ├─acme-register.service
● ├─bss-init.service
● ├─bss.service
● ├─cloud-init-server.service
● ├─coresmd-coredhcp.service
× ├─coresmd-coredns.service
● ├─haproxy.service
● ├─hydra-gen-jwks.service
● ├─hydra-migrate.service
● ├─hydra.service
● ├─opaal-idp.service
● ├─opaal.service
● ├─openchami-cert-trust.service
● ├─postgres.service
● ├─smd-init.service
● ├─smd.service
● └─step-ca.service
"""

# check logs coresmd-coredns.service
journalctl -eu coresmd-coredns.service
#out:
"""
...
... plugin/coresmd: smd_url is required
...
"""
# but in tutorial was writing about configuring dhcp-things, so i diceded to ignore it yet

# install ochami cli client
latest_release_url=$(curl -s https://api.github.com/repos/OpenCHAMI/ochami/releases/latest | jq -r '.assets[] | select(.name | endswith("amd64.rpm")) | .browser_download_url')
curl -L "${latest_release_url}" -o ochami.rpm
sudo dnf install -y ./ochami.rpm

ochami version
#out:
"""
Version:    0.6.1
Tag:        v0.6.1
Branch:     HEAD
Commit:     ee95e45e63a0d71be4a0fe3dd0cb797ceabb749a
Git State:  clean
Date:       2026-02-05T22:44:56Z
Go:         go1.25.7
Compiler:   gc
Build Host: runnervmkj6or
Build User: runner
"""

sudo ochami config cluster set --system --default demo cluster.uri https://demo.openchami.cluster:8443
ochami config show
#out:
"""
clusters:
    - cluster:
        enable-auth: true
        uri: https://demo.openchami.cluster:8443
      name: demo
default-cluster: demo
log:
    format: rfc3339
    level: warning
"""

ochami bss service status
#out:
"""
2026-02-15T02:48:59-05:00 ERR status.go:51 > failed to get BSS status error="GetStatus(): error getting BSS all status: error making GET request to BSS: failed to execute HTTP request: Get \"https://demo.openchami.cluster:8443/boot/v1/service/status\": dial tcp 172.16.0.254:8443: i/o timeout"
2026-02-15T02:48:59-05:00 ERR cli.go:519 > see 'ochami bss service status --help' for long command help
"""

# trying use command from Troubleshooting section
sudo openchami-certificate-update update demo.openchami.cluster
sudo systemctl restart openchami.target

# but coresmd-coredns still down
# another command
sudo systemctl restart acme-deploy
# and it doesnt work, there is no even this service on chart diagram in tutorial
# i guess i run too new version of program, newer than tutorial

# i realized that libvirtd was off, and ochami subnet wasn't existed
sudo systemctl start libvirtd
# then i was able to ping it
ping demo.openchami.cluster

# trying update cirtificates and restart it
sudo openchami-certificate-update update demo.openchami.cluster && sudo systemctl restart openchami.target
# and coresmd-coredns still down(

# but ochami bss is now running!
ochami bss service status
#out:
"""
{"bss-status":"running"}
"""

# and ochame smd services statuses is good
ochami smd service status
#out:
"""
{"code":0,"message":"HSM is healthy"}
"""
```

## Phase 2.
```bash
# seems to need start it manualy (or by script) after reboot cause it cannot be
# enabled in systemd as its "transient or generated service"
sudo systemctl start acme-deploy

# libvirt i need to run manually too cause its innactive after reboot
sudo systemctl start libvirtd

# but after restarting openchami.target its not work yet
sudo systmemctl list-dependecies openchami.target
"""
openchami.target
○ ├─acme-deploy.service
● ├─acme-register.service
● ├─bss-init.service
○ ├─bss.service
○ ├─cloud-init-server.service
● ├─coresmd-coredhcp.service
○ ├─coresmd-coredns.service
○ ├─haproxy.service
○ ├─hydra-gen-jwks.service
● ├─hydra-migrate.service
○ ├─hydra.service
● ├─opaal-idp.service
● ├─opaal.service
● ├─openchami-cert-trust.service
● ├─postgres.service
● ├─smd-init.service
○ ├─smd.service
● └─step-ca.service
"""

# in bss-init.service logs found this
"""
Feb 18 21:33:17 localhost.localdomain bss-init[17254]: 2026/02/18 16:33:17.314265 main.go:235: ERROR: failed to ping Postgres at postgres:5432 (attempt 3, retrying in 5 seconds): dial tcp: lookup postgres on 10.89.3.1:53: read udp 10.89.3.41:52906->10.89.3.1:53: read: no route to host
Feb 18 21:33:17 localhost.localdomain bss-init[17238]: 2026/02/18 16:33:17.314265 main.go:235: ERROR: failed to ping Postgres at postgres:5432 (attempt 3, retrying in 5 seconds): dial tcp: lookup postgres on 10.89.3.1:53: read udp 10.89.3.41:52906->10.89.3.1:53: read: no route to host
"""
# its strange that service trying to find postgres at 10.89.3.1:53 cause we aren't pointed this address

# the same thing in smd-init.service
"""
Feb 18 21:53:13 localhost.localdomain smd-init[11265]: 2026/02/18 16:53:13.670751 main.go:205: Ping failed: 'dial tcp: lookup postgres on 10.89.3.1:53: read udp 10.89.3.25:41456->10.89.3.1:53: read: no route to host'
Feb 18 21:53:13 localhost.localdomain smd-init[11265]: 2026/02/18 16:53:13.670779 main.go:206: Retrying after 5 seconds...
"""

# the same thing in opaal.service
"""
Feb 18 21:20:34 localhost.localdomain systemd[1]: Started The opaal container.
Feb 18 21:20:34 localhost.localdomain opaal[13229]: failed to fetch server config: failed to do request: Get "http://opaal-idp:3332/.well-known/openid-configuration": dial tcp: lookup opaal-idp on 10.89.0.1:53: read udp 10.89.0.13:58290->10.89.0.1:53: i/o timeout
"""

# and in hydra-migrate.service
"""
Feb 18 22:18:13 localhost.localdomain hydra-migrate[29731]: time=2026-02-18T17:18:13Z level=info msg=Retrying in 5.000000 seconds... audience=application error=map[message:failed to connect to `user=hydra-user database=hydradb`: hostname resolving error: lookup postgres on 10.89.3.1:53: read udp 10.89.3.75:57220->10.89.3.1:53: read: no route to host] service_name=Ory Hydra service_version=v2.3.0
"""

# i realized that its podman's nets
# i would never (well maybe too late) suggest that problem might be in firewall
# summary: the firewall was blocking all attempts to resolve names in this podman
# openchami-internal.network because it wasnt present in its "trusted" list
sudo firewall-cmd --get-active-zones
"""
libvirt
  interfaces: virbr0
libvirt-routed
  interfaces: virbr-openchami
public
  interfaces: enp0s3
trusted
  sources: 10.88.0.0/16
"""

# to fix that we need add it there
sudo firewall-cmd --zone=trusted --add-interface=podman4 --permanent && \
    sudo firewall-cmd --reload

# with acme-deploy.register, hydra services i see the same problem, so add to trusted
# firewall list all the podmans network interfaces
# after that i was able to start whole dependencies of openchami.target service
# and start itself
# and get valid status:
ochami bss service status
"""
{"bss-status":"running"}
"""


# ACTUAL START DOING PHASE 2
# create nodes.yaml for discovery
mkdir -p /opt/workdir/nodes
curl -o /opt/workdir/nodes/nodes.yaml https://raw.githubusercontent.com/OpenCHAMI/tutorial-2025/refs/heads/main/Phase%202/nodes.yaml
cat /opt/workdir/nodes/nodes.yaml  # Verify contents

# regenerate demo access token, because system was reboot
export DEMO_ACCESS_TOKEN=$(sudo bash -lc 'gen_access_token')

# populate SMD with node information
ochami discover static -f yaml -d @/opt/workdir/nodes/nodes.yaml

# check it work
"""
danil@localhost /o/workdir> ochami smd component get | jq '.Components[] | select(.Type == "Node")'
{
  "Enabled": true,
  "ID": "x1000c0s0b0n0",
  "NID": 1,
  "Role": "Compute",
  "State": "On",
  "Type": "Node"
}
{
  "Enabled": true,
  "ID": "x1000c0s0b1n0",
  "NID": 2,
  "Role": "Compute",
  "State": "On",
  "Type": "Node"
}
{
  "Enabled": true,
  "ID": "x1000c0s0b2n0",
  "NID": 3,
  "Role": "Compute",
  "State": "On",
  "Type": "Node"
}
{
  "Enabled": true,
  "ID": "x1000c0s0b3n0",
  "NID": 4,
  "Role": "Compute",
  "State": "On",
  "Type": "Node"
}
{
  "Enabled": true,
  "ID": "x1000c0s0b4n0",
  "NID": 5,
  "Role": "Compute",
  "State": "On",
  "Type": "Node"
}
"""


"""
danil@localhost /o/workdir> mkdir -p /opt/workdir/images
danil@localhost /o/workdir> cd /opt/workdir/images
danil@localhost /o/w/images> curl -L https://github.com/regclient/regclient/releases/latest/download/regctl-linux-amd64 > regctl && sudo mv regctl /usr/local/bin/regctl && sudo chmod 755 /usr/local/bin/regctl
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 11.6M  100 11.6M    0     0  6311k      0  0:00:01  0:00:01 --:--:-- 15.5M
[sudo] password for danil: 
danil@localhost /o/w/images> /usr/local/bin/regctl registry set --tls disabled demo.openchami.cluster:5000
danil@localhost /o/w/images> man regctl
No manual entry for regctl
danil@localhost /o/w/images [16]> cat ~/.regctl/config.json
{
  "hosts": {
    "demo.openchami.cluster:5000": {
      "tls": "disabled",
      "hostname": "demo.openchami.cluster:5000",
      "reqConcurrent": 3
    }
  }
}
"""

# it turns out that was neccessary to install s3cmd 
sudo dnf install -y s3cmd

# create config for using s3cmd with our minio
echo <<EOF >  ~/.s3cfg
# Setup endpoint
host_base = demo.openchami.cluster:9000
host_bucket = demo.openchami.cluster:9000
bucket_location = us-east-1
use_https = False

# Setup access keys
access_key = admindanil
secret_key = admindanil

# Enable S3 v4 signature APIs
signature_v2 = False
EOF

# do this
s3cmd mb s3://efi
s3cmd setacl s3://efi --acl-public
s3cmd mb s3://boot-images
s3cmd setacl s3://boot-images --acl-public

# create /opt/workdir/s3-public-read-boot.json with:
"""
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":"*",
      "Action":["s3:GetObject"],
      "Resource":["arn:aws:s3:::boot-images/*"]
    }
  ]
}
"""

# create /opt/workdir/s3-public-read-efi.json with:
"""
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":"*",
      "Action":["s3:GetObject"],
      "Resource":["arn:aws:s3:::efi/*"]
    }
  ]
}
"""

s3cmd setpolicy /opt/workdir/s3-public-read-boot.json s3://boot-images \
    --host=172.16.0.254:9000 \
    --host-bucket=172.16.0.254:9000

s3cmd setpolicy /opt/workdir/s3-public-read-efi.json s3://efi \
    --host=172.16.0.254:9000 \
    --host-bucket=172.16.0.254:9000

# ... build base command

# after starting build base image the space was wasted in 10min, so i was
# made expand VM storage if this image
# on my pc:
vboxmanage modifyhd rocky9/rocky9.vdi --resize 51200

# inside vm:
sudo lvextend -l +100%FREE /dev/rl/root
sudo xfs_growfs /


# ... build command again:
# craete yaml of base image /opt/workdir/images/rocky-base-9.yaml
"""
options:
  layer_type: 'base'
  name: 'rocky-base'
  publish_tags: '9'
  pkg_manager: 'dnf'
  parent: 'scratch'
  publish_registry: 'demo.openchami.cluster:5000/demo'
  registry_opts_push:
    - '--tls-verify=false'

repos:
  - alias: 'Rocky_9_BaseOS'
    url: 'https://dl.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/'
    gpg: 'https://dl.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9'
  - alias: 'Rocky_9_AppStream'
    url: 'https://dl.rockylinux.org/pub/rocky/9/AppStream/x86_64/os/'
    gpg: 'https://dl.rockylinux.org/pub/rocky/RPM-GPG-KEY-Rocky-9'

package_groups:
  - 'Minimal Install'
  - 'Development Tools'

packages:
  - chrony
  - cloud-init
  - dracut-live
  - kernel
  - rsyslog
  - sudo
  - wget

cmds:
  - cmd: 'dracut --add "dmsquash-live livenet network-manager" --kver $(basename /lib/modules/*) -N -f --logfile /tmp/dracut.log 2>/dev/null'
  - cmd: 'echo DRACUT LOG:; cat /tmp/dracut.log'
"""

podman run --rm \
    --device /dev/fuse \
    --network host \
    -v /opt/workdir/images/rocky-base-9.yaml:/home/builder/config.yaml \
        ghcr.io/openchami/image-build-el9:v0.1.1 image-build \
    --config config.yaml \
    --log-level DEBUG

# build succeded
regctl repo ls demo.openchami.cluster:5000
# out: demo/rocky-base
regctl tag ls demo.openchami.cluster:5000/demo/rocky-base
# out: 9

# build compute image
vim /opt/workdir/images/compute-base-rocky9.yaml
"""
options:
  layer_type: 'base'
  name: 'compute-base'
  publish_tags:
    - 'rocky9'
  pkg_manager: 'dnf'
  parent: 'demo.openchami.cluster:5000/demo/rocky-base:9'
  registry_opts_pull:
    - '--tls-verify=false'

  # Publish SquashFS image to local S3
  publish_s3: 'http://demo.openchami.cluster:9000'
  s3_prefix: 'compute/base/'
  s3_bucket: 'boot-images'

  # Publish OCI image to container registry
  #
  # This is the only way to be able to re-use this image as
  # a parent for another image layer.
  publish_registry: 'demo.openchami.cluster:5000/demo'
  registry_opts_push:
    - '--tls-verify=false'

repos:
  - alias: 'Epel9'
    url: 'https://dl.fedoraproject.org/pub/epel/9/Everything/x86_64/'
    gpg: 'https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-9'

packages:
  - boxes
  - cowsay
  - figlet
  - fortune-mod
  - git
  - nfs-utils
  - tcpdump
  - traceroute
  - vim
"""

podman run --rm \
    --device /dev/fuse \
    --network host \
    -e S3_ACCESS=admindanil \
    -e S3_SECRET=admindanil \
    -v /opt/workdir/images/compute-base-rocky9.yaml:/home/builder/config.yaml \
    ghcr.io/openchami/image-build-el9:v0.1.1 image-build --config config.yaml --log-level DEBUG
# i could not build it due to timeout of "dnf insta <packages>", its failed after check
# all mirrors for installing

# i was forced to change adapter from Bridge to NAT and organize vpn tunnel for host
# so then i finnaly builded it
"""
...
-------------------BUILD LAYER--------------------
Generating labels
Labels: {'org.openchami.image.name': 'compute-base', 'org.openchami.image.type': 'base', 'org.openchami.image.parent': 'demo.openchami.cluster:5000/demo/rocky-base:9', 'org.openchami.image.package-manager': 'dnf', 'org.openchami.image.tags': 'rocky9', 'org.openchami.image.build-date': '2026-03-29T08:29:39.716081'}
Publishing to S3 at boot-images
/home/builder/.local/share/containers/storage/overlay/13eb48a3b3eb50b011e243c3aae648f97857d04c72ebe9f6adb47679bb4c651e/merged
squashing container image
Image Name: compute/base/rocky9.7-compute-base-rocky9
initramfs: initramfs-5.14.0-611.41.1.el9_7.x86_64.img
vmlinuz: vmlinuz-5.14.0-611.41.1.el9_7.x86_64
Pushing /home/builder/.local/share/containers/storage/overlay/13eb48a3b3eb50b011e243c3aae648f97857d04c72ebe9f6adb47679bb4c651e/merged/boot/initramfs-5.14.0-611.41.1.el9_7.x86_64.img as efi-images/compute/base/initramfs-5.14.0-611.41.1.el9_7.x86_64.img to boot-images
Pushing /home/builder/.local/share/containers/storage/overlay/13eb48a3b3eb50b011e243c3aae648f97857d04c72ebe9f6adb47679bb4c651e/merged/boot/vmlinuz-5.14.0-611.41.1.el9_7.x86_64 as efi-images/compute/base/vmlinuz-5.14.0-611.41.1.el9_7.x86_64 to boot-images
Pushing /var/tmp/tmpmoktesht/rootfs as compute/base/rocky9.7-compute-base-rocky9 to boot-images
Publishing to registry at demo.openchami.cluster:5000/demo
pushing layer compute-base to demo.openchami.cluster:5000/demo/compute-base:rocky9
"""

regctl repo ls demo.openchami.cluster:5000
"""
demo/compute-base
demo/rocky-base
"""

regctl tag ls demo.openchami.cluster:5000/demo/compute-base
# out: rocky9

s3cmd ls -Hr s3://boot-images | grep compute/base
"""
2026-03-29 08:32  1492M  s3://boot-images/compute/base/rocky9.7-compute-base-rocky9
2026-03-29 08:32    85M  s3://boot-images/efi-images/compute/base/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
2026-03-29 08:32    14M  s3://boot-images/efi-images/compute/base/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
"""

# create debug image yaml
"""
options:
  layer_type: base
  name: compute-debug
  publish_tags:
    - 'rocky9'
  pkg_manager: dnf
  parent: '172.16.0.254:5000/demo/compute-base:rocky9'
  registry_opts_pull:
    - '--tls-verify=false'

  # Publish to local S3
  publish_s3: 'http://172.16.0.254:9000'
  s3_prefix: 'compute/debug/'
  s3_bucket: 'boot-images'

packages:
  - shadow-utils

cmds:
  - cmd: "useradd -mG wheel -p 'test' testuser"
"""

podman run --rm \
    --device /dev/fuse \
    -e S3_ACCESS=admindanil \
    -e S3_SECRET=admindanil \
    -v /opt/workdir/images/compute-debug-rocky9.yaml:/home/builder/config.yaml \
    ghcr.io/openchami/image-build-el9:v0.1.1 image-build --config config.yaml --log-level DEBUG

# check success
s3cmd ls -Hr s3://boot-images/
"""
2026-03-29 08:32  1492M  s3://boot-images/compute/base/rocky9.7-compute-base-rocky9
2026-03-29 08:51  1492M  s3://boot-images/compute/debug/rocky9.7-compute-debug-rocky9
2026-03-29 08:32    85M  s3://boot-images/efi-images/compute/base/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
2026-03-29 08:32    14M  s3://boot-images/efi-images/compute/base/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
2026-03-29 08:51    85M  s3://boot-images/efi-images/compute/debug/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
2026-03-29 08:51    14M  s3://boot-images/efi-images/compute/debug/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
"""

s3cmd ls -Hr s3://boot-images | grep compute/debug
"""
2026-03-29 08:51  1492M  s3://boot-images/compute/debug/rocky9.7-compute-debug-rocky9
2026-03-29 08:51    85M  s3://boot-images/efi-images/compute/debug/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
2026-03-29 08:51    14M  s3://boot-images/efi-images/compute/debug/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
"""

# get actual urls
s3cmd ls -Hr s3://boot-images | grep compute/debug | awk '{print $4}' | sed 's-s3://-http://172.16.0.254:9000/-'
"""
http://172.16.0.254:9000/boot-images/compute/debug/rocky9.7-compute-debug-rocky9
http://172.16.0.254:9000/boot-images/efi-images/compute/debug/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
http://172.16.0.254:9000/boot-images/efi-images/compute/debug/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
"""

# add as root file /etc/profile.d/build-image.sh
"""
build-image-rh9()
{
    if [ -z "$1" ]; then
        echo 'Path to image config file required.' 1>&2;
        return 1;
    fi;
    if [ ! -f "$1" ]; then
        echo "$1 does not exist." 1>&2;
        return 1;
    fi;
    podman run \
            --rm \
            --device /dev/fuse \
            -e S3_ACCESS=admindanil \
            -e S3_SECRET=admindanil \
            -v "$(realpath $1)":/home/builder/config.yaml:Z \
            ${EXTRA_PODMAN_ARGS} \
            ghcr.io/openchami/image-build-el9:v0.1.1 \
            image-build \
                --config config.yaml \
                --log-level DEBUG
}

build-image-rh8()
{
    if [ -z "$1" ]; then
        echo 'Path to image config file required.' 1>&2;
        return 1;
    fi;
    if [ ! -f "$1" ]; then
        echo "$1 does not exist." 1>&2;
        return 1;
    fi;
    podman run \
           --rm \
           --device /dev/fuse \
           -e S3_ACCESS=admindanil \
           -e S3_SECRET=admindanil \
           -v "$(realpath $1)":/home/builder/config.yaml:Z \
           ${EXTRA_PODMAN_ARGS} \
           ghcr.io/openchami/image-build:v0.1.0 \
           image-build \
                --config config.yaml \
                --log-level DEBUG
}
alias build-image=build-image-rh9
"""

mkdir -p /opt/workdir/cloud-init
cd /opt/workdir/cloud-init
ssh-keygen -t ed25519

cat <<EOF | tee /opt/workdir/cloud-init/ci-defaults.yaml
---
base-url: "http://172.16.0.254:8081/cloud-init"
cluster-name: "demo"
nid-length: 3
public-keys:
  - "$(cat ~/.ssh/id_ed25519.pub)"
short-name: "nid"
EOF
"""
---
base-url: "http://172.16.0.254:8081/cloud-init"
cluster-name: "demo"
nid-length: 3
public-keys:
  - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHd4cL2rUAfoo1RM3fPGLxswZozGOzAFcIKtBNcuFrnZ danil@localhost.localdomain"
short-name: "nid"
"""

ochami cloud-init defaults set -f yaml -d @/opt/workdir/cloud-init/ci-defaults.yaml
ochami cloud-init defaults get -F json-pretty
"""
{
  "base-url": "http://172.16.0.254:8081/cloud-init",
  "cluster-name": "demo",
  "nid-length": 3,
  "public-keys": [
    "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHd4cL2rUAfoo1RM3fPGLxswZozGOzAFcIKtBNcuFrnZ danil@localhost.localdomain"
  ],
  "short-name": "nid"
}
"""

# create /opt/workdir/cloud-init/ci-group-compute.yaml with:
"""
- name: compute
  description: "compute config"
  file:
    encoding: plain
    content: |
      ## template: jinja
      #cloud-config
      merge_how:
      - name: list
        settings: [append]
      - name: dict
        settings: [no_replace, recurse_list]
      users:
        - name: root
          ssh_authorized_keys: {{ ds.meta_data.instance_data.v1.public_keys }}
      disable_root: false
"""

ochami cloud-init group set -f yaml -d @/opt/workdir/cloud-init/ci-group-compute.yaml
ochami cloud-init group get config compute
"""
## template: jinja
#cloud-config
merge_how:
- name: list
  settings: [append]
- name: dict
  settings: [no_replace, recurse_list]
users:
  - name: root
    ssh_authorized_keys: {{ ds.meta_data.instance_data.v1.public_keys }}
disable_root: false
"""

ochami cloud-init group render compute x1000c0s0b0n0
"""
## template: jinja
#cloud-config
merge_how:
- name: list
  settings: [append]
- name: dict
  settings: [no_replace, recurse_list]
users:
  - name: root
    ssh_authorized_keys: ['ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHd4cL2rUAfoo1RM3fPGLxswZozGOzAFcIKtBNcuFrnZ danil@localhost.localdomain']
disable_root: false
"""

ochami cloud-init node get vendor-data x1000c0s0b0n0
"""
#include
http://172.16.0.254:8081/cloud-init/compute.yaml
"""

ochami cloud-init node get meta-data x1000c0s0b0n0 -F yaml
"""
- cluster-name: demo
  hostname: nid001
  instance-id: i-fd5c29bc
  instance_data:
    v1:
        instance_id: i-fd5c29bc
        local_ipv4: 172.16.0.1
        public_keys:
            - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHd4cL2rUAfoo1RM3fPGLxswZozGOzAFcIKtBNcuFrnZ danil@localhost.localdomain
        vendor_data:
            cloud_init_base_url: http://172.16.0.254:8081/cloud-init
            cluster_name: demo
            groups:
                compute:
                    Description: compute config
            version: "1.0"
  local-hostname: nid001
"""

s3cmd ls -Hr s3://boot-images/ | awk '{print $4}' | grep base
"""
s3://boot-images/compute/base/rocky9.7-compute-base-rocky9
s3://boot-images/efi-images/compute/base/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
s3://boot-images/efi-images/compute/base/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
"""

mkdir -p /opt/workdir/boot

URIS=$(s3cmd ls -Hr s3://boot-images | grep compute/base | awk '{print $4}' | sed 's-s3://-http://172.16.0.254:9000/-' | xargs)
URI_IMG=$(echo "$URIS" | cut -d' ' -f1)
URI_INITRAMFS=$(echo "$URIS" | cut -d' ' -f2)
URI_KERNEL=$(echo "$URIS" | cut -d' ' -f3)
cat <<EOF | tee /opt/workdir/boot/boot-compute-base.yaml
---
kernel: '${URI_KERNEL}'
initrd: '${URI_INITRAMFS}'
params: 'nomodeset ro root=live:${URI_IMG} ip=dhcp overlayroot=tmpfs overlayroot_cfgdisk=disabled apparmor=0 selinux=0 console=ttyS0,115200 ip6=off cloud-init=enabled ds=nocloud-net;s=http://172.16.0.254:8081/cloud-init'
macs:
  - 52:54:00:be:ef:01
  - 52:54:00:be:ef:02
  - 52:54:00:be:ef:03
  - 52:54:00:be:ef:04
  - 52:54:00:be:ef:05
EOF
"""
---
kernel: 'http://172.16.0.254:9000/boot-images/efi-images/compute/base/vmlinuz-5.14.0-611.41.1.el9_7.x86_64'
initrd: 'http://172.16.0.254:9000/boot-images/efi-images/compute/base/initramfs-5.14.0-611.41.1.el9_7.x86_64.img'
params: 'nomodeset ro root=live:http://172.16.0.254:9000/boot-images/compute/base/rocky9.7-compute-base-rocky9 ip=dhcp overlayroot=tmpfs overlayroot_cfgdisk=disabled apparmor=0 selinux=0 console=ttyS0,115200 ip6=off cloud-init=enabled ds=nocloud-net;s=http://172.16.0.254:8081/cloud-init'
macs:
  - 52:54:00:be:ef:01
  - 52:54:00:be:ef:02
  - 52:54:00:be:ef:03
  - 52:54:00:be:ef:04
  - 52:54:00:be:ef:05
"""

ochami bss boot params set -f yaml -d @/opt/workdir/boot/boot-compute-base.yaml

ochami bss boot params get -F yaml
"""
- cloud-init:
    meta-data: null
    phone-home:
        fqdn: ""
        hostname: ""
        instance_id: ""
        pub_key_dsa: ""
        pub_key_ecdsa: ""
        pub_key_rsa: ""
    user-data: null
  initrd: http://172.16.0.254:9000/boot-images/efi-images/compute/base/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
  kernel: http://172.16.0.254:9000/boot-images/efi-images/compute/base/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
  macs:
    - 52:54:00:be:ef:01
    - 52:54:00:be:ef:02
    - 52:54:00:be:ef:03
    - 52:54:00:be:ef:04
    - 52:54:00:be:ef:05
  params: nomodeset ro root=live:http://172.16.0.254:9000/boot-images/compute/base/rocky9.7-compute-base-rocky9 ip=dhcp overlayroot=tmpfs overlayroot_cfgdisk=disabled apparmor=0 selinux=0 console=ttyS0,115200 ip6=off cloud-init=enabled ds=nocloud-net;s=http://172.16.0.254:8081/cloud-init
"""

# yes, boot parameters contains base image, but command in tutorial wrong, it should be:
ochami bss boot params get | jq
"""
[
  {
    "cloud-init": {
      "meta-data": null,
      "phone-home": {
        "fqdn": "",
        "hostname": "",
        "instance_id": "",
        "pub_key_dsa": "",
        "pub_key_ecdsa": "",
        "pub_key_rsa": ""
      },
      "user-data": null
    },
    "initrd": "http://172.16.0.254:9000/boot-images/efi-images/compute/base/initramfs-5.14.0-611.41.1.el9_7.x86_64.img",
    "kernel": "http://172.16.0.254:9000/boot-images/efi-images/compute/base/vmlinuz-5.14.0-611.41.1.el9_7.x86_64",
    "macs": [
      "52:54:00:be:ef:01",
      "52:54:00:be:ef:02",
      "52:54:00:be:ef:03",
      "52:54:00:be:ef:04",
      "52:54:00:be:ef:05"
    ],
    "params": "nomodeset ro root=live:http://172.16.0.254:9000/boot-images/compute/base/rocky9.7-compute-base-rocky9 ip=dhcp overlayroot=tmpfs overlayroot_cfgdisk=disabled apparmor=0 selinux=0 console=ttyS0,115200 ip6=off cloud-init=enabled ds=nocloud-net;s=http://172.16.0.254:8081/cloud-init"
  }
]
"""

sudo virsh destroy compute1
# out: error: failed to get domain 'compute1'

virt-install
"""
WARNING  KVM acceleration not available, using 'qemu'
ERROR    
--os-variant/--osinfo OS name is required, but no value was
set or detected.

This is now a fatal error. Specifying an OS name is required
for modern, performant, and secure virtual machine defaults.

You can see a full list of possible OS name values with:

   virt-install --osinfo list

If your Linux distro is not listed, try one of generic values
such as: linux2024, linux2022, linux2020, linux2018, linux2016

If you just need to get the old behavior back, you can use:

  --osinfo detect=on,require=off

Or export VIRTINSTALL_OSINFO_DISABLE_REQUIRE=1
"""

# and i rialize i skiped whole 2.6 )))))))))))))))

URIS=$(s3cmd ls -Hr s3://boot-images | grep compute/debug | awk '{print $4}' | sed 's-s3://-http://172.16.0.254:9000/-' | xargs) 
URI_IMG=$(echo "$URIS" | cut -d' ' -f1)
URI_INITRAMFS=$(echo "$URIS" | cut -d' ' -f2)
URI_KERNEL=$(echo "$URIS" | cut -d' ' -f3)
cat <<EOF | tee /opt/workdir/boot/boot-compute-debug.yaml
---
kernel: '${URI_KERNEL}'
initrd: '${URI_INITRAMFS}'
params: 'nomodeset ro root=live:${URI_IMG} ip=dhcp overlayroot=tmpfs overlayroot_cfgdisk=disabled apparmor=0 selinux=0 console=ttyS0,115200 ip6=off cloud-init=enabled ds=nocloud-net;s=http://172.16.0.254:8081/cloud-init'
macs:
  - 52:54:00:be:ef:01
  - 52:54:00:be:ef:02
  - 52:54:00:be:ef:03
  - 52:54:00:be:ef:04
  - 52:54:00:be:ef:05
EOF
"""
---
kernel: 'http://172.16.0.254:9000/boot-images/efi-images/compute/debug/vmlinuz-5.14.0-611.41.1.el9_7.x86_64'
initrd: 'http://172.16.0.254:9000/boot-images/efi-images/compute/debug/initramfs-5.14.0-611.41.1.el9_7.x86_64.img'
params: 'nomodeset ro root=live:http://172.16.0.254:9000/boot-images/compute/debug/rocky9.7-compute-debug-rocky9 ip=dhcp overlayroot=tmpfs overlayroot_cfgdisk=disabled apparmor=0 selinux=0 console=ttyS0,115200 ip6=off cloud-init=enabled ds=nocloud-net;s=http://172.16.0.254:8081/cloud-init'
macs:
  - 52:54:00:be:ef:01
  - 52:54:00:be:ef:02
  - 52:54:00:be:ef:03
  - 52:54:00:be:ef:04
  - 52:54:00:be:ef:05
"""

ochami bss boot params get -F yaml
"""
- cloud-init:
    meta-data: null
    phone-home:
        fqdn: ""
        hostname: ""
        instance_id: ""
        pub_key_dsa: ""
        pub_key_ecdsa: ""
        pub_key_rsa: ""
    user-data: null
  initrd: http://172.16.0.254:9000/boot-images/efi-images/compute/debug/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
  kernel: http://172.16.0.254:9000/boot-images/efi-images/compute/debug/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
  macs:
    - 52:54:00:be:ef:01
    - 52:54:00:be:ef:02
    - 52:54:00:be:ef:03
    - 52:54:00:be:ef:04
    - 52:54:00:be:ef:05
  params: nomodeset ro root=live:http://172.16.0.254:9000/boot-images/compute/debug/rocky9.7-compute-debug-rocky9 ip=dhcp overlayroot=tmpfs overlayroot_cfgdisk=disabled apparmor=0 selinux=0 console=ttyS0,115200 ip6=off cloud-init=enabled ds=nocloud-net;s=http://172.16.0.254:8081/cloud-init
"""

sudo virt-install \
   --name compute1 \
   --memory 4096 \
   --vcpus 1 \
   --disk none \
   --pxe \
   --os-variant centos-stream9 \
   --network network=openchami-net,model=virtio,mac=52:54:00:be:ef:01 \
   --graphics none \
   --console pty,target_type=serial \
   --boot network,hd \
   --boot loader=/usr/share/OVMF/OVMF_CODE.secboot.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/OVMF/OVMF_VARS.fd,loader_secure=no \
   --virt-type kvm
# out: ERROR    Host does not support domain type kvm for virtualization type 'hvm' with architecture 'x86_64'
# i dont have "nested virtualization" on this vm...

```
