# OpenShift lab — manual Ignition installation through web-console login

Checked on 7 October 2026 against the linked GitHub guide and Red Hat OpenShift 4.20 documentation.

This replaces the attached Steps 42–44, includes the Step 41 prerequisites, and continues through bootstrap completion, node certificate approval and web-console login. It starts from your current state: the helper already has generated files under `/home/cloudadmin/ocp-lab/cluster/manifests/` and `/home/cloudadmin/ocp-lab/cluster/openshift/`.

## Your environment and the files you need

| Setting | Value |
|---|---|
| Helper / Ignition HTTP server | `192.168.0.61` |
| Ignition HTTP URL | `http://192.168.0.61:8080/` |
| HTTP document directory | `/var/www/ocp` |
| Existing IPA DNS and NTP server | `192.168.0.10` |
| Cluster name / base domain | `ocp` / `openhelp.net` |
| Installation directory | `/home/cloudadmin/ocp-lab/cluster` |
| Public SSH key already created | `/home/cloudadmin/.ssh/ocp_ed25519.pub` |
| Matching private SSH key | `/home/cloudadmin/.ssh/ocp_ed25519` |
| Permanent cluster nodes | 3 masters + 3 workers |
| Additional UPI installation machine | 1 temporary bootstrap VM |

The helper keeps running HAProxy and the HTTP server. The temporary bootstrap VM runs RHCOS and receives `bootstrap.ign`.

### Do you need to enter node MAC addresses?

**No, for this manual ISO installation with persistent static IP addresses.** You do not enter a master or worker MAC address in `install-config.yaml`, the chrony MachineConfigs, or the three generated role Ignition files. Each VM needs its own network profile, IP address and matching IPA DNS records. Preserve its network profile with `--copy-network` when installing RHCOS.

| Network setup | Where a MAC mapping is needed |
|---|---|
| Your static-IP, manual ISO procedure | No MAC-to-IP reservation is needed |
| DHCP with fixed reservations | Map each node's NIC MAC to its assigned IP on the DHCP server |
| Network profile explicitly matched by MAC | Keep that profile's MAC match correct |

The linked GitHub training guide records MAC addresses because it creates DHCP reservations in `dhcpd.conf`. IPA providing DNS and NTP does not, by itself, provide those DHCP reservations. If a VM is still using DHCP, either configure its intended static address or arrange a reservation before continuing. Keeping a MAC inventory is optional for troubleshooting; every VM must still have a unique NIC MAC.

### DNS and load-balancer mapping for this topology

Keep your existing node IP addresses and load-balancer addresses. Use this mapping when checking IPA and HAProxy:

| Name or service | Destination |
|---|---|
| Node A/PTR records | That individual bootstrap, master or worker IP |
| `api.ocp.openhelp.net`, `api-int.ocp.openhelp.net` | Your API load-balancer address; TCP 6443 reaches the masters, and temporarily bootstrap |
| Machine Config Server, TCP 22623 | Your internal API load-balancer address; reaches the masters, and temporarily bootstrap |
| `*.apps.ocp.openhelp.net` | Your ingress load-balancer address; TCP 80/443 reaches workers running router pods |
| Ignition downloads | Helper `192.168.0.61`, TCP 8080 |
| Node/client DNS and node NTP | IPA `192.168.0.10`, DNS TCP/UDP 53 and NTP UDP 123 |

If HAProxy binds these services directly to the helper's `192.168.0.61` address, the API and application DNS records point to `.61`. If your earlier topology uses separate API/ingress virtual IPs, keep those existing addresses and verify HAProxy actually listens on them. The API and application records do not point to IPA just because IPA hosts DNS.

**Use the three role files created by `openshift-install`:**

| Machine | Hostname | Ignition file | Download URL |
|---|---|---|---|
| Bootstrap | `bootstrap.ocp.openhelp.net` | `bootstrap.ign` | `http://192.168.0.61:8080/bootstrap.ign` |
| Master 01 | `master01.ocp.openhelp.net` | `master.ign` | `http://192.168.0.61:8080/master.ign` |
| Master 02 | `master02.ocp.openhelp.net` | `master.ign` | `http://192.168.0.61:8080/master.ign` |
| Master 03 | `master03.ocp.openhelp.net` | `master.ign` | `http://192.168.0.61:8080/master.ign` |
| Worker 01 | `worker01.ocp.openhelp.net` | `worker.ign` | `http://192.168.0.61:8080/worker.ign` |
| Worker 02 | `worker02.ocp.openhelp.net` | `worker.ign` | `http://192.168.0.61:8080/worker.ign` |
| Worker 03 | `worker03.ocp.openhelp.net` | `worker.ign` | `http://192.168.0.61:8080/worker.ign` |

All three masters share `master.ign`; all three workers share `worker.ign`. Their existing static IP assignments and IPA DNS records distinguish the machines. The seven hand-written wrappers from the old Step 43 are replaced by this role-based procedure. Their hostname, NTP and data-disk functions are addressed explicitly below.

Commands in Steps 40–49 run **on the helper as root** unless a live/installed node is explicitly named. Step 50 identifies commands for your Windows browser computer. Run each command separately. These are manual instructions; there are no shell scripts, Python scripts, loops or `jq` commands.

