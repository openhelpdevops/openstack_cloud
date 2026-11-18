# MAC-specific PXE installation for `ocp-test.kubinfra.com`

Use this step **after** the network, IPA DNS/NTP, HAProxy, HTTP server, OpenShift installer, and Ignition files from the main runbook are ready. It replaces the earlier PXE/DHCP configuration and node network-boot portions. It follows the screenshot's `pxelinux.cfg/{bootstrap,master,worker}` and `01-<MAC>` symlink layout. **Run commands on the bastion as root**, except the Proxmox and node-console checks. The RHEL 10 path uses Kea; the RHEL 9 path uses `dhcpd`. Choose exactly one.

This procedure is for **x86_64 VMs using SeaBIOS/legacy PXE** on an isolated Proxmox bridge. OVMF/UEFI does not run `pxelinux.0`; it requires a separate UEFI bootloader configuration. The bastion's `ens19` is `172.16.50.2/24` on that bridge. Its `ens18` is `192.168.0.40/24`. IPA `192.168.0.30` remains the **only DNS server** in both DHCP variants; the bastion does not run DNS. The nodes route through `172.16.50.2`, with forwarding/NAT already set up in the main runbook. All HTTP URLs below use the existing Apache document root `/var/www/html` and its private listener `172.16.50.2:8080`.

**This is an unattended disk installation.** Booting a node through its MAC entry overwrites its target disk. Verify every actual VM MAC, disk type, and install device before enabling that node's link. The screenshot's example MACs and old domain/IP values are not this lab's inventory.

## 1. Prove the prerequisites before creating a PXE entry

On Proxmox, set **BIOS: SeaBIOS** and attach each of the seven VM NICs to the isolated cluster bridge shared with bastion `ens19`. Configure the **disk first** in persistent boot order; use the VM's one-time **Network/PXE** boot for installation. Record each NIC's MAC from **VM → Hardware → Network Device** or `qm config <VMID>` on the Proxmox host. Check each disk from its VM configuration or a live ISO with `lsblk -o NAME,SIZE,MODEL,TYPE`. A VirtIO SCSI disk commonly appears as `/dev/sda`; VirtIO Block commonly appears as `/dev/vda`. The role files below require all masters to have the *same device path* and all workers to have the *same device path*.

On the bastion:

```bash
ip -br -4 address show ens19
systemctl is-active dnsmasq dhcpd kea-dhcp4 || true
ss -lunp | grep -E ':(67|69)\b' || true
ss -lntp | grep ':8080' || true
httpd -t
curl -fsS http://172.16.50.2:8080/ocp/health.txt
dig @192.168.0.30 api-int.ocp-test.kubinfra.com +short
dig @192.168.0.30 -x 172.16.50.50 +short
```

Expect `ens19` to have `172.16.50.2/24`, `Syntax OK`, health text `ocp asset server ready`, API-int `172.16.50.49`, and master01's PTR. **Stop if HTTP port 8080 does not accept a connection.** `systemctl enable --now httpd` will not reload an already running Apache configuration: use `systemctl restart httpd`, then inspect `systemctl status httpd --no-pager -l` and `journalctl -u httpd -n 40 --no-pager`. The URL for `/var/www/html/ocp/health.txt` is `/ocp/health.txt`, not `/health.txt`.

There must be **one** DHCP server on this bridge. If an older DHCP process serves `ens19`, retire that configuration before enabling the selected service below. Do not connect `ens19` to the home LAN. In the main runbook, complete masquerading/forwarding, HAProxy, and the private firewalld rules as well.

## 2. Install the PXE packages and loader

The RHEL host's BaseOS and AppStream repositories must already be enabled. On **RHEL 10**, install Kea; on **RHEL 9**, install the Red Hat `dhcp-server` package. Install the common packages and discover the real RPM-owned paths for the loader (this avoids the previously seen `/tftpboot/pxelinux.0: No such file` error):

