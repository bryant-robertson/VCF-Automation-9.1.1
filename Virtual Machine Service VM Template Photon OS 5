# Photon OS Base Image for VCF Automation VM Service

Procedure for building and updating a Photon OS 5 base image that deploys correctly through the **Virtual Machine Service** in a VCF Automation 9.1.x All Apps organization, including cloud-init guest customization and load balancer access.

## Background

Images deployed by the VM Service are customized at first boot through **cloud-init**, which reads its configuration from VMware guestinfo properties (the `VMware` datasource).

With `cloud-init-26.2-3.ph5`, cloud-init never runs on first boot. The package installs `ds-identify` at `/usr/libexec/cloud-init/ds-identify`, but the systemd generator looks for it at `/usr/lib/cloud-init/ds-identify`. The generator fails (`rc=127`) and cloud-init is never enabled for the boot. As a result, VMs deployed with Cloud-Init customization get:

- no IPv4 address (only a link-local IPv6 address)
- the default hostname `photon`
- no cloud-init users
- the condition `VirtualMachineGuestNetworkConfigSynced: False — Neither IPv4 nor IPv6 address reported by guest`

Step 3 below works around this with a symlink.

## Prerequisites

- A Photon OS 5 build VM in vCenter, kept as a regular powered-off VM (not converted to a template)
- Console access to the build VM as root
- A content library in the VCF Automation organization attached to the target namespace(s)

---

## 1. Power on the build VM

Power on the build VM in vCenter. If it was converted to a template, convert it back to a VM first. Log in as root through the console.

## 2. Install and update software

```bash
tdnf update -y
# install any additional software for the image here
```

> **Note:** If `tdnf update` installed a new cloud-init version, re-run the check in step 3 afterward.

## 3. Fix the cloud-init ds-identify path

```bash
mkdir -p /usr/lib/cloud-init
[ -e /usr/lib/cloud-init/ds-identify ] || ln -s /usr/libexec/cloud-init/ds-identify /usr/lib/cloud-init/ds-identify
ls -l /usr/lib/cloud-init/ds-identify
```

## 4. Confirm cloud-init and VMware Tools are ready

```bash
# All four should report "enabled"
systemctl is-enabled cloud-init-local cloud-init cloud-config cloud-final

# Remove the disable flag if present
ls /etc/cloud/cloud-init.disabled 2>/dev/null && rm -f /etc/cloud/cloud-init.disabled

# datasource_list must include 'VMware'
grep -n "datasource_list" /etc/cloud/cloud.cfg

# Should return nothing (cloud-init networking must not be disabled)
grep -rn "config: disabled" /etc/cloud/cloud.cfg /etc/cloud/cloud.cfg.d/

# Should report "enabled"
systemctl is-enabled vmtoolsd
```

## 5. Check the network configuration

```bash
ls /etc/systemd/network/
```

- **Keep** `50-dhcp-en.network` and `99-dhcp-en.network` (DHCP fallback).
- **Remove** any static `.network` files added during the build.
- **Remove** any `10-cloud-init-*.network` files.

The VM Service supplies network configuration at deploy time, so none should be baked into the image.

## 6. Open required firewall ports

Photon OS allows only SSH inbound by default. Open any ports your workloads need, for example HTTPS:

```bash
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables-save > /etc/systemd/scripts/ip4save
```

Repeat the `iptables -A` line for each additional port, then save again.

## 7. Disable root password expiration

```bash
chage -M -1 root
```

## 8. Clean up and generalize

Run this as the **final** step before shutdown.

```bash
tdnf clean all
cloud-init clean --logs --seed
truncate -s 0 /etc/machine-id
rm -f /etc/ssh/ssh_host_*
rm -rf /tmp/* /var/tmp/*
journalctl --rotate && journalctl --vacuum-time=1s
unset HISTFILE; rm -f /root/.bash_history
shutdown -h now
```

> **Note:** Removing the SSH host keys is safe. cloud-init regenerates them on each VM's first boot.

## 9. Publish the image

Choose one method.

### Option A: Export and upload

1. In vCenter, select the powered-off VM and choose **Actions → Template → Export OVF Template**.
2. Package the export as a single OVA:
   ```bash
   ovftool photon-custom.ovf photon-custom.ova
   ```
3. In the VCF Automation organization portal, go to **Build & Deploy → Content Hub → Content Libraries → *your library* → VM Images → Upload** and upload the OVA.

### Option B: Clone to a subscribed vCenter library

1. Right-click the VM in vCenter and choose **Clone → Clone to Template in Library**.
2. Set the template type to **OVF** and select a published vCenter content library.
3. The VCF Automation content library subscribed to that vCenter library syncs the new image automatically.

## 10. Test deploy

In the organization portal, go to **Build & Deploy → Services → Virtual Machine → Create VM** and deploy from the new image with:

- **Guest Customization:** Cloud-Init, with a user defined
- **Load Balancer:** **New** (not Existing), with the required ports (for example TCP 22 and 443)

> **Warning:** Do not choose an **Existing** load balancer that belongs to a VKS cluster (for example `kubernetes-cluster-xxxx`). Doing so adds the VM to the cluster's API load balancer on port 6443.

### Validation checklist

- [ ] The VM details page shows an IPv4 address
- [ ] The hostname is `vm-xxxx`, not `photon`
- [ ] The cloud-init user can log in from **Open Web Console**
- [ ] Under **Manage & Govern → Networking → IP Management → *external IP block* → Used IPs**, a `vm-lb-xxxx_<port>_TCP` entry appears for each load balancer port
- [ ] `https://<VIP>` responds once an application is listening on 443 in the guest

---

## Troubleshooting: cloud-init did not run

If a deployed VM has no IPv4 address or the cloud-init user does not exist, log in as root from **Open Web Console** and run:

```bash
cloud-init status --long
cat /run/cloud-init/cloud-init-generator.log
vmware-rpctool "info-get guestinfo.userdata" | head -c 80; echo
```

| Symptom | Cause | Fix |
| --- | --- | --- |
| `status: not started` and generator log shows `no ds-identify in /usr/lib/cloud-init/ds-identify` / `rc=127` | ds-identify path mismatch | Step 3, then rebuild the image |
| `guestinfo.userdata` is empty | Cloud-Init customization not delivered | Check the VM's Guest Customization settings |
| `/etc/cloud/cloud-init.disabled` exists | cloud-init disabled in the image | Step 4, then rebuild the image |
| `datasource_list` does not include `VMware` | Datasource excluded | Add `VMware` to `datasource_list`, then rebuild the image |

To repair an already-deployed VM without rebuilding, apply the step 3 symlink on that VM and reboot. Because cloud-init never completed on it, it will process the VM Service configuration on the next boot.

## Updating the image later

Keep the build VM powered off in vCenter. To produce a new image version, repeat this procedure from step 1.