## Step 40 — confirm the starting point

### 40.1 Check the installer version

```bash
openshift-install version
```

The MachineConfig YAML examples below use Ignition `3.5.0`, as recommended for OpenShift 4.20. Use an installer and RHCOS live ISO for your selected OpenShift release. If your installer reports another minor release, check that release's documentation before copying version-specific configuration. Preserve the schema versions inside files generated by the installer.

### 40.2 Inspect your existing manifest directories

```bash
ls -l /home/cloudadmin/ocp-lab/cluster/manifests/
ls -l /home/cloudadmin/ocp-lab/cluster/openshift/
```

Your existing `99_openshift-machineconfig_99-master-ssh.yaml` and `99_openshift-machineconfig_99-worker-ssh.yaml` are installer-generated SSH configurations. Keep them. The chrony files in Step 41 are additional configurations you create.

If `install-config.yaml` has disappeared after `create manifests`, that is normal: the installer consumes configuration inputs. Continue this installation attempt from its existing manifests and installer state. You do not need to recreate `install-config.yaml` or generate a second SSH key now.

Check whether Ignition files were already generated:

```bash
ls -l /home/cloudadmin/ocp-lab/cluster/bootstrap.ign /home/cloudadmin/ocp-lab/cluster/master.ign /home/cloudadmin/ocp-lab/cluster/worker.ign
```

At your current stage, `No such file or directory` for these three files simply means Step 42 has not run yet. If they already exist and you have since changed manifests, follow the regeneration note near the end of this document before using them.

### 40.3 Check master scheduling

```bash
nano /home/cloudadmin/ocp-lab/cluster/manifests/cluster-scheduler-02-config.yml
```

For your 3-master + 3-worker layout, the `spec` section should contain:

```yaml
spec:
  mastersSchedulable: false
```

Keep the other contents of that file. Save with **Ctrl+O → Enter**, then exit with **Ctrl+X**.

Your earlier `compute.replicas: 0` is correct for this UPI workflow: you create the worker VMs yourself. It does not prevent the three manually installed workers from joining.

### 40.4 Check IPA DNS and NTP

Use your existing node IP addresses. The attachment does not list those addresses, so this document does not assign new ones.

On the helper and on each RHCOS live console, identify the primary network profile for the `192.168.0.0/24` network:

```bash
nmcli -f NAME,DEVICE connection show
```

Replace `NODE_CONNECTION` with that profile's actual name. Set IPA as its persistent DNS server, keeping the existing IP address and gateway settings:

```bash
sudo nmcli connection modify "NODE_CONNECTION" ipv4.dns "192.168.0.10" ipv4.dns-search "ocp.openhelp.net" ipv4.ignore-auto-dns yes
sudo nmcli connection up "NODE_CONNECTION"
cat /etc/resolv.conf
```

Confirm the profile has your intended persistent static IP and gateway:

```bash
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns connection show "NODE_CONNECTION"
```

For the static-IP procedure, expect `ipv4.method: manual`, that VM's assigned address with `/24`, its existing gateway and DNS `192.168.0.10`. An `auto` profile uses DHCP. If you still need to set a static address, replace `NODE_IP` and `LAN_GATEWAY` with the address assigned to this VM and your actual gateway, then run from its console:

```bash
sudo nmcli connection modify "NODE_CONNECTION" ipv4.method manual ipv4.addresses "NODE_IP/24" ipv4.gateway "LAN_GATEWAY" ipv4.dns "192.168.0.10" ipv4.dns-search "ocp.openhelp.net" ipv4.ignore-auto-dns yes
sudo nmcli connection up "NODE_CONNECTION"
```

Do not give multiple nodes the same IP. The gateway address is not supplied in the attachment; use the one from your network plan.

Use the VMware console when reactivating a connection, because an SSH session may disconnect. Expect `nameserver 192.168.0.10` in the resolver configuration. On RHCOS, `--copy-network` in the later installation command preserves the configured profile.

From the helper, query IPA directly:

```bash
dig @192.168.0.10 +short bootstrap.ocp.openhelp.net
dig @192.168.0.10 +short master01.ocp.openhelp.net
dig @192.168.0.10 +short master02.ocp.openhelp.net
dig @192.168.0.10 +short master03.ocp.openhelp.net
dig @192.168.0.10 +short worker01.ocp.openhelp.net
dig @192.168.0.10 +short worker02.ocp.openhelp.net
dig @192.168.0.10 +short worker03.ocp.openhelp.net
dig @192.168.0.10 +short api.ocp.openhelp.net
dig @192.168.0.10 +short api-int.ocp.openhelp.net
dig @192.168.0.10 +short test.apps.ocp.openhelp.net
```

Each node name must return its assigned IP. API names and the application wildcard must return the addresses used by your existing HAProxy configuration.

For **each of the seven node IPs**, replace `NODE_IP` below with the address returned for that node:

```bash
dig @192.168.0.10 +short -x NODE_IP
```

The PTR answer must match the intended FQDN in the machine table. Replace any obsolete PTR record for an IP reused by this lab. Use IPA as the DNS server in the nodes' persistent network profiles.