```bash
set -e
. /etc/os-release
case "$VERSION_ID" in
  10.*) dnf install -y kea tftp-server syslinux-tftpboot curl jq bind-utils httpd policycoreutils-python-utils python3 ;;
  9.*)  dnf install -y dhcp-server tftp-server syslinux-tftpboot curl jq bind-utils httpd policycoreutils-python-utils python3 ;;
  *) echo "This step covers RHEL 9 or RHEL 10 only: $VERSION_ID" >&2; exit 1 ;;
esac

install -d -m 700 /root/ocp-config-backup /root/ocp-inputs
install -d -m 755 /var/lib/tftpboot /var/www/html/ocp/rhcos /var/www/html/ocp/ignition
rpm -q syslinux-tftpboot tftp-server
rpm -ql syslinux-tftpboot | grep -E '/(pxelinux\.0|ldlinux\.c32)$'

pxe_src=$(rpm -ql syslinux-tftpboot | grep -m1 '/pxelinux\.0$')
ld_src=$(rpm -ql syslinux-tftpboot | grep -m1 '/ldlinux\.c32$')
test -f "$pxe_src" && test -f "$ld_src"
install -m 0644 "$pxe_src" /var/lib/tftpboot/pxelinux.0
install -m 0644 "$ld_src" /var/lib/tftpboot/ldlinux.c32
restorecon -RFv /var/lib/tftpboot
systemctl enable --now tftp.socket
ls -lZ /var/lib/tftpboot/pxelinux.0 /var/lib/tftpboot/ldlinux.c32
```

If `dnf install syslinux-tftpboot` fails, check `dnf repolist` and `dnf provides '*/pxelinux.0'` before continuing. If `rpm -ql` is empty, the package is not installed; **do not** try copying a guessed `/tftpboot` path. The files above live at the TFTP root because DHCP below advertises `pxelinux.0` (without a subdirectory).

The `tftp.socket` listener on UDP 69 is normal. It does not mean DHCP is running.

## 3. Verify matching RHCOS PXE assets and role Ignition

The three RHCOS files must come from the **same installer** as the Ignition files. If the main runbook already downloaded and SHA-256 checked them, keep those files. Otherwise, with that exact `openshift-install` on the bastion:

```bash
set -euo pipefail
openshift-install version
openshift-install coreos print-stream-json > /root/ocp-inputs/rhcos-stream.json
for item in kernel initramfs rootfs; do
  url=$(jq -er --arg x "$item" '.architectures.x86_64.artifacts.metal.formats.pxe[$x].location' /root/ocp-inputs/rhcos-stream.json)
  sha=$(jq -er --arg x "$item" '.architectures.x86_64.artifacts.metal.formats.pxe[$x].sha256' /root/ocp-inputs/rhcos-stream.json)
  case "$item" in
    kernel)    dest=/var/www/html/ocp/rhcos/live-kernel ;;
    initramfs) dest=/var/www/html/ocp/rhcos/live-initramfs.img ;;
    rootfs)    dest=/var/www/html/ocp/rhcos/live-rootfs.img ;;
  esac
  curl --fail --location --retry 3 "$url" -o "$dest"
  printf '%s  %s\n' "$sha" "$dest" | sha256sum --check
done
chmod 0644 /var/www/html/ocp/rhcos/*
restorecon -RFv /var/www/html/ocp/rhcos
```

The main runbook publishes `/root/ocp-install/{bootstrap,master,worker}.ign` as `/var/www/html/ocp/ignition/{bootstrap,master,worker}.ign`. Verify **those are the current cluster's files**, and never expose `/root/ocp-install/auth`, the pull secret, or the kubeadmin password. Run the following gate:

```bash
for item in rhcos/live-kernel rhcos/live-initramfs.img rhcos/live-rootfs.img \
            ignition/bootstrap.ign ignition/master.ign ignition/worker.ign; do
  curl --fail --silent --show-error --head "http://172.16.50.2:8080/ocp/$item" || exit 1
done
```

Every request must return `200 OK`. A `403` points to Apache access rules/SELinux; `404` means the path or file is wrong. Do not begin PXE boot with expired Ignition files or a mismatched RHCOS stream.

## 4. Inventory the seven current MACs and target disks

Create the same inventory consumed by the main guide, editing **every** `REPLACE_...` before continuing. Do not reuse MACs from the screenshot:

```bash
cat > /root/ocp-inputs/nodes.tsv <<'EOF'
# name role ip mac target_disk
bootstrap bootstrap 172.16.50.62 REPLACE_BOOTSTRAP_MAC /dev/sda
master01 master 172.16.50.50 REPLACE_MASTER01_MAC /dev/sda
master02 master 172.16.50.51 REPLACE_MASTER02_MAC /dev/sda
master03 master 172.16.50.52 REPLACE_MASTER03_MAC /dev/sda
worker01 worker 172.16.50.53 REPLACE_WORKER01_MAC /dev/sda
worker02 worker 172.16.50.54 REPLACE_WORKER02_MAC /dev/sda
worker03 worker 172.16.50.55 REPLACE_WORKER03_MAC /dev/sda
EOF
vi /root/ocp-inputs/nodes.tsv
```

