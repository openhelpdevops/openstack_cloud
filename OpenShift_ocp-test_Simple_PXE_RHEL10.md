# Simple RHEL 10 PXE steps for `ocp-test.kubinfra.com`

This is the PXE portion of your existing OpenShift UPI installation. Continue using the other steps from your bastion guide. These instructions use **manual edits and individual commands**. They use the same layout as your screenshot: three role files named `bootstrap`, `master`, and `worker`, with one MAC-specific link per VM.

Run bastion commands as `root`. The bastion has `ens19 = 172.16.50.2/24`; its public NIC is `ens18 = 192.168.0.40/24`. IPA at **192.168.0.30** supplies DNS (and NTP after you enable its time service). The seven node VMs must share the isolated Proxmox bridge with `ens19`. This is **BIOS/SeaBIOS PXE** for x86_64, using RHEL 10 **Kea DHCP**, TFTP/PXELINUX, and the existing Apache listener on `172.16.50.2:8080`. Do not connect the DHCP-serving bridge to your home LAN.

| VM | IP | PXE role | Ignition |
|---|---|---|---|
| bootstrap | 172.16.50.62 | bootstrap | bootstrap.ign |
| master01 | 172.16.50.50 | master | master.ign |
| master02 | 172.16.50.51 | master | master.ign |
| master03 | 172.16.50.52 | master | master.ign |
| worker01 | 172.16.50.53 | worker | worker.ign |
| worker02 | 172.16.50.54 | worker | worker.ign |
| worker03 | 172.16.50.55 | worker | worker.ign |

**Before proceeding:** in Proxmox record the real MAC for each VM from **VM → Hardware → Network Device**. Confirm every VM is **SeaBIOS** and that the installation disk is `/dev/sda`. The examples below assume `/dev/sda` for **all seven**; the install will overwrite that disk. If a VM instead has `/dev/vda` or a different disk, change the affected role configuration before PXE booting. All masters share one role file; all workers share one role file, so each group must have the same target device path.

## Step 1 — Make sure the existing HTTP server works

Your file `/var/www/html/ocp/health.txt` has URL **`/ocp/health.txt`**. Check the NIC, the Apache service, the listener, and the correct URL:

```bash
ip -br -4 address show ens19
systemctl status httpd --no-pager -l
ss -lntp | grep ':8080'
curl -fsS http://172.16.50.2:8080/ocp/health.txt
```

The last command must print `ocp asset server ready`. If Apache is already running but port 8080 is absent, check its listener setting:

```bash
grep -RniE '^[[:space:]]*Listen[[:space:]]' /etc/httpd/conf/httpd.conf /etc/httpd/conf.d/
```

In `/etc/httpd/conf/httpd.conf`, comment out the default `Listen 80` so it becomes `#Listen 80`; HAProxy uses port 80 in the main guide. Edit the private listener file and ensure it contains:

```bash
vi /etc/httpd/conf/httpd.conf
vi /etc/httpd/conf.d/00-ocp-install.conf
```

```apache
Listen 172.16.50.2:8080
ServerName bastion.ocp-test.kubinfra.com

<Directory "/var/www/html/ocp">
    Options -Indexes
    AllowOverride None
    Require ip 172.16.50.0/24
</Directory>
```

Apply the change and repeat the health test:

```bash
httpd -t
systemctl restart httpd
ss -lntp | grep ':8080'
curl -fsS http://172.16.50.2:8080/ocp/health.txt
```

If `systemctl restart httpd` fails, read `journalctl -u httpd -n 40 --no-pager`. **Do not start PXE** until this HTTP gate succeeds.

## Step 2 — Install Kea, TFTP, and the BIOS PXE loader

Your RHEL 10 BaseOS and AppStream repositories must be enabled as in the main guide. Create the backup directory first; it was missing in the earlier `cp` error. Then install packages:

```bash
mkdir -p /root/ocp-config-backup
dnf install -y kea tftp-server syslinux-tftpboot curl bind-utils
rpm -q kea tftp-server syslinux-tftpboot
rpm -ql syslinux-tftpboot | grep -E '/(pxelinux\.0|ldlinux\.c32)$'
```

