# OpenShift RHCOS: automatic role selection with DHCP and PXELINUX

Prepared 16 November 2026. Target: a RHEL 9 bastion and x86_64 nodes booting in **legacy BIOS mode**. These are instructions to run on your bastion; nothing has been changed on your lab.

## 1. What this configuration does

Register each node's boot-NIC MAC address in DHCP and link its PXELINUX MAC filename to the appropriate role file. A registered master loads `master`, which installs RHCOS with `master.ign`. A registered worker loads `worker`, which installs RHCOS with `worker.ign`. There is no role-selection menu.

DHCP supplies the node address, router, DNS server, TFTP server and bootloader. PXELINUX selects the role using the MAC link. A VM's display name does not determine its role. All masters share `master.ign`; all workers share `worker.ign` from the same cluster installation.

**Run this installation once per new node.** `coreos.inst.install_dev` authorizes erasing the specified node disk. Configure local disk first in the persistent firmware boot order and use a one-time PXE boot for provisioning. If you boot an armed MAC entry through PXE again, it reinstalls RHCOS. The server does not automatically detect an installed disk or remove a MAC link.

PXE handles RHCOS installation and passes the generated OpenShift configuration. DNS, routing, time synchronization, load balancing, registry access, a valid pull secret, and any required CSR approvals are still necessary for a working OpenShift cluster.

## 2. Your earlier lab settings

The main instructions use the network from `OPENSHIFT_4_22_VMWARE_UPI_OPENHELP_COMPLETE_UPDATED.md`. The attached screenshot is used for the folder/role-file arrangement. Its `192.168.22.1` network is a separate example in section 16.

| Setting | Main configuration |
|---|---|
| Cluster | `ocp.openhelp.net` |
| Provisioning subnet | `172.16.50.0/24` |
| Bastion provisioning NIC | `ens224` |
| Bastion provisioning address | `172.16.50.2/24` |
| TFTP next-server | `172.16.50.2` |
| HTTP base URL | `http://172.16.50.2:8080` |
| Node default gateway | `172.16.50.1` — your VyOS router |
| DNS | `192.168.0.10` — your IPA DNS |
| Existing NTP server | `192.168.0.10`, after verifying it serves NTP |
| HTTP document root | `/var/www/ocp` |
| TFTP root | `/var/lib/tftpboot` |
| Installer output directory in examples | `/root/ocp-install` — replace with your actual installation directory |
| OS installation disk in examples | `/dev/sda` — verify on each new node before arming PXE |

| Node | Reserved IP | MAC from the earlier guide | Role |
|---|---|---|---|
| bootstrap | `172.16.50.62` | `00:0c:29:33:60:60` | bootstrap |
| master01 | `172.16.50.50` | `00:50:56:38:15:01` | master |
| master02 | `172.16.50.51` | `00:50:56:35:f8:e7` | master |
| master03 | `172.16.50.52` | `00:50:56:2e:3f:53` | master |
| worker01 | `172.16.50.53` | `00:50:56:36:09:ec` | worker |
| worker02 | `172.16.50.54` | `00:50:56:20:25:48` | worker |
| worker03 | `172.16.50.55` | **Read the actual NIC MAC from the VM** | worker |

The old guide did not contain worker03's MAC. Its DHCP block below is commented until you replace that value. If you now use different Proxmox VMs, replace **all** old MACs with the observed MACs of their provisioning NICs, in both DHCP and PXELINUX links.

For this PXELINUX configuration, use **SeaBIOS / legacy BIOS** and legacy PXE. `pxelinux.0` is not an OVMF/UEFI bootloader. This guide deliberately does not advertise a BIOS loader to a client that reports a UEFI architecture. Keep the installation and subsequent disk boot in the same firmware mode.

## 3. Directory and URL mapping

Keep `bootstrap`, `master`, `worker`, `default`, and the relative MAC symlinks directly inside `/var/lib/tftpboot/pxelinux.cfg`, as in your screenshot. Do not insert an extra `ocp/` or `pxelinux/` directory into these TFTP paths.