For example, only if the *actual* master01 VM MAC is `BC:24:11:AA:BB:CC`, its symlink will be `01-bc-24-11-aa-bb-cc -> master`. Change `/dev/sda` to the verified OS device when needed. If master or worker machines use different target disk paths, use separate **per-node** PXE configs; do not run this shared-role generator until their paths are uniform.

## 5. Generate role files, seven links, and both DHCP variants

This generator works in a staging directory. It validates the exact name, role, IP, MAC uniqueness, and disk path before producing any live configuration. It creates the screenshot's three role files, a safe `default` that boots the local disk, and seven MAC links. The generated Kea and dhcpd configs have **no dynamic range** and deliver `192.168.0.30` as the sole DNS and NTP address.

```bash
cat > /root/ocp-inputs/build-ocp-pxe.py <<'PY'
#!/usr/bin/env python3
import collections
import ipaddress
import json
import re
from pathlib import Path

inventory = Path('/root/ocp-inputs/nodes.tsv')
stage = Path('/root/ocp-pxe-stage')
expected = {
    'bootstrap': ('bootstrap', '172.16.50.62'),
    'master01': ('master', '172.16.50.50'),
    'master02': ('master', '172.16.50.51'),
    'master03': ('master', '172.16.50.52'),
    'worker01': ('worker', '172.16.50.53'),
    'worker02': ('worker', '172.16.50.54'),
    'worker03': ('worker', '172.16.50.55'),
}
rows = []
for raw in inventory.read_text().splitlines():
    raw = raw.strip()
    if not raw or raw.startswith('#'):
        continue
    parts = raw.split()
    if len(parts) != 5:
        raise SystemExit(f'Expected 5 columns: {raw}')
    name, role, ip, mac, disk = parts
    mac = mac.lower()
    if not re.fullmatch(r'(?:[0-9a-f]{2}:){5}[0-9a-f]{2}', mac):
        raise SystemExit(f'Invalid/unreplaced MAC for {name}: {mac}')
    if (name not in expected or (role, ip) != expected[name]
            or ipaddress.ip_address(ip) not in ipaddress.ip_network('172.16.50.0/24')):
        raise SystemExit(f'Unexpected name/role/IP: {raw}')
    if not re.fullmatch(r'/dev/(?:[a-zA-Z0-9_/-]+)', disk) or '..' in disk:
        raise SystemExit(f'Invalid disk: {raw}')
    rows.append((name, role, ip, mac, disk))

if len(rows) != 7 or {r[0] for r in rows} != set(expected):
    raise SystemExit('Inventory must list exactly the seven expected nodes')
for i in (0, 2, 3):
    if len({r[i] for r in rows}) != 7:
        raise SystemExit('Duplicate hostname, IP or MAC')
disks = {}
for role in ('bootstrap', 'master', 'worker'):
    unique = {r[4] for r in rows if r[1] == role}
    if len(unique) != 1:
        raise SystemExit(f'{role} target disks differ; use per-node PXE entries')
    disks[role] = unique.pop()
if stage.exists() and any(stage.iterdir()):
    raise SystemExit(f'{stage} is not empty; archive it before regeneration')

cfg = stage / 'pxelinux.cfg'
cfg.mkdir(parents=True, exist_ok=True)
base = 'http://172.16.50.2:8080/ocp'
for role in ('bootstrap', 'master', 'worker'):
    (cfg / role).write_text(
        f'DEFAULT {role}\nPROMPT 0\nTIMEOUT 30\n'
        f'LABEL {role}\n'
        f'  KERNEL {base}/rhcos/live-kernel\n'
        f'  APPEND initrd={base}/rhcos/live-initramfs.img '
        f'ip=dhcp rd.neednet=1 '
        f'coreos.live.rootfs_url={base}/rhcos/live-rootfs.img '
        f'coreos.inst.install_dev={disks[role]} '
        f'coreos.inst.ignition_url={base}/ignition/{role}.ign\n'
    )
(cfg / 'default').write_text(
    'DEFAULT local\nPROMPT 0\nTIMEOUT 10\nLABEL local\n  LOCALBOOT 0\n'
)
for name, role, ip, mac, disk in rows:
    (cfg / ('01-' + mac.replace(':', '-'))).symlink_to(role)

kea = {'Dhcp4': {
    'interfaces-config': {'interfaces': ['ens19'], 'service-sockets-require-all': True},
    'authoritative': True, 'valid-lifetime': 43200, 'match-client-id': False,
    'subnet4': [{
        'id': 1, 'subnet': '172.16.50.0/24', 'interface': 'ens19',
        'next-server': '172.16.50.2', 'boot-file-name': 'pxelinux.0',
        'option-data': [
            {'name': 'routers', 'data': '172.16.50.2'},
            {'name': 'domain-name-servers', 'data': '192.168.0.30'},
            {'name': 'domain-name', 'data': 'ocp-test.kubinfra.com'},
            {'name': 'domain-search', 'data': 'ocp-test.kubinfra.com, kubinfra.com'},
            {'name': 'ntp-servers', 'data': '192.168.0.30'},
        ],
        'reservations': [
            {'hw-address': mac, 'ip-address': ip,
             'hostname': name + '.ocp-test.kubinfra.com'}
            for name, role, ip, mac, disk in rows
        ],
    }],
}}
(stage / 'kea-dhcp4.conf').write_text(json.dumps(kea, indent=2) + '\n')

dhcpd = [
    'authoritative;', 'default-lease-time 43200;', 'max-lease-time 43200;',
    'option domain-name "ocp-test.kubinfra.com";',
    'option domain-name-servers 192.168.0.30;',
    'option ntp-servers 192.168.0.30;',
    'option domain-search "ocp-test.kubinfra.com", "kubinfra.com";',
    'next-server 172.16.50.2;', 'filename "pxelinux.0";',
    'subnet 172.16.50.0 netmask 255.255.255.0 {',
    '  option routers 172.16.50.2;',
    '  option subnet-mask 255.255.255.0;',
    '  option broadcast-address 172.16.50.255;',
    '}',
]
for name, role, ip, mac, disk in rows:
    dhcpd.extend([
        f'host {name} {{', f'  hardware ethernet {mac};',
        f'  fixed-address {ip};',
        f'  option host-name "{name}.ocp-test.kubinfra.com";', '}',
    ])
(stage / 'dhcpd.conf').write_text('\n'.join(dhcpd) + '\n')
print('Generated seven MAC links, three role files, Kea config and dhcpd config')
for name, role, ip, mac, disk in rows:
    print(f'{name:10s} {ip:15s} 01-{mac.replace(":", "-")} -> {role} ({disk})')
PY

python3 /root/ocp-inputs/build-ocp-pxe.py
ls -l /root/ocp-pxe-stage/pxelinux.cfg/
grep -H 'coreos.inst.ignition_url=' /root/ocp-pxe-stage/pxelinux.cfg/{bootstrap,master,worker}
```