RHCOS can obtain a hostname through reverse DNS when DHCP or a static hostname does not supply one. This procedure uses your IPA A and PTR records for that purpose. A hostname set with `hostnamectl` in the live ISO is not preserved by `coreos-installer --copy-network`.

Test IPA's NTP response from the helper:

```bash
chronyd -Q -t 10 'server 192.168.0.10 iburst'
```

A response that reports a clock offset, such as `System clock wrong by ... seconds (ignored)`, confirms a measurement was obtained. This test does not change the clock. If it times out, verify that IPA permits the node network in chrony's `allow` configuration and that UDP 123 is reachable. On IPA, check `chronyc tracking` and confirm it reports synchronization.

On **IPA `192.168.0.10`**, an appropriate chrony client-access rule is:

```text
allow 192.168.0.0/24
```

Add it to `/etc/chrony.conf` only if no existing rule already covers this network, and restart `chronyd` if you change that file. Permit UDP 123 through IPA's active firewall zone. IPA keeps its own upstream time sources; the helper and cluster nodes use IPA as their time source.

If the **helper** is not already synchronized to IPA, edit its `/etc/chrony.conf`, replace its upstream `server`/`pool` entries with `server 192.168.0.10 iburst`, keep its other chrony settings and restart `chronyd`. Confirm with `chronyc -n sources -v` and `chronyc tracking` before generating assets.

The bootstrap and nodes also need access to their required release container images through your existing gateway, proxy or configured mirror registry. Downloading the ISO and generating Ignition files do not supply those images.

## Step 41 — prepare master and worker NTP configuration

**Run on the helper.** Create these files under `cluster/openshift/`, after manifest generation and before Ignition generation. If they already exist, inspect and correct them. Do not add duplicate MachineConfigs with the same `metadata.name` under both `manifests/` and `openshift/`.

```bash
mkdir -p /home/cloudadmin/ocp-lab/cluster/openshift
nano /home/cloudadmin/ocp-lab/cluster/openshift/99-master-lab-chrony.yaml
```

Paste:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-master-lab-chrony
  labels:
    machineconfiguration.openshift.io/role: master
spec:
  config:
    ignition:
      version: 3.5.0
    storage:
      files:
        - path: /etc/chrony.conf
          mode: 420
          overwrite: true
          contents:
            source: data:,server%20192.168.0.10%20iburst%0Adriftfile%20%2Fvar%2Flib%2Fchrony%2Fdrift%0Amakestep%201.0%203%0Artcsync%0Alogdir%20%2Fvar%2Flog%2Fchrony%0A
```

Save and exit nano. Create the worker file:

```bash
nano /home/cloudadmin/ocp-lab/cluster/openshift/99-worker-lab-chrony.yaml
```

Paste:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-lab-chrony
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.5.0
    storage:
      files:
        - path: /etc/chrony.conf
          mode: 420
          overwrite: true
          contents:
            source: data:,server%20192.168.0.10%20iburst%0Adriftfile%20%2Fvar%2Flib%2Fchrony%2Fdrift%0Amakestep%201.0%203%0Artcsync%0Alogdir%20%2Fvar%2Flog%2Fchrony%0A
```

Save and exit nano. Keep each `source:` value on one physical line. Its encoded contents are these five chrony configuration lines:

```text
server 192.168.0.10 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
```

`mode: 420` means permissions `0644`. The installer incorporates these MachineConfig manifests into the assets used to configure master and worker nodes; they are not files that you pass directly to `coreos-installer`.

The temporary bootstrap machine has a separate time-service configuration. Its manual IPA NTP handoff is included after the node installation commands below.

**If you need the original worker `/var/hpvolumes` data-disk setup, complete Appendix A now, before Step 42.** That preserves the old wrapper's disk function through one worker MachineConfig. You can also leave the data disks untouched and configure VM storage after the cluster is ready.

## Step 42 — generate the three role Ignition files

### 42.1 Save a private checkpoint

Run this once with an unused backup name. It copies your current installation directory, including its installer state and manifests.

```bash
mkdir -p /home/cloudadmin/ocp-lab/backups
chmod 700 /home/cloudadmin/ocp-lab/backups
cp -a /home/cloudadmin/ocp-lab/cluster /home/cloudadmin/ocp-lab/backups/cluster-before-ignition-20261007
```

### 42.2 Generate Ignition

You already generated manifests. Continue from that directory:

```bash
openshift-install create ignition-configs --dir=/home/cloudadmin/ocp-lab/cluster --log-level=info
```

Wait for the command to complete successfully. This is the command that creates the installation Ignition files. Do not manually write the generated role JSON.

### 42.3 Confirm the result

```bash
ls -lh /home/cloudadmin/ocp-lab/cluster/bootstrap.ign /home/cloudadmin/ocp-lab/cluster/master.ign /home/cloudadmin/ocp-lab/cluster/worker.ign
ls -l /home/cloudadmin/ocp-lab/cluster/auth/
date -Is
```

Expected results:

| Created file | Purpose |
|---|---|
| `bootstrap.ign` | Starts the temporary bootstrap machine |
| `master.ign` | Boots each control-plane machine with its role configuration |
| `worker.ign` | Boots each compute machine with its role configuration |
| `auth/kubeconfig` | Administrator CLI access for this installation |
| `auth/kubeadmin-password` | Initial console administrator password |