| Purpose | Filesystem location | Requested name or URL |
|---|---|---|
| BIOS bootloader | `/var/lib/tftpboot/pxelinux.0` | DHCP `filename "pxelinux.0"` |
| PXELINUX core module | `/var/lib/tftpboot/ldlinux.c32` | TFTP `ldlinux.c32` |
| Bootstrap role | `/var/lib/tftpboot/pxelinux.cfg/bootstrap` | Loaded through bootstrap MAC symlink |
| Master role | `/var/lib/tftpboot/pxelinux.cfg/master` | Loaded through master MAC symlinks |
| Worker role | `/var/lib/tftpboot/pxelinux.cfg/worker` | Loaded through worker MAC symlinks |
| Fallback | `/var/lib/tftpboot/pxelinux.cfg/default` | Local boot; no installation |
| Live kernel | `/var/lib/tftpboot/rhcos/rhcos-4.22.21-x86_64-live-kernel.x86_64` | TFTP `rhcos/rhcos-4.22.21-x86_64-live-kernel.x86_64` |
| Live initramfs | `/var/lib/tftpboot/rhcos/rhcos-4.22.21-x86_64-live-initramfs.x86_64.img` | TFTP `rhcos/rhcos-4.22.21-x86_64-live-initramfs.x86_64.img` |
| Live rootfs | `/var/www/ocp/rhcos/rhcos-4.22.21-x86_64-live-rootfs.x86_64.img` | `http://172.16.50.2:8080/rhcos/rhcos-4.22.21-x86_64-live-rootfs.x86_64.img` |
| Bootstrap Ignition | `/var/www/ocp/ignition/bootstrap.ign` | `http://172.16.50.2:8080/ignition/bootstrap.ign` |
| Master Ignition | `/var/www/ocp/ignition/master.ign` | `http://172.16.50.2:8080/ignition/master.ign` |
| Worker Ignition | `/var/www/ocp/ignition/worker.ign` | `http://172.16.50.2:8080/ignition/worker.ign` |

The `rhcos-4.22.21-x86_64` prefix is retained **only as the local filename label visible in your screenshot**. The download commands below obtain the actual matching media from your chosen `openshift-install` binary's metadata. They do not assume a release URL from that label. You may change the label, provided you change every reference consistently. Do not reuse mismatched old media just because its filename looks right.

## 4. Prepare the RHEL bastion

Run the following as root on the bastion. First verify that `ens224` really has `172.16.50.2/24` and is attached to the node network. Back up existing files before replacing their contents.

```bash
ip -br address
ip route
systemctl status kea-dhcp4 dhcpd dnsmasq --no-pager
ss -lunp
```

Your earlier lab used **Kea**. To replace it with ISC DHCP, do not leave both services running on the same node segment. The following stops the old Kea service on this bastion; leave your VyOS routing/SNAT in place. Also keep VMware/Proxmox/other DHCP services disabled on this isolated provisioning segment.

```bash
systemctl disable --now kea-dhcp4
dnf install -y dhcp-server tftp-server tftp syslinux-tftpboot httpd jq curl policycoreutils-python-utils
mkdir -p /var/lib/tftpboot/pxelinux.cfg
mkdir -p /var/lib/tftpboot/rhcos
mkdir -p /var/www/ocp/rhcos
mkdir -p /var/www/ocp/ignition
mkdir -p /root/ocp-pxe-media
mkdir -p /etc/systemd/system/dhcpd.service.d
mkdir -p /etc/systemd/system/tftp.service.d
```

Use the matching bootloader/core module from the installed package. `rpm -ql` avoids depending on whether your package stores them under `/tftpboot` or another source directory.

```bash
rpm -ql syslinux-tftpboot
PXE_LOADER=$(rpm -ql syslinux-tftpboot | grep '/pxelinux.0$' | head -n 1)
PXE_MODULE=$(rpm -ql syslinux-tftpboot | grep '/ldlinux.c32$' | head -n 1)
test -f "$PXE_LOADER"
test -f "$PXE_MODULE"
install -m 0644 "$PXE_LOADER" /var/lib/tftpboot/pxelinux.0
install -m 0644 "$PXE_MODULE" /var/lib/tftpboot/ldlinux.c32
```

If either `test` fails, stop and correct the package source paths. A menu module is not required: these role files boot their default entry directly, without `UI menu.c32`.

## 5. Obtain matching RHCOS files

Use the same selected OpenShift installer binary used for the installation's manifests and Ignition files. If you already have its verified, matching kernel/initramfs/rootfs artifacts, copy those instead of downloading them again.

```bash
openshift-install version
openshift-install coreos print-stream-json > /root/ocp-pxe-media/stream.json
jq -e '.architectures.x86_64.artifacts.metal.formats.pxe' /root/ocp-pxe-media/stream.json
cd /root/ocp-pxe-media
curl --fail --location --retry 3 --output rhcos-4.22.21-x86_64-live-kernel.x86_64 "$(jq -er '.architectures.x86_64.artifacts.metal.formats.pxe.kernel.location' stream.json)"
curl --fail --location --retry 3 --output rhcos-4.22.21-x86_64-live-initramfs.x86_64.img "$(jq -er '.architectures.x86_64.artifacts.metal.formats.pxe.initramfs.location' stream.json)"
curl --fail --location --retry 3 --output rhcos-4.22.21-x86_64-live-rootfs.x86_64.img "$(jq -er '.architectures.x86_64.artifacts.metal.formats.pxe.rootfs.location' stream.json)"
jq -er '.architectures.x86_64.artifacts.metal.formats.pxe | (.kernel.sha256 + "  rhcos-4.22.21-x86_64-live-kernel.x86_64"), (.initramfs.sha256 + "  rhcos-4.22.21-x86_64-live-initramfs.x86_64.img"), (.rootfs.sha256 + "  rhcos-4.22.21-x86_64-live-rootfs.x86_64.img")' stream.json > SHA256SUMS
sha256sum --check SHA256SUMS
```