Compare the printed mapping to **each VM in Proxmox** and compare every `coreos.inst.install_dev` with its disk. Stop if any row is wrong. The role files are a **single APPEND line each**; do not wrap that line manually. There is no `UI menu.c32` dependency.

## 6. Validate and activate exactly one DHCP service

Choose the branch that matches `cat /etc/redhat-release`. Keep any previous service configuration in `/root/ocp-config-backup` and do not overwrite an unrelated DHCP server on another network.

**RHEL 10 / Kea:**

```bash
kea-dhcp4 -t /root/ocp-pxe-stage/kea-dhcp4.conf
test -f /etc/kea/kea-dhcp4.conf && cp -a /etc/kea/kea-dhcp4.conf /root/ocp-config-backup/kea-dhcp4.conf.before-ocp-pxe
install -m 0644 /root/ocp-pxe-stage/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf
restorecon -v /etc/kea/kea-dhcp4.conf
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

**RHEL 9 / ISC dhcpd:**

```bash
dhcpd -t -cf /root/ocp-pxe-stage/dhcpd.conf
test -f /etc/dhcp/dhcpd.conf && cp -a /etc/dhcp/dhcpd.conf /root/ocp-config-backup/dhcpd.conf.before-ocp-pxe
install -d -m 0755 /etc/dhcp
install -m 0644 /root/ocp-pxe-stage/dhcpd.conf /etc/dhcp/dhcpd.conf
restorecon -v /etc/dhcp/dhcpd.conf
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