The final command must print **two real source paths**. On RHEL 10 they commonly show `/tftpboot/pxelinux.0` and `/tftpboot/ldlinux.c32`. Only when those exact files are shown, run:

```bash
mkdir -p /var/lib/tftpboot/pxelinux.cfg
install -m 0644 /tftpboot/pxelinux.0 /var/lib/tftpboot/pxelinux.0
install -m 0644 /tftpboot/ldlinux.c32 /var/lib/tftpboot/ldlinux.c32
restorecon -RFv /var/lib/tftpboot
systemctl enable --now tftp.socket
ls -l /var/lib/tftpboot/pxelinux.0 /var/lib/tftpboot/ldlinux.c32
```

If RPM lists different source paths, use those paths in the two `install` commands. If it prints no paths, stop and check `dnf repolist` and `dnf provides '*/pxelinux.0'`; the previously attempted copy failed because `/tftpboot/` did not contain the package files.

## Step 3 — Check the RHCOS and Ignition URLs

Use the **matching** RHCOS PXE images already downloaded from your `openshift-install coreos print-stream-json` step. The files must be exactly:

```text
[root@bastion ~]# ll /var/www/html/ocp/rhcos/
total 1426256
-rw-r--r--. 1 root root  109518236 Nov 18 15:46 rhcos-live-initramfs.x86_64.img
-rw-r--r--. 1 root root   15190824 Nov 18 15:46 rhcos-live-kernel.x86_64
-rw-r--r--. 1 root root 1320582656 Nov 18 15:47 rhcos-live-rootfs.x86_64.img
[root@bastion ~]# ll /var/www/html/ocp/ignition
total 320
-rw-r--r--. 1 root root 317634 Nov 18 17:13 bootstrap.ign
-rw-r--r--. 1 root root   1718 Nov 18 17:13 master.ign
-rw-r--r--. 1 root root   1718 Nov 18 17:13 worker.ign

```

Check each HTTP location. Every response must be **200 OK**:

```bash
curl -fI http://172.16.50.2:8080/ocp/rhcos/rhcos-live-kernel.x86_64
curl -fI http://172.16.50.2:8080/ocp/rhcos/rhcos-live-initramfs.x86_64.img
curl -fI http://172.16.50.2:8080/ocp/rhcos/rhcos-live-rootfs.x86_64.img
curl -fI http://172.16.50.2:8080/ocp/ignition/bootstrap.ign
curl -fI http://172.16.50.2:8080/ocp/ignition/master.ign
curl -fI http://172.16.50.2:8080/ocp/ignition/worker.ign
```

The three Ignition files must be generated for **this** `ocp-test` cluster and served from the `ignition` directory. Keep the installer `auth` directory and pull secret private. Do not boot with an old cluster's Ignition files.

## Step 4 — Create the Kea DHCP configuration by hand


Run on the bastion:
dnf install -y kea
rpm -q kea
mkdir -p /root/ocp-config-backup
cp -a /etc/kea/kea-dhcp4.conf /root/ocp-config-backup/kea-dhcp4.conf.before-pxe
vi /etc/kea/kea-dhcp4.conf

In vi, use the complete seven-node Kea configuration in Step 4 of your RHEL 10 PXE guide. Replace all seven MAC placeholders with the real Proxmox MACs. It binds DHCP to ens19, advertises pxelinux.0, and gives nodes 192.168.0.30 as their only DNS server.

Then validate and start it:
jq -e . /etc/kea/kea-dhcp4.conf
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
firewall-cmd --get-zone-of-interface=ens19

If the last command prints internal, continue:
firewall-cmd --permanent --zone=internal --add-service=dhcp
firewall-cmd --reload
systemctl enable --now kea-dhcp4
systemctl status kea-dhcp4 --no-pager -l
ss -lunp | grep ':67'

Expect active (running) and a UDP port 67 listener. If Kea’s configuration test reports an error, fix it before starting the service. docs.redhat.com



Check that `ens19` is on the isolated cluster bridge and there is no other DHCP server answering there:

```bash
systemctl is-active dnsmasq dhcpd kea-dhcp4
ss -lunp | grep ':67'
```

Back up any existing Kea file, then edit the RHEL 10 configuration:

```bash
cp -a /etc/kea/kea-dhcp4.conf /root/ocp-config-backup/kea-dhcp4.conf.before-pxe
vi /etc/kea/kea-dhcp4.conf
```

Replace its contents with the JSON below. **Before saving, replace all seven `REPLACE_...` strings with the corresponding real MAC addresses** in colon format, such as `bc:24:11:aa:bb:cc`. Do not use the screenshot's example MACs unless they are actually assigned to these VMs.

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": ["ens19"]
    },
    "valid-lifetime": 43200,
    "authoritative": true,
    "match-client-id": false,
    "subnet4": [
      {
        "id": 1,
        "subnet": "172.16.50.0/24",
        "interface": "ens19",
        "next-server": "172.16.50.2",
        "boot-file-name": "pxelinux.0",
        "option-data": [
          { "name": "routers", "data": "172.16.50.2" },
          { "name": "domain-name-servers", "data": "192.168.0.30" },
          { "name": "domain-name", "data": "ocp-test.kubinfra.com" },
          { "name": "ntp-servers", "data": "192.168.0.30" }
        ],
        "reservations": [
          { "hw-address": "REPLACE_BOOTSTRAP_MAC", "ip-address": "172.16.50.62", "hostname": "bootstrap.ocp-test.kubinfra.com" },
          { "hw-address": "REPLACE_MASTER01_MAC",  "ip-address": "172.16.50.50", "hostname": "master01.ocp-test.kubinfra.com" },
          { "hw-address": "REPLACE_MASTER02_MAC",  "ip-address": "172.16.50.51", "hostname": "master02.ocp-test.kubinfra.com" },
          { "hw-address": "REPLACE_MASTER03_MAC",  "ip-address": "172.16.50.52", "hostname": "master03.ocp-test.kubinfra.com" },
          { "hw-address": "REPLACE_WORKER01_MAC",  "ip-address": "172.16.50.53", "hostname": "worker01.ocp-test.kubinfra.com" },
          { "hw-address": "REPLACE_WORKER02_MAC",  "ip-address": "172.16.50.54", "hostname": "worker02.ocp-test.kubinfra.com" },
          { "hw-address": "REPLACE_WORKER03_MAC",  "ip-address": "172.16.50.55", "hostname": "worker03.ocp-test.kubinfra.com" }
        ]
      }
    ]
  }
}
```

Kea offers addresses only to these seven MACs because no dynamic pool is defined. Option 6 points nodes **only to IPA 192.168.0.30 for DNS**. The `ntp-servers` option assumes you already configured IPA's chronyd to *serve* time in the main guide. `next-server` sends clients to the bastion for TFTP. Confirm the JSON and Kea syntax now; **do not start Kea yet**:

```bash
jq -e . /etc/kea/kea-dhcp4.conf >/dev/null
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

## Step 5 — Create three role PXE files

Edit `/var/lib/tftpboot/pxelinux.cfg/bootstrap`:

```bash
vi /var/lib/tftpboot/pxelinux.cfg/bootstrap
```

```text
DEFAULT bootstrap
PROMPT 0
TIMEOUT 30
LABEL bootstrap
  KERNEL http://172.16.50.2:8080/ocp/rhcos/live-kernel
  APPEND initrd=http://172.16.50.2:8080/ocp/rhcos/live-initramfs.img ip=dhcp rd.neednet=1 coreos.live.rootfs_url=http://172.16.50.2:8080/ocp/rhcos/live-rootfs.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.2:8080/ocp/ignition/bootstrap.ign
```

Edit `/var/lib/tftpboot/pxelinux.cfg/master`:

```bash
vi /var/lib/tftpboot/pxelinux.cfg/master
```

```text
DEFAULT master
PROMPT 0
TIMEOUT 30
LABEL master
  KERNEL http://172.16.50.2:8080/ocp/rhcos/live-kernel
  APPEND initrd=http://172.16.50.2:8080/ocp/rhcos/live-initramfs.img ip=dhcp rd.neednet=1 coreos.live.rootfs_url=http://172.16.50.2:8080/ocp/rhcos/live-rootfs.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.2:8080/ocp/ignition/master.ign
```