All three checksum results must say `OK` before copying the files into service directories.

```bash
install -m 0644 rhcos-4.22.21-x86_64-live-kernel.x86_64 /var/lib/tftpboot/rhcos/
install -m 0644 rhcos-4.22.21-x86_64-live-initramfs.x86_64.img /var/lib/tftpboot/rhcos/
install -m 0644 rhcos-4.22.21-x86_64-live-rootfs.x86_64.img /var/www/ocp/rhcos/
```

The rootfs is served by HTTP. The kernel and initramfs are served by TFTP, matching the relative `KERNEL` and `initrd=` paths in the screenshot.

## 6. Place the OpenShift Ignition files

Use `bootstrap.ign`, `master.ign`, and `worker.ign` from the **same cluster installation**. For a fresh installation only, generate them using your prepared `install-config.yaml` and the existing manifest procedure. Use fresh files within Red Hat's recommended installation window of 12 hours. Do not generate a new unrelated cluster's Ignition files to add a worker to an existing cluster.

If your output directory differs, replace `/root/ocp-install` below. Do not publish the installation directory, `auth/kubeconfig`, kubeadmin password, private SSH key, or pull-secret file as the HTTP document root.

```bash
install -m 0644 /root/ocp-install/bootstrap.ign /var/www/ocp/ignition/bootstrap.ign
install -m 0644 /root/ocp-install/master.ign /var/www/ocp/ignition/master.ign
install -m 0644 /root/ocp-install/worker.ign /var/www/ocp/ignition/worker.ign
jq -e . /var/www/ocp/ignition/bootstrap.ign > /dev/null
jq -e . /var/www/ocp/ignition/master.ign > /dev/null
jq -e . /var/www/ocp/ignition/worker.ign > /dev/null
```

`jq` checks that the files are JSON; it does not validate Ignition's schema, certificate age, or cluster identity. Serve these provisioning files only to the intended lab network.

## 7. HTTP on port 8080

If your current Apache configuration already serves `/var/www/ocp` on `172.16.50.2:8080`, keep it and create the subdirectories above. Do not add a second overlapping `Listen` directive. The configuration below is for setting up that listener when it does not already exist.

Inspect the active configuration first:

```bash
grep -R -n '^[[:space:]]*Listen\|DocumentRoot\|VirtualHost' /etc/httpd/conf /etc/httpd/conf.d
ss -lntp
```

If this bastion's HAProxy owns port 80, comment Apache's default `Listen 80` in `/etc/httpd/conf/httpd.conf`. Also avoid an enabled Apache SSL listener on port 443 if HAProxy owns it. Keep only one Apache listener definition for `172.16.50.2:8080`.

Create `/etc/httpd/conf.d/ocp-pxe.conf` with:

```apache
Listen 172.16.50.2:8080

<VirtualHost 172.16.50.2:8080>
    ServerName bastion.ocp.openhelp.net
    DocumentRoot "/var/www/ocp"

    <Directory "/var/www/ocp">
        Options -Indexes
        AllowOverride None
        Require ip 127.0.0.1 172.16.50.0/24
    </Directory>

    AddType application/json .ign
    ErrorLog logs/ocp-pxe-error.log
    CustomLog logs/ocp-pxe-access.log combined
</VirtualHost>
```

Correct the SELinux port assignment if needed. On stock RHEL policies, 8080 is often already assigned to `http_cache_port_t`; in that case use `-m`, not `-a`. Inspect before making the one applicable change.

```bash
semanage port -l | grep -E 'http_port_t|http_cache_port_t'
```

If 8080 exists under `http_cache_port_t`:

```bash
semanage port -m -t http_port_t -p tcp 8080
```

If 8080 is not defined, use this instead:

```bash
semanage port -a -t http_port_t -p tcp 8080
```

If it is already `http_port_t`, neither command is needed. Then restore labels and validate Apache:

```bash
restorecon -RFv /var/www/ocp
httpd -t
```

## 8. Complete `/etc/dhcp/dhcpd.conf`

This serves only the listed reservations; no dynamic pool is needed. DHCP returns `pxelinux.0` to legacy BIOS clients. Masters and workers receive the **same bootloader filename**; their separate role selection happens through PXELINUX MAC links.