The RHEL 9 config defines only the private `172.16.50.0/24` subnet, so `dhcpd` selects the `ens19` interface on this two-subnet bastion. If another interface also has a `172.16.50.0/24` address, explicitly constrain dhcpd to `ens19` using the RHEL 9 interface binding instructions in the References before starting it. RHEL 10 Kea explicitly binds to `ens19`.

**Install the PXELINUX configs only after validating the generated DHCP config and the MAC/disk table.** The following archives an existing `pxelinux.cfg` directory. Do not run it on a TFTP root shared with other PXE deployments.

```bash
set -e
stamp=$(date +%Y%m%d%H%M%S)
cp -a /root/ocp-pxe-stage/pxelinux.cfg /var/lib/tftpboot/.ocp-pxelinux.cfg.new
if [ -e /var/lib/tftpboot/pxelinux.cfg ]; then
  mv /var/lib/tftpboot/pxelinux.cfg "/root/ocp-config-backup/pxelinux.cfg.$stamp"
fi
mv /var/lib/tftpboot/.ocp-pxelinux.cfg.new /var/lib/tftpboot/pxelinux.cfg
chmod 0644 /var/lib/tftpboot/pxelinux.cfg/{default,bootstrap,master,worker}
restorecon -RFv /var/lib/tftpboot
ls -l /var/lib/tftpboot/pxelinux.cfg/
```

Open the selected private-interface firewall zone for DHCP, TFTP, and HTTP 8080. These rules do not expose the PXE server on `ens18`:

```bash
pxe_zone=$(firewall-cmd --get-zone-of-interface=ens19)
test -n "$pxe_zone" && test "$pxe_zone" != no
firewall-cmd --permanent --zone="$pxe_zone" --add-service=dhcp
firewall-cmd --permanent --zone="$pxe_zone" --add-service=tftp
firewall-cmd --permanent --zone="$pxe_zone" --add-port=8080/tcp
firewall-cmd --reload
firewall-cmd --zone="$pxe_zone" --list-all
```

The main runbook also needs its forwarding/NAT and HAProxy firewalld settings. Firewall input rules here do not replace those rules.

Now enable **only the matching service**. RHEL 10:

```bash
systemctl enable kea-dhcp4
systemctl restart kea-dhcp4
systemctl status kea-dhcp4 --no-pager -l
```

RHEL 9:

```bash
systemctl enable dhcpd
systemctl restart dhcpd
systemctl status dhcpd --no-pager -l
```

## 7. Test the exact boot chain before any node installs

```bash
systemctl is-active httpd tftp.socket
systemctl is-active kea-dhcp4 || systemctl is-active dhcpd
ss -lntup | grep -E ':(67|69|8080)\b'
ls -l /var/lib/tftpboot/{pxelinux.0,ldlinux.c32}
find /var/lib/tftpboot/pxelinux.cfg -maxdepth 1 -type l -printf '%f -> %l\n' | sort
bootstrap_mac=$(awk '$1 == "bootstrap" {print tolower($4)}' /root/ocp-inputs/nodes.tsv | tr ':' '-')
readlink -f "/var/lib/tftpboot/pxelinux.cfg/01-$bootstrap_mac"
curl -fsS http://172.16.50.2:8080/ocp/health.txt
dig @192.168.0.30 bootstrap.ocp-test.kubinfra.com +short
dig @192.168.0.30 api-int.ocp-test.kubinfra.com +short
```

Replace the `readlink` placeholder with one actual generated filename. It must resolve to the corresponding role. If your installed curl supports TFTP (`curl -V` lists `tftp`), check a role file from the TFTP listener itself, substituting one actual MAC:

```bash
curl -fsS 'tftp://172.16.50.2/pxelinux.cfg/01-bc-24-11-aa-bb-cc'
```

The example address in this *test command* must be replaced with an actual generated link. The response should show `DEFAULT master` for a master, `DEFAULT worker` for a worker, or `DEFAULT bootstrap` for bootstrap. If TFTP fails, use `journalctl -u tftp.socket -b --no-pager` and check SELinux/firewalld. Test the exact client path with a private-side VM before booting the seven nodes, if one is available: verify DHCP offers `172.16.50.2` as server/next-server, option 6 contains only `192.168.0.30`, and `dig @192.168.0.30 ...` works through the bastion's NAT.

## 8. One-time PXE boot and monitor