Edit `/var/lib/tftpboot/pxelinux.cfg/worker`:

```bash
vi /var/lib/tftpboot/pxelinux.cfg/worker
```

```text
DEFAULT worker
PROMPT 0
TIMEOUT 30
LABEL worker
  KERNEL http://172.16.50.2:8080/ocp/rhcos/live-kernel
  APPEND initrd=http://172.16.50.2:8080/ocp/rhcos/live-initramfs.img ip=dhcp rd.neednet=1 coreos.live.rootfs_url=http://172.16.50.2:8080/ocp/rhcos/live-rootfs.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://172.16.50.2:8080/ocp/ignition/worker.ign
```

**Keep each `APPEND` entry on one physical line** when editing. `TIMEOUT 30` selects the role automatically after about three seconds. There is no menu to select manually. All nodes in the same role use the same `.ign` file.

Create a safe default for any NIC without a matching MAC file:

```bash
vi /var/lib/tftpboot/pxelinux.cfg/default
```

```text
DEFAULT local
PROMPT 0
TIMEOUT 10
LABEL local
  LOCALBOOT 0
```

The default boots the local disk; it does not install an unknown machine.

## Step 6 — Link each real VM MAC to its role

PXELINUX requests a filename with prefix `01-`, **lowercase hexadecimal**, and **hyphens**, not colons. For example, a real MAC `bc:24:11:aa:bb:cc` becomes filename `01-bc-24-11-aa-bb-cc`. Edit and run the following commands **one by one**, replacing the uppercase placeholder in each filename with the corresponding VM's real lowercase, hyphenated MAC. Do not type `<` or `>` around the MAC.

```bash
ln -s bootstrap /var/lib/tftpboot/pxelinux.cfg/01-REPLACE_BOOTSTRAP_MAC
ln -s master /var/lib/tftpboot/pxelinux.cfg/01-REPLACE_MASTER01_MAC
ln -s master /var/lib/tftpboot/pxelinux.cfg/01-REPLACE_MASTER02_MAC
ln -s master /var/lib/tftpboot/pxelinux.cfg/01-REPLACE_MASTER03_MAC
ln -s worker /var/lib/tftpboot/pxelinux.cfg/01-REPLACE_WORKER01_MAC
ln -s worker /var/lib/tftpboot/pxelinux.cfg/01-REPLACE_WORKER02_MAC
ln -s worker /var/lib/tftpboot/pxelinux.cfg/01-REPLACE_WORKER03_MAC


cd /var/lib/tftpboot/pxelinux.cfg || exit

ln -sfn bootstrap 01-bc-24-11-04-89-92

ln -sfn master 01-bc-24-11-58-52-8a
ln -sfn master 01-bc-24-11-73-0a-0b
ln -sfn master 01-bc-24-11-94-ab-66

ln -sfn worker 01-bc-24-11-12-6c-62
ln -sfn worker 01-bc-24-11-65-ea-88
ln -sfn worker 01-bc-24-11-bb-f6-ca



```

**Do not run this block unchanged.** The placeholder names intentionally cannot match a real PXE client's MAC. If old links or UUID/IP-specific PXE files are already in this directory, archive them first; PXELINUX might choose them ahead of these links. Verify exactly seven correct links and the four plain files:

```bash
ls -l /var/lib/tftpboot/pxelinux.cfg/
restorecon -RFv /var/lib/tftpboot
```

An actual master01 entry should look like `01-bc-24-11-aa-bb-cc -> master` **only if that is master01's real MAC**. Compare every link against the Proxmox inventory and the matching Kea reservation. If `ln -s` reports `File exists`, inspect the existing link with `ls -l` before changing it.

## Step 7 — Open the private firewall and start Kea

In the main guide, `ens19` is assigned to the firewalld **internal** zone. Check that first:

```bash
firewall-cmd --get-zone-of-interface=ens19
```

If it prints `internal`, allow the three PXE services on **that** zone:

```bash
firewall-cmd --permanent --zone=internal --add-service=dhcp
firewall-cmd --permanent --zone=internal --add-service=tftp
firewall-cmd --permanent --zone=internal --add-port=8080/tcp
firewall-cmd --reload
systemctl enable --now tftp.socket
systemctl enable --now kea-dhcp4
systemctl status kea-dhcp4 tftp.socket httpd --no-pager -l
ss -lntup | grep -E ':(67|69|8080)\b'
```

If `ens19` is assigned to another zone, use its **actual** zone instead of `internal` in the three firewall commands. Keep the forwarding, masquerading, HAProxy, and other firewall settings from your main guide. Kea is DHCP only; it does not resolve DNS names. IPA at `192.168.0.30` answers the nodes' DNS requests.

## Step 8 — Check IPA DNS and perform one-time PXE boots

Verify IPA's answers from the bastion:

```bash
dig @192.168.0.30 bootstrap.ocp-test.kubinfra.com +short
dig @192.168.0.30 master01.ocp-test.kubinfra.com +short
dig @192.168.0.30 api-int.ocp-test.kubinfra.com +short
dig @192.168.0.30 -x 172.16.50.50 +short
```

Expect `.62`, `.50`, `.49`, and `master01.ocp-test.kubinfra.com.` respectively. In Proxmox, set **disk first** as the normal boot order, then use a **one-time network boot** for each installation:

1. PXE boot **bootstrap** first; confirm it receives `172.16.50.62`, loads `bootstrap`, and fetches `bootstrap.ign`.
2. PXE boot **master01**, **master02**, and **master03**; each MAC link must select `master` and fetch `master.ign`.
3. PXE boot **worker01**, **worker02**, and **worker03**; each must select `worker` and fetch `worker.ign`.

Watch the DHCP and Apache logs on the bastion in separate terminals:

```bash
journalctl -fu kea-dhcp4
```

```bash
tail -f /var/log/httpd/access_log
```

On a booted node, verify that DHCP gave it **192.168.0.30 as the DNS server**, the correct reserved IP, and `172.16.50.2` as its default gateway. Test `getent hosts api-int.ocp-test.kubinfra.com`; the answer must be `172.16.50.49`. Continue the main runbook's HAProxy, bootstrap-complete, and CSR approval steps.

After a node has installed successfully and booted from disk, delete **only that node's** `01-<MAC>` symlink. Leave its Kea DHCP reservation in place so it keeps its IP address. Once all seven are installed, the `default` local-disk entry prevents an accidental unknown PXE client from starting an installation. A future deliberate reinstall requires restoring only that node's verified MAC link and a fresh appropriate Ignition file.

## Quick troubleshooting

| What you see | Check first |
|---|---|
| `curl` cannot connect to `172.16.50.2:8080` | `ss -lntp`, Apache `Listen`, `systemctl restart httpd`, and the `/ocp/` URL prefix |
| Kea fails to start | `kea-dhcp4 -t /etc/kea/kea-dhcp4.conf` and `journalctl -u kea-dhcp4 -n 40 --no-pager` |
| No DHCP offer | Actual Proxmox NIC MAC, isolated bridge, Kea service, another DHCP server, firewall on `ens19` |
| TFTP fails | `tftp.socket`, `/var/lib/tftpboot/pxelinux.0`, `/var/lib/tftpboot/ldlinux.c32`, UDP 69 |
| Wrong role | Exact lowercase `01-<MAC>` filename, link target, old PXE files |
| RHCOS or `.ign` download fails | All six HTTP `200` tests in Step 3, file permissions, Apache log |
| Node cannot resolve DNS | DHCP option `domain-name-servers` contains only `192.168.0.30`; check IPA and bastion NAT |
| Node starts installer again | Restore disk-first boot order and remove that node's MAC symlink |

## References

- [OpenShift 4.22 bare-metal UPI: PXE entries and matching RHCOS downloads](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_bare_metal/user-provisioned-infrastructure)
- [RHEL 10: Kea DHCP service and reservations](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_networking_infrastructure_services/providing-dhcp-services)
- [RHEL 10: BIOS PXE, syslinux-tftpboot, and TFTP](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interactively_installing_rhel_over_the_network/preparing-a-pxe-installation-source)

The MACs and actual disk devices are only known on your Proxmox host. This configuration must pass the checks above on your bastion before it can be called operational.