```conf
authoritative;
ddns-update-style none;
default-lease-time 3600;
max-lease-time 86400;
allow booting;
allow bootp;

option architecture-type code 93 = unsigned integer 16;

subnet 172.16.50.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option broadcast-address 172.16.50.255;
    option routers 172.16.50.1;
    option domain-name-servers 192.168.0.10;
    option domain-name "ocp.openhelp.net";
    option domain-search "ocp.openhelp.net";

    next-server 172.16.50.2;

    # Legacy BIOS only. UEFI clients need an EFI loader and separate config.
    if exists architecture-type {
        if option architecture-type = 00:00 {
            filename "pxelinux.0";
        }
    } else {
        filename "pxelinux.0";
    }

    host bootstrap {
        hardware ethernet 00:0c:29:33:60:60;
        fixed-address 172.16.50.62;
        option host-name "bootstrap.ocp.openhelp.net";
    }

    host master01 {
        hardware ethernet 00:50:56:38:15:01;
        fixed-address 172.16.50.50;
        option host-name "master01.ocp.openhelp.net";
    }

    host master02 {
        hardware ethernet 00:50:56:35:f8:e7;
        fixed-address 172.16.50.51;
        option host-name "master02.ocp.openhelp.net";
    }

    host master03 {
        hardware ethernet 00:50:56:2e:3f:53;
        fixed-address 172.16.50.52;
        option host-name "master03.ocp.openhelp.net";
    }

    host worker01 {
        hardware ethernet 00:50:56:36:09:ec;
        fixed-address 172.16.50.53;
        option host-name "worker01.ocp.openhelp.net";
    }

    host worker02 {
        hardware ethernet 00:50:56:20:25:48;
        fixed-address 172.16.50.54;
        option host-name "worker02.ocp.openhelp.net";
    }

    # Replace the MAC and uncomment this entire host block for worker03.
    # host worker03 {
    #     hardware ethernet REPLACE_WITH_WORKER03_MAC;
    #     fixed-address 172.16.50.55;
    #     option host-name "worker03.ocp.openhelp.net";
    # }
}
```

Do not advertise the bastion as gateway: your earlier lab routes nodes through VyOS at `172.16.50.1`. Do not advertise the gateway as DNS: your DNS is IPA at `192.168.0.10`. DHCP reservations do not create DNS records because dynamic DNS updates are disabled here; retain the A/PTR records in IPA.

## 9. `dhcpd.service` restricted to `ens224`

Keep the package-provided service. Create `/etc/systemd/system/dhcpd.service.d/override.conf` with the following contents; this overrides only its command and retains the vendor's user, runtime and other service settings.

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/dhcpd -4 -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid ens224
```

The empty `ExecStart=` is required before replacing the vendor command. If the current bastion is different, replace `ens224` with the actual NIC connected to the node network. Do not edit `/usr/lib/systemd/system/dhcpd.service`.

## 10. TFTP root and socket activation

Create `/etc/systemd/system/tftp.service.d/override.conf` with:

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/in.tftpd -s /var/lib/tftpboot
```

This pins the TFTP root to the folder containing `pxelinux.0`, `pxelinux.cfg`, and `rhcos`. Enable `tftp.socket`; an idle `tftp.service` can stop between requests because it is socket-activated.

Relative symlinks inside `pxelinux.cfg` must point to files within this TFTP tree. Avoid absolute symlinks outside it: `in.tftpd -s` serves a restricted root.

## 11. PXELINUX role files

Verify the install disk on each new machine first. `/dev/sda` is common with a SCSI disk; VirtIO block disks may be `/dev/vda`, and NVMe disks may be `/dev/nvme0n1`. These names are examples, not proof of your disk layout. Use `lsblk -o NAME,SIZE,MODEL,TYPE` in a live environment to identify the intended OS disk. If nodes use different disk names, give those nodes separate PXE files or role variants with the correct target disk.

Each `APPEND` statement below must be **one physical line**. It may wrap visually in an editor; do not insert newlines or shell backslashes inside it.

File `/var/lib/tftpboot/pxelinux.cfg/bootstrap`:

```conf
DEFAULT bootstrap
PROMPT 0
TIMEOUT 0
LABEL bootstrap
    KERNEL rhcos/rhcos-4.22.21-x86_64-live-kernel.x86_64
    APPEND initrd=rhcos/rhcos-4.22.21-x86_64-live-initramfs.x86_64.img ip=dhcp rd.neednet=1 coreos.live.rootfs_url=http://172.16.50.2:8080/rhcos/rhcos-4.22.21-x86_64-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.2:8080/ignition/bootstrap.ign
```

File `/var/lib/tftpboot/pxelinux.cfg/master`:

```conf
DEFAULT master
PROMPT 0
TIMEOUT 0
LABEL master
    KERNEL rhcos/rhcos-4.22.21-x86_64-live-kernel.x86_64
    APPEND initrd=rhcos/rhcos-4.22.21-x86_64-live-initramfs.x86_64.img ip=dhcp rd.neednet=1 coreos.live.rootfs_url=http://172.16.50.2:8080/rhcos/rhcos-4.22.21-x86_64-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.2:8080/ignition/master.ign
```

