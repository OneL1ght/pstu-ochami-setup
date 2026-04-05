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

# i enabled nested VM on this virtualbox machine
# and got inside debug VM in virtualbox vm)
"""
[testuser@nid0001 ~]$ findmnt /
TARGET SOURCE        FSTYPE  OPTIONS
/      LiveOS_rootfs overlay rw,relatime,lowerdir=/run/rootfsbase,upperdir=/run/
"""
# played around a bit (as recommend in tutorial) and closed it
"""
[testuser@nid0001 ~]$ gcc --version
gcc (GCC) 11.5.0 20240719 (Red Hat 11.5.0-11)
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE
"""


# check this again
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
# what i see
"""
s3://boot-images/compute/base/rocky9.7-compute-base-rocky9
s3://boot-images/efi-images/compute/base/initramfs-5.14.0-611.41.1.el9_7.x86_64.img
s3://boot-images/efi-images/compute/base/vmlinuz-5.14.0-611.41.1.el9_7.x86_64
"""

# create boot compute yaml
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

sudo virsh destroy compute1 && sudo virsh start --console compute1

ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@172.16.0.1
# could not connnect due to this error in cloud-init i guess...
# and ssh server wont be up
"""
         Starting Network Manager Script Dispatcher Service...
[  OK  ] Started Network Manager Script Dispatcher Service.
[   57.212603] cloud-init[1055]: 2026-04-02 16:04:15,875 - main.py[ERROR]: failed stage init
[   57.220327] cloud-init[1055]: Traceback (most recent call last):
[   57.228505] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connection.py", line 169, in _new_conn
[   57.238029] cloud-init[1055]:     conn = connection.create_connection(
[   57.243985] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/util/connection.py", line 73, in create_connection
[   57.254205] cloud-init[1055]:     for res in socket.getaddrinfo(host, port, family, socket.SOCK_STREAM):
[   57.262634] cloud-init[1055]:   File "/usr/lib64/python3.9/socket.py", line 966, in getaddrinfo
[   57.271163] cloud-init[1055]:     for res in _socket.getaddrinfo(host, port, family, type, proto, flags):
[   57.277820] cloud-init[1055]: socket.gaierror: [Errno -2] Name or service not known
[   57.285189] cloud-init[1055]: During handling of the above exception, another exception occurred:
[   57.293703] cloud-init[1055]: Traceback (most recent call last):
[   57.299371] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connectionpool.py", line 700, in urlopen
[   57.318730] cloud-init[1055]:     httplib_response = self._make_request(
[   57.320559] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connectionpool.py", line 395, in _make_request
[   57.323447] cloud-init[1055]:     conn.request(method, url, **httplib_request_kw)
[   57.325484] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connection.py", line 234, in request
[   57.328240] cloud-init[1055]:     super(HTTPConnection, self).request(method, url, body=body, headers=headers)
[   57.333743] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 1285, in request
[   57.344497] cloud-init[1055]:     self._send_request(method, url, body, headers, encode_chunked)
[   57.362873] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 1331, in _send_request
[   57.367350] cloud-init[1055]:     self.endheaders(body, encode_chunked=encode_chunked)
[   57.369355] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 1280, in endheaders
[   57.371592] cloud-init[1055]:     self._send_output(message_body, encode_chunked=encode_chunked)
[   57.373776] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 1040, in _send_output
[   57.376137] cloud-init[1055]:     self.send(msg)
[   57.377446] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 980, in send
[   57.385646] cloud-init[1055]:     self.connect()
[   57.389632] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connection.py", line 200, in connect
[   57.398054] cloud-init[1055]:     conn = self._new_conn()
[   57.415107] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connection.py", line 181, in _new_conn
[   57.418082] cloud-init[1055]:     raise NewConnectionError(
[   57.419681] cloud-init[1055]: urllib3.exceptions.NewConnectionError: <urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known
[   57.423925] cloud-init[1055]: During handling of the above exception, another exception occurred:
[   57.426192] cloud-init[1055]: Traceback (most recent call last):
[   57.433157] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/requests/adapters.py", line 612, in send
[   57.438795] cloud-init[1055]:     resp = conn.urlopen(
[   57.445145] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connectionpool.py", line 756, in urlopen
[   57.466273] cloud-init[1055]:     retries = retries.increment(
[   57.467980] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/util/retry.py", line 576, in increment
[   57.470728] cloud-init[1055]:     raise MaxRetryError(_pool, url, error or ResponseError(cause))
[   57.473017] cloud-init[1055]: urllib3.exceptions.MaxRetryError: HTTPConnectionPool(host='cloud-init', port=27777): Max retries exceeded with url: /compute.yaml (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known'))
[   57.487349] cloud-init[1055]: During handling of the above exception, another exception occurred:
[   57.494374] cloud-init[1055]: Traceback (most recent call last):
[   57.499212] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/url_helper.py", line 484, in readurl
[   57.517774] cloud-init[1055]:     r = session.request(**req_args)
[   57.519525] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/requests/sessions.py", line 544, in request
[   57.522323] cloud-init[1055]:     resp = self.send(prep, **send_kwargs)
[   57.524286] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/requests/sessions.py", line 657, in send
[   57.526981] cloud-init[1055]:     r = adapter.send(request, **kwargs)
[   57.533587] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/requests/adapters.py", line 689, in send
[   57.539436] cloud-init[1055]:     raise ConnectionError(e, request=request)
[   57.545957] cloud-init[1055]: requests.exceptions.ConnectionError: HTTPConnectionPool(host='cloud-init', port=27777): Max retries exceeded with url: /compute.yaml (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known'))
[   57.570778] cloud-init[1055]: The above exception was the direct cause of the following exception:
[   57.573275] cloud-init[1055]: Traceback (most recent call last):
[   57.575011] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 237, in _do_include
[   57.577685] cloud-init[1055]:     resp = read_file_or_url(
[   57.584513] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/url_helper.py", line 254, in read_file_or_url
[   57.592919] cloud-init[1055]:     return readurl(url, **kwargs)
[   57.597135] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/url_helper.py", line 528, in readurl
[   57.616480] cloud-init[1055]:     raise url_error from e
[   57.617931] cloud-init[1055]: cloudinit.url_helper.UrlError: HTTPConnectionPool(host='cloud-init', port=27777): Max retries exceeded with url: /compute.yaml (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known'))
[   57.624709] cloud-init[1055]: The above exception was the direct cause of the following exception:
[   57.627940] cloud-init[1055]: Traceback (most recent call last):
[   57.635187] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/cmd/main.py", line 922, in status_wrapper
[   57.643558] cloud-init[1055]:     ret = functor(name, args)
[   57.648618] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/cmd/main.py", line 589, in main_init
[   57.667655] cloud-init[1055]:     init.update()
[   57.669000] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/stages.py", line 575, in update
[   57.671672] cloud-init[1055]:     self.datasource.get_vendordata(), "vendordata"
[   57.673664] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/sources/__init__.py", line 657, in get_vendordata
[   57.676602] cloud-init[1055]:     self.vendordata = self.ud_proc.process(self.get_vendordata_raw())
[   57.680401] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 87, in process
[   57.688474] cloud-init[1055]:     self._process_msg(convert_string(blob), accumulating_msg)
[   57.695593] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 158, in _process_msg
[   57.716422] cloud-init[1055]:     self._do_include(payload, append_msg)
[   57.718246] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 263, in _do_include
[   57.721066] cloud-init[1055]:     _handle_error(message, urle)
[   57.722881] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 71, in _handle_error
[   57.726824] cloud-init[1055]:     raise RuntimeError(error_message) from source_exception
[   57.735635] cloud-init[1055]: RuntimeError: HTTPConnectionPool(host='cloud-init', port=27777): Max retries exceeded with url: /compute.yaml (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known')) for url: http://cloud-init:27777/compute.yaml
[   57.759852] cloud-init[1055]: failed run of stage init
[   57.764481] cloud-init[1055]: ------------------------------------------------------------
[   57.782725] cloud-init[1055]: Traceback (most recent call last):
[   57.784330] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connection.py", line 169, in _new_conn
[   57.787008] cloud-init[1055]:     conn = connection.create_connection(
[   57.788858] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/util/connection.py", line 73, in create_connection
[   57.791700] cloud-init[1055]:     for res in socket.getaddrinfo(host, port, family, socket.SOCK_STREAM):
[   57.795352] cloud-init[1055]:   File "/usr/lib64/python3.9/socket.py", line 966, in getaddrinfo
[   57.803468] cloud-init[1055]:     for res in _socket.getaddrinfo(host, port, family, type, proto, flags):
[   57.810339] cloud-init[1055]: socket.gaierror: [Errno -2] Name or service not known
[   57.816910] cloud-init[1055]: During handling of the above exception, another exception occurred:
[   57.828161] cloud-init[1055]: Traceback (most recent call last):
[   57.829918] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connectionpool.py", line 700, in urlopen
[   57.837491] cloud-init[1055]:     httplib_response = self._make_request(
[   57.845716] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connectionpool.py", line 395, in _make_request
[   57.853730] cloud-init[1055]:     conn.request(method, url, **httplib_request_kw)
[   57.869168] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connection.py", line 234, in request
[   57.873906] cloud-init[1055]:     super(HTTPConnection, self).request(method, url, body=body, headers=headers)
[   57.877292] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 1285, in request
[   57.881003] cloud-init[1055]:     self._send_request(method, url, body, headers, encode_chunked)
[   57.889496] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 1331, in _send_request
[   57.899214] cloud-init[1055]:     self.endheaders(body, encode_chunked=encode_chunked)
[   57.907161] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 1280, in endheaders
[   57.919319] cloud-init[1055]:     self._send_output(message_body, encode_chunked=encode_chunked)
[   57.923734] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 1040, in _send_output
[   57.933038] cloud-init[1055]:     self.send(msg)
[   57.937368] cloud-init[1055]:   File "/usr/lib64/python3.9/http/client.py", line 980, in send
[   57.944880] cloud-init[1055]:     self.connect()
[   57.948647] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connection.py", line 200, in connect
[   57.971171] cloud-init[1055]:     conn = self._new_conn()
[   57.974653] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connection.py", line 181, in _new_conn
[   57.977580] cloud-init[1055]:     raise NewConnectionError(
[   57.979089] cloud-init[1055]: urllib3.exceptions.NewConnectionError: <urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known
[   57.983488] cloud-init[1055]: During handling of the above exception, another exception occurred:
[   57.986264] cloud-init[1055]: Traceback (most recent call last):
[   57.993357] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/requests/adapters.py", line 612, in send
[   57.998528] cloud-init[1055]:     resp = conn.urlopen(
[   58.003447] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/connectionpool.py", line 756, in urlopen
[   58.024263] cloud-init[1055]:     retries = retries.increment(
[   58.025852] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/urllib3/util/retry.py", line 576, in increment
[   58.028553] cloud-init[1055]:     raise MaxRetryError(_pool, url, error or ResponseError(cause))
[   58.030833] cloud-init[1055]: urllib3.exceptions.MaxRetryError: HTTPConnectionPool(host='cloud-init', port=27777): Max retries exceeded with url: /compute.yaml (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known'))
[   58.042591] cloud-init[1055]: During handling of the above exception, another exception occurred:
[   58.049586] cloud-init[1055]: Traceback (most recent call last):
[   58.054297] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/url_helper.py", line 484, in readurl
[   58.062386] cloud-init[1055]:     r = session.request(**req_args)
[   58.067089] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/requests/sessions.py", line 544, in request
[   58.081034] cloud-init[1055]:     resp = self.send(prep, **send_kwargs)
[   58.087251] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/requests/sessions.py", line 657, in send
[   58.107105] cloud-init[1055]:     r = adapter.send(request, **kwargs)
[   58.108800] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/requests/adapters.py", line 689, in send
[   58.111374] cloud-init[1055]:     raise ConnectionError(e, request=request)
[   58.113476] cloud-init[1055]: requests.exceptions.ConnectionError: HTTPConnectionPool(host='cloud-init', port=27777): Max retries exceeded with url: /compute.yaml (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known'))
[   58.128338] cloud-init[1055]: The above exception was the direct cause of the following exception:
[   58.135343] cloud-init[1055]: Traceback (most recent call last):
[   58.154252] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 237, in _do_include
[   58.158325] cloud-init[1055]:     resp = read_file_or_url(
[   58.159860] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/url_helper.py", line 254, in read_file_or_url
[   58.162900] cloud-init[1055]:     return readurl(url, **kwargs)
[   58.165882] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/url_helper.py", line 528, in readurl
[   58.170705] cloud-init[1055]:     raise url_error from e
[   58.177138] cloud-init[1055]: cloudinit.url_helper.UrlError: HTTPConnectionPool(host='cloud-init', port=27777): Max retries exceeded with url: /compute.yaml (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known'))
[   58.206663] cloud-init[1055]: The above exception was the direct cause of the following exception:
[   58.209042] cloud-init[1055]: Traceback (most recent call last):
[   58.210742] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/cmd/main.py", line 922, in status_wrapper
[   58.213793] cloud-init[1055]:     ret = functor(name, args)
[   58.215434] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/cmd/main.py", line 589, in main_init
[   58.224159] cloud-init[1055]:     init.update()
[   58.226857] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/stages.py", line 575, in update
[   58.234746] cloud-init[1055]:     self.datasource.get_vendordata(), "vendordata"
[   58.253660] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/sources/__init__.py", line 657, in get_vendordata
[   58.256856] cloud-init[1055]:     self.vendordata = self.ud_proc.process(self.get_vendordata_raw())
[   58.259253] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 87, in process
[   58.261918] cloud-init[1055]:     self._process_msg(convert_string(blob), accumulating_msg)
[   58.264056] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 158, in _process_msg
[   58.272661] cloud-init[1055]:     self._do_include(payload, append_msg)
[   58.276737] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 263, in _do_include
[   58.299817] cloud-init[1055]:     _handle_error(message, urle)
[   58.303653] cloud-init[1055]:   File "/usr/lib/python3.9/site-packages/cloudinit/user_data.py", line 71, in _handle_error
[   58.306573] cloud-init[1055]:     raise RuntimeError(error_message) from source_exception
[   58.308929] cloud-init[1055]: RuntimeError: HTTPConnectionPool(host='cloud-init', port=27777): Max retries exceeded with url: /compute.yaml (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f9ce3d20640>: Failed to establish a new connection: [Errno -2] Name or service not known')) for url: http://cloud-init:27777/compute.yaml
[   58.323648] cloud-init[1055]: ------------------------------------------------------------
[FAILED] Failed to start Cloud-init: Network Stage.
See 'systemctl status cloud-init.service' for details.
[  OK  ] Reached target Cloud-config availability.
[  OK  ] Reached target Network is Online.
         Starting Cloud-init: Config Stage...
         Starting Crash recovery kernel arming...
         Starting Notify NFS peers of a restart...
         Starting System Logging Service...
         Starting OpenSSH server daemon...
         Starting Permit User Sessions...
[  OK  ] Started Notify NFS peers of a restart.
[  OK  ] Finished Permit User Sessions.
[  OK  ] Started Command Scheduler.
[  OK  ] Started Getty on tty1.
[  OK  ] Started Serial Getty on ttyS0.
[  OK  ] Reached target Login Prompts.
[FAILED] Failed to start OpenSSH server daemon.
See 'systemctl status sshd.service' for details.
[  OK  ] Started System Logging Service.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Record Runlevel Change in UTMP...
[  OK  ] Finished Record Runlevel Change in UTMP.
[   59.007321] cloud-init[1137]: Cloud-init v. 24.4-7.el9_7.1.rocky.0.1 running 'modules:config' at Thu, 02 Apr 2026 16:04:17 +0000. Up 58.98 seconds.
[  OK  ] Finished Cloud-init: Config Stage.
         Starting Cloud-init: Final Stage...
[   59.311937] cloud-init[1141]: Cloud-init v. 24.4-7.el9_7.1.rocky.0.1 running 'modules:final' at Thu, 02 Apr 2026 16:04:17 +0000. Up 59.28 seconds.
[   59.348883] cloud-init[1141]: 2026-04-02 16:04:18,024 - log_util.py[WARNING]: Running module ssh_authkey_fingerprints (<module 'cloudinit.config.cc_ssh_authkey_fingerprints' from '/usr/lib/python3.9/site-packages/cloudinit/config/cc_ssh_authkey_fingerprints.py'>) failed

[   59.387904] cloud-init[1141]: Cloud-init v. 24.4-7.el9_7.1.rocky.0.1 finished at Thu, 02 Apr 2026 16:04:18 +0000. Datasource DataSourceNoCloudNet [seed=cmdline,http://172.16.0.254:8081/cloud-init].  Up 59.38 seconds
[FAILED] Failed to start Cloud-init: Final Stage.
See 'systemctl status cloud-final.service' for details.
[  OK  ] Reached target Cloud-init target.
"""


# on next iteration of doing phase 2 i check this at the beginning:
ochami cloud-init defaults get -F json-pretty
# it was an empty, i did all steps from 2.7 (withs `export DEMO_TOKEN....`): 
# and then again up compute vm, and success, i get access to compute node with
# root over ssh from different terminal session:
"""
danil@localhost /o/w/cloud-init> ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no root@172.16.0.1
Warning: Permanently added '172.16.0.1' (ED25519) to the list of known hosts.
[root@nid001 ~]# 
"""

```