**First:** use a one-time Proxmox network boot for **bootstrap**. It should receive `.62`, download `pxelinux.0`, select `01-<its MAC> -> bootstrap`, fetch the HTTP kernel, initramfs, rootfs and `bootstrap.ign`, install RHCOS, then restart from disk. Watch the console; do not assume the install succeeded solely because the VM rebooted. Its persistent boot order stays disk first.

**Next:** one-time PXE boot **master01–03**, each with its `-> master` link and the shared `master.ign`. **Finally:** boot **worker01–03**, each with its `-> worker` link and `worker.ign`. Workers can wait for the control plane, but their certificates still need the main runbook's CSR approval step. Keep the original load-balancer/backend and `wait-for bootstrap-complete` steps.

While booting, watch on the bastion in separate terminals:

```bash
tcpdump -ni ens19 -e -vvv 'udp port 67 or udp port 68 or udp port 69'
```

```bash
journalctl -fu kea-dhcp4   # RHEL 10; use journalctl -fu dhcpd on RHEL 9
```

```bash
tail -f /var/log/httpd/access_log
```

Expect **one DHCP offer**, the right reserved IP and MAC, requests for `pxelinux.0` and the role file, and HTTP `200` requests for RHCOS and the matching `.ign`. On an installed node, `cat /etc/resolv.conf` should show IPA `192.168.0.30` as resolver; verify `getent hosts api-int.ocp-test.kubinfra.com`, `chronyc sources`, `ip route`, and its FQDN. The node must be able to reach the API/MCS and registry through the already configured gateway and HAProxy.

**Prevent reinstall after each successful disk boot:** remove *that node's* `01-<MAC>` link while retaining its DHCP reservation. Do not remove all links until each corresponding node is successfully installed. You can locate a node's exact link from the generated table or:

```bash
grep '^master01 ' /root/ocp-inputs/nodes.tsv
# Example ONLY: rm /var/lib/tftpboot/pxelinux.cfg/01-bc-24-11-aa-bb-cc
```

After all seven nodes are installed, the `default` PXE entry boots a local disk if a network boot is accidentally selected. **Keep the selected DHCP service running** so nodes retain their IPs. Preserve the HTTP rootfs/kernel/initramfs and worker Ignition for future workers; protect the installation directory and `auth/` privately. To reinstall a node deliberately, obtain fresh compatible assets if required, verify the disk/MAC, restore only that node's role link, use a one-time PXE boot, then remove the link again.

## What to check if boot stops

| Symptom | First check |
|---|---|
| No DHCP offer | Bridge and NIC MAC, selected DHCP service, only one DHCP server, `ens19` firewalld zone, Proxmox NIC firewall |
| TFTP timeout | `tftp.socket`, UDP 69 rule, `pxelinux.0` under `/var/lib/tftpboot`, TFTP/SELinux logs |
| Wrong menu/role | Exact `01-<lowercase MAC with hyphens>` symlink, old UUID/IP PXE configs, actual VM MAC |
| Cannot start kernel/initramfs | Apache listener at `172.16.50.2:8080`, HTTP `200`, PXELINUX `ldlinux.c32` from same RPM |
| `rootfs` or Ignition 403/404 | `/ocp/` URL prefix, `Require ip` rule, SELinux labels, correct files and release |
| Installs to wrong disk or repeats | Stop; verify VM disk mapping and persistent disk-first boot order, then remove its install link |
| Nodes cannot resolve or fetch MCS | DHCP option 6 only IPA `.30`, NAT/routing, IPA A/PTR records, HAProxy `172.16.50.49:22623` |

## Primary references

- [Red Hat OpenShift 4.22: UPI bare metal RHCOS PXE parameters and matching RHCOS stream](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_bare_metal/user-provisioned-infrastructure)
- [RHEL 10: BIOS PXE/TFTP and Kea boot-file-name](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interactively_installing_rhel_over_the_network/preparing-a-pxe-installation-source)
- [RHEL 10: Kea DHCPv4 reservations](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_networking_infrastructure_services/providing-dhcp-services)
- [RHEL 9: dhcpd service and interface selection](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/providing-dhcp-services_networking-infrastructure-services)
- [Kea DHCPv4 server reference: host reservations and option data](https://kea.readthedocs.io/en/stable/arm/dhcp4-srv.html)

This procedure is configured for your lab and reviewed against primary documentation. It has **not** been executed on the bastion or the Proxmox VMs; verify each gate with your real MACs, disk paths, firewall zone, RHCOS version, and live service logs.