File `/var/lib/tftpboot/pxelinux.cfg/worker`:

```conf
DEFAULT worker
PROMPT 0
TIMEOUT 0
LABEL worker
    KERNEL rhcos/rhcos-4.22.21-x86_64-live-kernel.x86_64
    APPEND initrd=rhcos/rhcos-4.22.21-x86_64-live-initramfs.x86_64.img ip=dhcp rd.neednet=1 coreos.live.rootfs_url=http://172.16.50.2:8080/rhcos/rhcos-4.22.21-x86_64-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.2:8080/ignition/worker.ign
```

With `PROMPT 0`, no `UI` menu, and a valid `DEFAULT`, these boot immediately. `TIMEOUT 0` disables the timeout mechanism; it does not mean a menu must appear.

For installation with a single provisioning NIC, `ip=dhcp` is sufficient. If a node has several active NICs, replace it with a NIC-specific setting such as `ip=ens18:dhcp`, using the NIC name actually seen by RHCOS. Keep the same reserved MAC/network for the installed system.

Use `coreos.inst.ignition_url` for the installed node's `bootstrap.ign`, `master.ign`, or `worker.ign`. `ignition.config.url` configures the live environment and is a different setting.

## 12. Default fallback and MAC links

File `/var/lib/tftpboot/pxelinux.cfg/default`:

```conf
DEFAULT local
PROMPT 0
TIMEOUT 1
LABEL local
    LOCALBOOT 0
```

This fallback has no installation entry. A registered host can also use it after you disarm that host's MAC link. Unknown MACs have no reservation in the main DHCP configuration, so they ordinarily receive no address from this server in the first place.

Create these relative links using the earlier lab MACs:

```bash
cd /var/lib/tftpboot/pxelinux.cfg
ln -sfn bootstrap 01-00-0c-29-33-60-60
ln -sfn master 01-00-50-56-38-15-01
ln -sfn master 01-00-50-56-35-f8-e7
ln -sfn master 01-00-50-56-2e-3f-53
ln -sfn worker 01-00-50-56-36-09-ec
ln -sfn worker 01-00-50-56-20-25-48
ls -l /var/lib/tftpboot/pxelinux.cfg
```

For worker03, first read its actual MAC. For a MAC such as `aa:bb:cc:dd:ee:ff`, the exact command pattern is:

```bash
ln -sfn worker /var/lib/tftpboot/pxelinux.cfg/01-aa-bb-cc-dd-ee-ff
```

Replace that example filename with the real lowercase MAC, and put the same real MAC into worker03's DHCP host declaration. The initial `01-` identifies Ethernet; it is not part of the hardware MAC. Use hyphens in the PXELINUX filename and colons in `hardware ethernet`.

PXELINUX can try a UUID configuration before the MAC configuration. Check for old UUID-specific files or DHCP configuration-file overrides if a machine appears to ignore its MAC link. The main configuration does not set those overrides.

## 13. Permissions, firewall, validation, service startup

```bash
chmod 0755 /var/lib/tftpboot /var/lib/tftpboot/pxelinux.cfg /var/lib/tftpboot/rhcos
chmod 0755 /var/www/ocp /var/www/ocp/rhcos /var/www/ocp/ignition
chmod 0644 /var/lib/tftpboot/pxelinux.0 /var/lib/tftpboot/ldlinux.c32
chmod 0644 /var/lib/tftpboot/pxelinux.cfg/bootstrap /var/lib/tftpboot/pxelinux.cfg/master /var/lib/tftpboot/pxelinux.cfg/worker /var/lib/tftpboot/pxelinux.cfg/default
restorecon -RFv /var/lib/tftpboot /var/www/ocp /etc/dhcp /etc/systemd/system/dhcpd.service.d /etc/systemd/system/tftp.service.d
firewall-cmd --get-active-zones
firewall-cmd --get-zone-of-interface=ens224
```

Add the following services to the **existing zone of `ens224`**. The example below uses `public`: replace `public` with the actual returned zone. If no explicit zone is assigned, obtain the default zone with `firewall-cmd --get-default-zone`. Do not move the interface into a different zone and accidentally drop existing OpenShift service rules.

```bash
firewall-cmd --permanent --zone=public --add-service=dhcp
firewall-cmd --permanent --zone=public --add-service=tftp
firewall-cmd --permanent --zone=public --add-port=8080/tcp
firewall-cmd --reload
dhcpd -t -cf /etc/dhcp/dhcpd.conf
httpd -t
systemctl daemon-reload
systemctl enable --now dhcpd
systemctl enable --now tftp.socket
systemctl enable --now httpd
systemctl restart dhcpd httpd
systemctl status dhcpd tftp.socket httpd --no-pager
systemctl cat dhcpd
systemctl cat tftp.service
```