The installer consumes its manifest inputs during asset generation, so those directories may disappear or become empty afterward. That is expected. The checkpoint retains your editable inputs.

Record the generation time. Use the assets for first boot within **12 hours**, following Red Hat's installation guidance. Keep all machines on assets from this same attempt.

## Step 43 — use the role files directly

There is no additional JSON file to create in this corrected step. Use the machine-to-file table at the start of this document.

Preserve each node's existing static networking through its NetworkManager profile and `--copy-network` at installation. The IPA PTR records provide its hostname after networking starts. Each machine must have a different node IP and the corresponding unique FQDN.

Any later instructions from the original guide referring to the following names must be updated:

| Original download filename | Replacement download filename |
|---|---|
| `bootstrap-host.ign` | `bootstrap.ign` |
| `master01.ign`, `master02.ign`, `master03.ign` | `master.ign` |
| `worker01.ign`, `worker02.ign`, `worker03.ign` | `worker.ign` |

Use the replacement file's SHA512 in the installation command. Seven host-file hashes and nested role-file hashes are no longer part of this workflow.

## Step 44 — publish and verify the three files

### 44.1 Copy to the existing HTTP document directory

**Helper, root shell:**

```bash
mkdir -p /var/www/ocp
cp /home/cloudadmin/ocp-lab/cluster/bootstrap.ign /var/www/ocp/bootstrap.ign
cp /home/cloudadmin/ocp-lab/cluster/master.ign /var/www/ocp/master.ign
cp /home/cloudadmin/ocp-lab/cluster/worker.ign /var/www/ocp/worker.ign
chmod 755 /var/www/ocp
chmod 644 /var/www/ocp/bootstrap.ign /var/www/ocp/master.ign /var/www/ocp/worker.ign
restorecon -RF /var/www/ocp
```

This uses ordinary `cp`, followed by explicit permissions. Apache can read these files without owning them.

The existing HTTP configuration must map URL `/` on `192.168.0.61:8080` to `/var/www/ocp`. The GitHub guide uses an `/ocp4/` URL beneath a different document root. Keep the paths in the table above consistent with your helper's actual configuration.

### 44.2 Confirm the HTTP service and URLs

```bash
httpd -t
systemctl status httpd --no-pager
ss -ltnp 'sport = :8080'
curl --fail --output /dev/null --write-out '%{http_code}\n' http://192.168.0.61:8080/bootstrap.ign
curl --fail --output /dev/null --write-out '%{http_code}\n' http://192.168.0.61:8080/master.ign
curl --fail --output /dev/null --write-out '%{http_code}\n' http://192.168.0.61:8080/worker.ign
```

Expected: Apache configuration `Syntax OK`, service `active (running)`, a listener on 8080, and HTTP `200` for all three files. Repeat the URL check from a RHCOS live console to confirm that the nodes can reach the helper.

If the service has not been started but its existing configuration passes `httpd -t`, start it:

```bash
systemctl enable --now httpd
```

For a `404` or `403`, use Appendix B before proceeding. A local helper test alone does not confirm that its firewall permits node access to TCP 8080.

### 44.3 Calculate three SHA512 digests

```bash
sha512sum /var/www/ocp/bootstrap.ign
sha512sum /var/www/ocp/master.ign
sha512sum /var/www/ocp/worker.ign
```

Record the **128 hexadecimal characters at the beginning** of each result. The filename printed after the digest is not part of it. Copy each digest from the trusted helper terminal into the corresponding node installation command.

| File | Where its digest is used |
|---|---|
| `bootstrap.ign` | Temporary bootstrap installation |
| `master.ign` | All three master installations |
| `worker.ign` | All three worker installations |

For HTTP Ignition URLs, the Red Hat procedure uses `--ignition-hash=sha512-DIGEST`. These are the hashes of the actual generated role files you are serving. Recalculate if you regenerate or change those files.

### 44.4 Keep credentials in the private installation directory

Publish the three required Ignition files only. Keep `auth/`, pull-secret files, private SSH keys, installation-config backups and installer-state backups outside `/var/www/ocp`.

```bash
chmod 600 /home/cloudadmin/ocp-lab/cluster/bootstrap.ign /home/cloudadmin/ocp-lab/cluster/master.ign /home/cloudadmin/ocp-lab/cluster/worker.ign
chmod 600 /home/cloudadmin/ocp-lab/cluster/auth/kubeconfig /home/cloudadmin/ocp-lab/cluster/auth/kubeadmin-password
```

These permissions apply to your private originals. The separately copied web-server files remain readable by Apache.

## Handoff — commands used on the RHCOS live consoles

The following commands replace the original guide's wrapper-based installation commands. Configure each machine's existing static networking in its live ISO first. The network profiles to preserve must be under `/etc/NetworkManager/system-connections/`.