Only continue when the DHCP and Apache configuration tests pass. If TFTP was already active when you changed its override, restart its socket/service during a provisioning maintenance window, before starting any nodes. Retain the rest of your existing OpenShift firewall rules; these three openings cover the additional provisioning services only.

Verify the web files without dumping Ignition contents into the terminal:

```bash
curl --fail --silent --show-error --output /dev/null http://172.16.50.2:8080/ignition/bootstrap.ign
curl --fail --silent --show-error --output /dev/null http://172.16.50.2:8080/ignition/master.ign
curl --fail --silent --show-error --output /dev/null http://172.16.50.2:8080/ignition/worker.ign
curl --fail --head http://172.16.50.2:8080/rhcos/rhcos-4.22.21-x86_64-live-rootfs.x86_64.img
curl --fail --silent --show-error --output /tmp/pxelinux.0.check tftp://172.16.50.2/pxelinux.0
cmp /var/lib/tftpboot/pxelinux.0 /tmp/pxelinux.0.check
curl --fail --silent --show-error --output /tmp/master-pxe.check tftp://172.16.50.2/pxelinux.cfg/01-00-50-56-38-15-01
cmp /var/lib/tftpboot/pxelinux.cfg/master /tmp/master-pxe.check
```

Repeat the HTTP checks from a machine on the node network. If your curl build lacks TFTP support, use the installed `tftp` client interactively: connect to `172.16.50.2`, enter `binary`, then `get pxelinux.0`, then `quit`.

Useful live logs:

```bash
journalctl -u dhcpd -f
journalctl -u tftp.service -f
tail -f /var/log/httpd/ocp-pxe-access.log
```

Run those follow commands in separate terminals. TFTP's default log verbosity varies; use its journal for errors rather than expecting every successful GET to be logged. DHCP must show the expected reservation, not a lease supplied by another DHCP server.

## 14. Boot and complete the cluster

1. Keep local disk first in the persistent boot order; select the provisioning NIC for a one-time PXE boot. Use SeaBIOS/legacy PXE for these files.
2. For a new cluster, boot bootstrap and all three masters using their mapped role files. Workers can be installed as well; their final provisioning waits for the machine-config/API services to become available.
3. RHCOS downloads its live rootfs, runs the disk installer, embeds the appropriate Ignition configuration and reboots. Allow the reboot to reach the installed local disk.
4. Ignition applies on the installed system's first boot. Existing DNS, HAProxy, VyOS and registry connectivity allow OpenShift bootstrapping to proceed.
5. Monitor bootstrap completion from the original installer directory:

```bash
openshift-install wait-for bootstrap-complete --dir /root/ocp-install --log-level=info
```

6. Once that reports success, remove bootstrap from the API/MCS load-balancer backends and shut down the temporary bootstrap VM, using your existing HAProxy procedure.
7. Check CSRs and approve only those belonging to the expected nodes. Recheck later for serving CSRs if required; approval is not necessarily a single pass.

```bash
export KUBECONFIG=/root/ocp-install/auth/kubeconfig
oc get csr
oc describe csr CSR_NAME
oc adm certificate approve CSR_NAME
oc get nodes -o wide
oc get clusteroperators
openshift-install wait-for install-complete --dir /root/ocp-install --log-level=info
```

`CSR_NAME` is a placeholder; substitute each actual CSR name after verifying its requesting node. Provisioning does not require an interactive role menu, but this configuration does not implement automatic CSR approval.

The previous lab's infrastructure must still satisfy these endpoints:

| Check | Expected earlier-lab value |
|---|---|
| `api.ocp.openhelp.net` | `192.168.0.49`, reachable by intended clients/nodes |
| `api-int.ocp.openhelp.net` | `172.16.50.49` |
| API load-balancer port | TCP 6443 to bootstrap/masters during bootstrap |
| Machine config load-balancer port | TCP 22623 to bootstrap/masters during bootstrap |
| Apps wildcard | `*.apps.ocp.openhelp.net` resolves to `192.168.0.48` |
| App ingress ports | TCP 80/443 to the ingress routers |
| Node forward/reverse DNS | Node IPs and the same node names supplied by DHCP |
| DNS/time/registry egress | Through the existing VyOS gateway at `172.16.50.1` |

No new IP masquerading on the bastion is required for this earlier design: VyOS already supplies egress. If you deliberately change the gateway to the bastion, configure routing/NAT for that new design separately and update DHCP consistently.

## 15. Disarm installation after success

Local-disk-first boot is the primary safeguard. After a node has completed its RHCOS disk installation, you can also repoint its MAC link to `default` so that accidental PXE boot does not reinstall it. Do not change the shared role file, because other machines may still need it.

Example for the earlier worker01:

```bash
ln -sfn default /var/lib/tftpboot/pxelinux.cfg/01-00-50-56-36-09-ec
```

This preserves the MAC-link structure. Repoint only the nodes whose disk installation has completed. To deliberately reinstall worker01 later, after verifying that erasing its disk is intended:

```bash
ln -sfn worker /var/lib/tftpboot/pxelinux.cfg/01-00-50-56-36-09-ec
```

Then use one-time PXE boot again. A MAC-to-role symlink is an installation selection, not a persistent OpenShift role controller.

## 16. Separate example: the screenshot's `192.168.22.1` network

Use this section **instead of** the earlier-lab addresses only if you are building a separate lab matching that screenshot. Do not apply it as an address change to an existing OpenHelp cluster.

The following seven MAC spellings are transcribed from the screenshot and must be checked against the actual VMs. The node IPs and `ocp.example.test` domain below are **explicit example choices**; they are not claimed as your old values. The screenshot's domain text is not clear enough to use as authoritative cluster configuration. Replace the example domain with the domain used by that lab's actual generated Ignition files and DNS.

| Node | Example IP | Screenshot MAC | Link target |
|---|---|---|---|
| bootstrap | `192.168.22.200` | `bc:24:11:22:22:c8` | bootstrap |
| master-1 | `192.168.22.201` | `bc:24:11:22:22:c9` | master |
| master-2 | `192.168.22.202` | `bc:24:11:22:22:ca` | master |
| master-3 | `192.168.22.203` | `bc:24:11:22:22:cb` | master |
| worker-1 | `192.168.22.211` | `bc:24:11:22:22:c3` | worker |
| worker-2 | `192.168.22.212` | `bc:24:11:22:22:c4` | worker |
| worker-3 | `192.168.22.213` | `bc:24:11:22:22:c5` | worker |

This example assumes `192.168.22.1` actually provides DNS and routing as well as HTTP/TFTP. If it only provides HTTP/TFTP, use the real DNS and gateway in their respective DHCP options. The screenshot alone does not prove that DNS or IP masquerading is configured.

Complete replacement `/etc/dhcp/dhcpd.conf` for this **separate example**:

```conf
authoritative;
ddns-update-style none;
default-lease-time 3600;
max-lease-time 86400;
allow booting;
allow bootp;

option architecture-type code 93 = unsigned integer 16;

subnet 192.168.22.0 netmask 255.255.255.0 {
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.22.255;
    option routers 192.168.22.1;
    option domain-name-servers 192.168.22.1;
    option domain-name "ocp.example.test";
    option domain-search "ocp.example.test";
    next-server 192.168.22.1;

    if exists architecture-type {
        if option architecture-type = 00:00 {
            filename "pxelinux.0";
        }
    } else {
        filename "pxelinux.0";
    }

    host bootstrap {
        hardware ethernet bc:24:11:22:22:c8;
        fixed-address 192.168.22.200;
        option host-name "bootstrap.ocp.example.test";
    }
    host master-1 {
        hardware ethernet bc:24:11:22:22:c9;
        fixed-address 192.168.22.201;
        option host-name "master-1.ocp.example.test";
    }
    host master-2 {
        hardware ethernet bc:24:11:22:22:ca;
        fixed-address 192.168.22.202;
        option host-name "master-2.ocp.example.test";
    }
    host master-3 {
        hardware ethernet bc:24:11:22:22:cb;
        fixed-address 192.168.22.203;
        option host-name "master-3.ocp.example.test";
    }
    host worker-1 {
        hardware ethernet bc:24:11:22:22:c3;
        fixed-address 192.168.22.211;
        option host-name "worker-1.ocp.example.test";
    }
    host worker-2 {
        hardware ethernet bc:24:11:22:22:c4;
        fixed-address 192.168.22.212;
        option host-name "worker-2.ocp.example.test";
    }
    host worker-3 {
        hardware ethernet bc:24:11:22:22:c5;
        fixed-address 192.168.22.213;
        option host-name "worker-3.ocp.example.test";
    }
}
```

Make these coordinated changes to the main procedure for this separate example:

| Main configuration | Separate screenshot example |
|---|---|
| `172.16.50.2` in HTTP listeners and all three role files | `192.168.22.1` |
| HTTP `Require ip 127.0.0.1 172.16.50.0/24` | `Require ip 127.0.0.1 192.168.22.0/24` |
| `ens224` in DHCP service override/firewall inspection | Actual NIC with `192.168.22.1/24` |
| Earlier-lab DHCP config | Complete separate DHCP config immediately above |
| Earlier-lab MAC links | Screenshot links immediately below |
| `ocp.openhelp.net` and its generated Ignition files | Actual separate cluster domain, DNS records, and its own generated Ignition files |

The TFTP and HTTP folder names stay the same. Create only the MAC links belonging to the selected lab:

```bash
cd /var/lib/tftpboot/pxelinux.cfg
ln -sfn bootstrap 01-bc-24-11-22-22-c8
ln -sfn master 01-bc-24-11-22-22-c9
ln -sfn master 01-bc-24-11-22-22-ca
ln -sfn master 01-bc-24-11-22-22-cb
ln -sfn worker 01-bc-24-11-22-22-c3
ln -sfn worker 01-bc-24-11-22-22-c4
ln -sfn worker 01-bc-24-11-22-22-c5
```

For example, the worker role file for that separate network is:

```conf
DEFAULT worker
PROMPT 0
TIMEOUT 0
LABEL worker
    KERNEL rhcos/rhcos-4.22.21-x86_64-live-kernel.x86_64
    APPEND initrd=rhcos/rhcos-4.22.21-x86_64-live-initramfs.x86_64.img ip=dhcp rd.neednet=1 coreos.live.rootfs_url=http://192.168.22.1:8080/rhcos/rhcos-4.22.21-x86_64-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.22.1:8080/ignition/worker.ign
```

Apply the same HTTP-address substitution in `bootstrap` and `master`. Do not combine the two DHCP subnets or their old/new Ignition files by accident.

## 17. Troubleshooting by stage

| Symptom | Check |
|---|---|
| No DHCP address | Correct NIC/bridge/VLAN; `dhcpd` running on the correct interface; actual MAC matches a host reservation; no competing DHCP server |
| Address received, no PXE boot file | Legacy BIOS enabled; DHCP architecture option; `next-server`; TFTP reachability |
| `ldlinux.c32` missing | Matching `ldlinux.c32` next to `pxelinux.0` under the actual TFTP root |
| Wrong role selected | MAC link target; UUID-specific config; stale DHCP PXELINUX config-file override |
| Kernel or initramfs not found | Relative `rhcos/...` path under `/var/lib/tftpboot`, filename spelling, permissions, SELinux labels |
| Live rootfs download fails | HTTP 8080 listener, firewall, URL and `httpd_sys_content_t` labels |
| Ignition HTTP 403/404 | Correct `/var/www/ocp/ignition` location, Apache document root/access rule, permissions and exact filename |
| Installer boots but never writes disk | `coreos.inst.install_dev` exists and is the intended disk; inspect `journalctl -b -u coreos-installer` in the live environment |
| Installs repeatedly after reboot | Persistent PXE-first boot or armed MAC entry; boot local disk and disarm that host's MAC link |
| Installed node fails to fetch machine configuration | `api-int` DNS, gateway, TCP 22623 load balancer, correct cluster Ignition and valid certificates |
| Node stays NotReady / does not register | DNS/time/registry reachability, TCP 6443/API, pending verified CSRs, kubelet logs |
| HTTP starts with address-in-use error | Duplicate Apache `Listen`, HAProxy conflict on 80/443, or another service on 8080 |

## 18. References and validation limits

The user's earlier guide was read for the actual OpenHelp network plan, DNS, router, bastion NIC and existing reservations. The attached screenshot was inspected for the requested PXELINUX folder and role-file arrangement.

Primary references used for the installation mechanism and service setup:

- [Red Hat OpenShift 4.22: user-provisioned bare-metal installation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_bare_metal/user-provisioned-infrastructure)
- [Red Hat RHEL 9: preparing a PXE installation source](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/automatically_installing_rhel/preparing-for-a-network-install_rhel-installer)
- [Red Hat RHEL 9: providing DHCP services](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/providing-dhcp-services_networking-infrastructure-services)
- [Syslinux: PXELINUX configuration lookup and working directory](https://wiki.syslinux.org/wiki/index.php?title=PXELINUX)
- [Syslinux: DEFAULT, PROMPT, TIMEOUT, APPEND and LOCALBOOT](https://wiki.syslinux.org/wiki/index.php?title=Config)
- [CoreOS Installer: getting started and automatic installation](https://coreos.github.io/coreos-installer/getting-started/)
- [Red Hat RHEL 9: deploying web servers and reverse proxies](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/deploying_web_servers_and_reverse_proxies/index)
- [Red Hat RHEL 9: Using SELinux](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/pdf/using_selinux/index)
- [Red Hat RHEL 9: firewalld configuration](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld_firewall-packet-filters)

The [Kifarunix guide supplied by the user](https://kifarunix.com/how-to-deploy-multinode-openshift-cluster-using-upi/) was reviewed for the requested general workflow. Its Ubuntu service names and different directory layout are not the RHEL configuration used here.

Role/URL/path consistency and shell-block syntax can be checked locally. The actual RHEL services, guest disk identities, DHCP transactions, firmware boot order and OpenShift installation must be validated in your lab using the commands above. No successful installation is claimed by producing this guide.