On **each live console**, inspect the disks and networking:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
ip -br address
ip route
nmcli connection show
```

**The OS installation command overwrites its destination disk.** `/dev/sda` below is an example: replace it with that VM's verified OS disk. A worker's separate data disk must not be used as the OS target.

Replace every digest placeholder with the corresponding complete SHA512 from Step 44.3.

### Temporary bootstrap live console

```bash
sudo coreos-installer install /dev/sda --ignition-url=http://192.168.0.61:8080/bootstrap.ign --ignition-hash=sha512-PASTE_BOOTSTRAP_SHA512_HERE --copy-network
```

### Each of master01, master02 and master03 live consoles

```bash
sudo coreos-installer install /dev/sda --ignition-url=http://192.168.0.61:8080/master.ign --ignition-hash=sha512-PASTE_MASTER_SHA512_HERE --copy-network
```

### Each of worker01, worker02 and worker03 live consoles

```bash
sudo coreos-installer install /dev/sda --ignition-url=http://192.168.0.61:8080/worker.ign --ignition-hash=sha512-PASTE_WORKER_SHA512_HERE --copy-network
```

Use the matching RHCOS live ISO for your chosen release. These commands follow the current documented ISO procedure; the older GitHub guide's separate raw-image download is not needed for that procedure.

After each installation command succeeds, disconnect the ISO in VMware and boot that VM from its installed OS:

```bash
sudo reboot
```

The new OS applies Ignition on first boot. Do not copy the RHCOS live session's hostname into your assumptions: `--copy-network` copies network profiles, not `/etc/hostname`. Check the installed node's identity through its console or SSH. If it joins with an incorrect name, correct the provisioning/DNS issue rather than renaming a joined node in place.

### Bootstrap NTP — manually point its installed OS to IPA

The master and worker MachineConfigs do not configure the temporary bootstrap OS's own chrony service. The original `bootstrap-host.ign` handled that separately. In this simplified workflow, configure it manually early in bootstrap startup.

First ensure the helper, VMware host and bootstrap live environment have a correct current clock. After the installed bootstrap OS becomes reachable, connect **from the helper** using your existing key:

```bash
ssh -i /home/cloudadmin/.ssh/ocp_ed25519 core@bootstrap.ocp.openhelp.net
```

On the **installed bootstrap machine**, open:

```bash
sudo vi /etc/chrony.conf
```

In `vi`, press **i** to edit. Set its time source to IPA by replacing its upstream `server`/`pool` entries with `server 192.168.0.10 iburst`. Keep its other chrony settings, including `makestep`. Press **Esc**, type **:wq** and press **Enter** to save and exit. Run:

```bash
sudo systemctl restart chronyd
chronyc -n sources -v
chronyc tracking
```

Wait for IPA to be selected (`^*` next to `192.168.0.10`) and for tracking to report synchronization. Complete this promptly before bringing up the other installed nodes. The helper should also use a synchronized time source.

After the masters and workers boot, verify on a representative node:

```bash
hostname -f
chronyc -n sources -v
```

Expect that node's intended FQDN and IPA as its selected NTP source. Repeat for all six permanent nodes.

Continue with Step 45 on the helper after all machines have booted their installed OS.

## Step 45 — finish bootstrap and update HAProxy

On the **helper**, monitor bootstrap:

```bash
openshift-install wait-for bootstrap-complete --dir=/home/cloudadmin/ocp-lab/cluster --log-level=info
```

Wait until the installer confirms bootstrap is complete and that bootstrap resources can be removed. A timeout is not confirmation; keep bootstrap running while investigating.

Save the current HAProxy configuration in a private directory. Use an unused backup filename if this one already exists:

```bash
mkdir -p /home/cloudadmin/ocp-lab/backups
chmod 700 /home/cloudadmin/ocp-lab/backups
cp -p /etc/haproxy/haproxy.cfg /home/cloudadmin/ocp-lab/backups/haproxy-before-bootstrap-removal.cfg
chmod 600 /home/cloudadmin/ocp-lab/backups/haproxy-before-bootstrap-removal.cfg
nano /etc/haproxy/haproxy.cfg
```

In your existing configuration, comment out or remove only the bootstrap `server` entry in each of the **6443** and **22623** pools. Keep all three masters in those pools. Keep the worker entries in the **80** and **443** ingress pools. Save, then validate:

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
```

After validation succeeds:

```bash
systemctl reload haproxy
systemctl status haproxy --no-pager
```

You can now power off the temporary bootstrap VM. Keep the helper, IPA, all three masters and all three workers running.

## Step 46 — connect the CLI and approve node certificates

On the **helper**, use the credentials from this same installation attempt:

```bash
export KUBECONFIG=/home/cloudadmin/ocp-lab/cluster/auth/kubeconfig
oc whoami
oc get nodes
oc get csr
```

`oc whoami` should show `system:admin`. The kubeconfig authenticates the CLI; the web console will use `kubeadmin` in Step 51.

UPI nodes can require manual approval of both kubelet **client** and **serving** certificate requests. For each `Pending` request, replace `CSR_NAME` with the actual name from `oc get csr` and inspect it:

```bash
oc describe csr CSR_NAME
oc get csr CSR_NAME -o jsonpath='{.spec.request}' | base64 --decode | openssl req -noout -text
```

Match the requested node identity to one of your six intended FQDNs. Client requests use signer `kubernetes.io/kube-apiserver-client-kubelet`; serving requests use `kubernetes.io/kubelet-serving` and must contain the appropriate node DNS/IP identities. Approve the requests belonging to your nodes:

```bash
oc adm certificate approve CSR_NAME
```

Approve pending client requests first, then check again for serving requests generated as those nodes join:

```bash
oc get csr
oc get nodes
```

Inspect and approve each matching pending serving request in the same way. Repeat as needed until all three masters and all three workers appear and reach `Ready`. Do not approve unknown requests merely because they are pending.

The expected permanent node names are:

```text
master01.ocp.openhelp.net
master02.ocp.openhelp.net
master03.ocp.openhelp.net
worker01.ocp.openhelp.net
worker02.ocp.openhelp.net
worker03.ocp.openhelp.net
```

Bootstrap is not a permanent cluster node. `compute.replicas: 0` in a UPI install does not provision workers for you; your three worker VMs must actually be installed, booted and joined.

## Step 47 — check cluster and ingress readiness

On the **helper**:

```bash
oc get nodes
oc get clusteroperators
oc get machineconfigpools
oc get pods -n openshift-ingress -o wide
oc get pods -n openshift-console
oc get pods -n openshift-authentication
```

Allow the initial rollout and MachineConfig updates to finish. Expect six `Ready` nodes; cluster operators settling to `AVAILABLE=True`, `PROGRESSING=False`, `DEGRADED=False`; and master/worker machine config pools with `UPDATED=True`, `UPDATING=False`, `DEGRADED=False`. Router, console and OAuth pods should have their containers ready.

For this separate-worker layout, router pods normally run on workers. HAProxy must forward TCP 80/443 to those workers. The console and OAuth routes both depend on ingress.

On `platform: none`, the image registry can initially have management state `Removed` because automatic registry storage is unavailable. Check it with:

```bash
oc get configs.imageregistry.operator.openshift.io cluster -o jsonpath='{.spec.managementState}{"\n"}'
```

Do not change registry storage merely to open the console. Configure appropriate registry storage separately when enabling internal image builds and pushes.

If an operator remains degraded, inspect that specific operator using its name from the list:

```bash
oc describe clusteroperator OPERATOR_NAME
```

## Step 48 — wait for installation completion

On the **helper**:

```bash
openshift-install wait-for install-complete --dir=/home/cloudadmin/ocp-lab/cluster --log-level=info
```

Successful output confirms installation completion and supplies a console URL and initial administrator credentials. If it times out, check Step 46's pending CSRs and Step 47's operators before trying again. A timeout does not require regenerating Ignition or reinstalling healthy nodes.

## Step 49 — confirm the console and OAuth routes

On the **helper**, with the exported kubeconfig:

```bash
oc whoami --show-console
oc get route console -n openshift-console
oc get route oauth-openshift -n openshift-authentication
dig @192.168.0.10 +short console-openshift-console.apps.ocp.openhelp.net
dig @192.168.0.10 +short oauth-openshift.apps.ocp.openhelp.net
```

For your cluster name and base domain, the normal console address is:

```text
https://console-openshift-console.apps.ocp.openhelp.net
```

The normal OAuth hostname is `oauth-openshift.apps.ocp.openhelp.net`. Use the actual route hosts returned by your cluster if you customized them. Both DNS queries should reach the same ingress load-balancer address covered by IPA's `*.apps.ocp.openhelp.net` wildcard. A console-only hosts-file entry is insufficient because login redirects to OAuth.

## Step 50 — make your Windows browser computer use IPA for the lab

These commands run on your **Windows computer**, not on the helper. It must have a network path to the ingress load-balancer address on TCP 443.

If Windows already uses IPA as its DNS server, skip the NRPT changes and run the resolution tests below. If it uses another DNS server, use the following conditional DNS rule to send this lab's queries to IPA while keeping your other DNS settings.

Open **PowerShell as Administrator** and inspect existing rules:

```powershell
Get-DnsClientNrptRule
```

If there is no existing rule for `.ocp.openhelp.net`, add one:

```powershell
Add-DnsClientNrptRule -Namespace ".ocp.openhelp.net" -NameServers "192.168.0.10"
```

If a matching rule already points to `.10`, keep it. If it points to an old DNS address, replace `EXISTING_RULE_NAME` with its actual `Name` from the earlier output and update that rule instead of adding a duplicate:

```powershell
Set-DnsClientNrptRule -Name "EXISTING_RULE_NAME" -NameServers "192.168.0.10"
```

Then test:

```powershell
Clear-DnsClientCache
Resolve-DnsName console-openshift-console.apps.ocp.openhelp.net
Resolve-DnsName oauth-openshift.apps.ocp.openhelp.net
Test-NetConnection console-openshift-console.apps.ocp.openhelp.net -Port 443
```

Expected: both names resolve to your ingress load-balancer address and `TcpTestSucceeded` is `True`. A public DNS server does not know your private IPA lab records. If OS resolution succeeds but the browser reports a DNS failure, check whether browser Secure DNS is bypassing the Windows lab DNS rule.

## Step 51 — open the console and log in

On the **helper**, display the password from this installation attempt:

```bash
cat /home/cloudadmin/ocp-lab/cluster/auth/kubeadmin-password
```

On the **Windows browser computer**, open the console URL from Step 49 in Edge or Chrome. If the browser reports an untrusted certificate, complete Step 52 and reopen the URL. On the login page, select the `kube:admin` provider if a provider choice appears, then enter:

| Field | Value |
|---|---|
| Username | `kubeadmin` |
| Password | Contents of this installation's `auth/kubeadmin-password` |

The `core` SSH account and its SSH key are for node access. They are not console login credentials. Using IPA for DNS and NTP does not automatically enable IPA user login to OpenShift; IPA authentication integration is a separate configuration.

After logging in, check cluster health in the Overview page and confirm all six nodes under **Compute → Nodes**. Installation and UI access are complete when the installer succeeds and you can log in to a healthy console.

## Step 52 — trust the default ingress CA on your lab browser computer

Use this for a new lab that is still using OpenShift's generated default ingress certificate. On the **helper**, first check for a custom default certificate:

```bash
oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.spec.defaultCertificate.name}{"\n"}'
```

If this returns a certificate secret name, use the CA that issued your custom certificate instead; do not assume `router-ca` is its issuer. If the result is blank, export only the generated CA's public certificate:

```bash
oc get secret router-ca -n openshift-ingress-operator -o jsonpath='{.data.tls\.crt}' | base64 --decode > /home/cloudadmin/ocp-lab/ocp-ingress-ca.crt
chmod 644 /home/cloudadmin/ocp-lab/ocp-ingress-ca.crt
openssl x509 -in /home/cloudadmin/ocp-lab/ocp-ingress-ca.crt -noout -subject -issuer -fingerprint -sha256
```

Copy `ocp-ingress-ca.crt` from the helper into your Windows Downloads directory using your existing SSH/SFTP access. For example, if `cloudadmin` can log in to the helper, run on **Windows**:

```powershell
scp cloudadmin@192.168.0.61:/home/cloudadmin/ocp-lab/ocp-ingress-ca.crt "$env:USERPROFILE\Downloads\ocp-ingress-ca.crt"
certutil -dump "$env:USERPROFILE\Downloads\ocp-ingress-ca.crt"
```

Confirm this is the CA certificate you exported from your own cluster. Then import that lab CA into your Windows user's trusted-root store:

```powershell
Import-Certificate -FilePath "$env:USERPROFILE\Downloads\ocp-ingress-ca.crt" -CertStoreLocation "Cert:\CurrentUser\Root"
```

Close and reopen Edge/Chrome, then retry Step 51. IPA's DNS/NTP configuration does not automatically replace or trust OpenShift's ingress certificate. A custom trusted ingress certificate can be configured later.

## Console access troubleshooting

| Symptom | First checks |
|---|---|
| Name does not resolve | IPA wildcard; Windows DNS/NRPT; console and OAuth hostnames |
| TCP 443 times out | Windows route to ingress address; helper HAProxy listener; active firewall zone |
| HTTP 503 | HAProxy worker backends; ready router pods; console/authentication operators |
| Console opens but login redirect fails | OAuth hostname resolution and OAuth/ingress readiness |
| Certificate warning | Step 52; certificate hostname and correct system time |
| Login rejected | Username `kubeadmin`; password file from this same installation attempt |
| Workers missing or `NotReady` | Worker VM boots; static network/IPA DNS; pending client and serving CSRs |

The browser uses the application hostname on **HTTPS 443**. Helper port **8080** serves Ignition downloads and does not host the OpenShift console.

## Appendix A — optional preservation of the original worker data-disk setup

The old worker wrappers partitioned and formatted `/dev/sdb` as XFS and mounted it at `/var/hpvolumes`. This is a storage choice, not a prerequisite for generating the standard Ignition role files.

For the same first-boot storage behavior, add one worker MachineConfig **before Step 42**. It applies to every worker in the default worker pool.

**Use it only after confirming all three workers have an empty, expendable data disk at `/dev/sdb`, separate from their OS disks. This configuration erases that data disk.** If paths differ across workers, or the disks contain data, omit this shared manifest and prepare the storage separately after installation. Do not apply this disk-formatting example to an already running cluster.

On the helper:

```bash
nano /home/cloudadmin/ocp-lab/cluster/openshift/98-worker-lab-hpvolumes.yaml
```

Paste:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 98-worker-lab-hpvolumes
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.5.0
    storage:
      disks:
        - device: /dev/sdb
          wipeTable: true
          partitions:
            - number: 1
              label: hpp-data
              sizeMiB: 0
      filesystems:
        - device: /dev/disk/by-partlabel/hpp-data
          path: /var/hpvolumes
          format: xfs
          wipeFilesystem: true
    systemd:
      units:
        - name: var-hpvolumes.mount
          enabled: true
          contents: |
            [Unit]
            Description=Lab HPP data disk
            Before=local-fs.target

            [Mount]
            What=/dev/disk/by-partlabel/hpp-data
            Where=/var/hpvolumes
            Type=xfs
            Options=defaults

            [Install]
            WantedBy=local-fs.target
```

Save and exit, then continue Step 42. This retains the original disk, partition label, mount point and mount-unit behavior while allowing every worker to use the same generated `worker.ign`.

After installation, verify on each worker:

```bash
lsblk -f
findmnt /var/hpvolumes
systemctl status var-hpvolumes.mount --no-pager
```

Expected: an XFS partition labelled `hpp-data` mounted at `/var/hpvolumes`. The mount provides the filesystem; any HostPath Provisioner/operator setup remains a separate step in your virtualization guide.

## Appendix B — HTTP directory troubleshooting

Use this only if Step 44's URL tests fail. Review your earlier HTTP configuration:

```bash
cat /etc/httpd/conf/httpd.conf
ls -l /etc/httpd/conf.d/
```

You need a single effective listener for `192.168.0.61:8080`, and a matching document root of `/var/www/ocp`. Review any existing OpenShift HTTP configuration before adding another virtual host.

If no existing configuration provides that mapping, manually set the relevant `Listen` directive in `httpd.conf` to:

```apache
Listen 192.168.0.61:8080
```

Avoid keeping Apache's default wildcard `Listen 80` when HAProxy serves HTTP on the same helper. Configure a matching virtual host in one chosen file, for example `/etc/httpd/conf.d/ocp-files.conf`:

```apache
<VirtualHost 192.168.0.61:8080>
    DocumentRoot "/var/www/ocp"
    <Directory "/var/www/ocp">
        Options -Indexes
        AllowOverride None
        Require ip 192.168.0.0/24
    </Directory>
</VirtualHost>
```

Validate before restarting:

```bash
httpd -t
```

After `Syntax OK`:

```bash
systemctl restart httpd
```

Ensure the helper's active firewall zone for the node-facing interface permits TCP 8080. Keep SELinux enforcing; use `restorecon` for the document tree and inspect service/SELinux errors if Apache cannot bind the port or read a file.

| Result | Check |
|---|---|
| HTTP `404` | File name and the effective `DocumentRoot`/URL mapping |
| HTTP `403` | Directory access rule, permissions and SELinux file context |
| Connection refused | HTTP service, listener address and port |
| Timeout from nodes only | Routing and the helper's node-facing firewall zone |

## If Ignition files already existed before the corrections

Installer assets are cached. Re-running `create ignition-configs` in an existing directory is not a reliable way to incorporate edits made after Ignition generation.

For a **new attempt before any node has booted**, retain the old directory and create a separate working directory. Restore your original, complete `install-config.yaml` backup into that new directory, review it, run `create manifests`, set `mastersSchedulable: false`, add the corrected MachineConfigs, and run `create ignition-configs`. Use that new directory consistently in every subsequent command and publish its three role files together.

If nodes have already started an installation, keep their current assets and state while diagnosing it. Mixing fresh files from another attempt with existing bootstrap/control-plane machines can break the installation. A fresh attempt requires a coordinated reinstall of the participating machines.

If your original `install-config.yaml` backup is unavailable, reconstruct a complete configuration in a new directory using the existing lab settings, your actual Red Hat pull secret and the public key you already created. The public key is:

```yaml
sshKey: 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIO7XiayqHYWe1BW0NPHBkCwOVWDdEKszRtV7DHh9xNb5 ocp-openhelp-lab'
```

This is an input for a new installation configuration. Continue your current pre-Ignition attempt directly from its existing manifests when they are available.

## Sources and validation

- [Requested GitHub training guide — Generate and host install files](https://github.com/krnetworktraining1/ocp4-metal-install#generate-and-host-install-files). Its README was retrieved directly from the repository's `master` branch. It generates three role files; its older networking, hosting path and installation examples were adapted to your helper and IPA services.
- [Red Hat OpenShift 4.20 — Creating manifests and Ignition files, section 1.12](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_any_platform/installing-platform-agnostic).
- [Red Hat OpenShift 4.20 — Installing on any platform, including DNS, networking and ISO installation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_any_platform/installing-platform-agnostic).
- [Red Hat OpenShift 4.20 — Bootstrap completion, node CSRs and completing UPI installation, sections 1.14–1.18](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_any_platform/installing-platform-agnostic).
- [Red Hat OpenShift 4.20 — Accessing the web console](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/web_console/web-console).
- [Red Hat OpenShift 4.20 — Ingress certificates, section 4.11](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/security_and_compliance/certificate-types-and-descriptions).
- [Microsoft — Add-DnsClientNrptRule](https://learn.microsoft.com/en-us/powershell/module/dnsclient/add-dnsclientnrptrule).
- [Microsoft — Set-DnsClientNrptRule](https://learn.microsoft.com/en-us/powershell/module/dnsclient/set-dnsclientnrptrule).
- [Microsoft — Import-Certificate](https://learn.microsoft.com/en-us/powershell/module/pki/import-certificate).
- [Red Hat OpenShift 4.20 — Configure chrony time service](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/machine_configuration/machine-configs-configure).
- [Red Hat OpenShift 4.20 — Customizing nodes and disk partitioning](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installation_configuration/installing-customizing).
- [CoreOS Installer — install command reference](https://coreos.github.io/coreos-installer/cmd/install/).
- [Ignition configuration specification 3.5.0](https://coreos.github.io/ignition/configuration-v3_5/).

Validation performed on this document: MachineConfig YAML parsing, role labels, decoded chrony configuration, worker disk/mount consistency, shell-command syntax, agreement of all role URLs, and review of the installation/console sequence against official documentation. Your helper and cluster were not accessed; the commands above perform the environment checks on your machines.
