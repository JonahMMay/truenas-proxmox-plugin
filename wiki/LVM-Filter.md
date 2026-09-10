# LVM Filter for TrueNAS NVMe/TCP

## The problem

If a VM's disk lives on a TrueNAS NVMe/TCP namespace and the guest
inside that VM uses LVM (default on Ubuntu, RHEL, and their
derivatives), the Proxmox host's LVM scanner sees the guest's PV
signatures at the outer namespace level and treats them as
host-level physical volumes.

Symptoms an operator will actually see:

- Every LVM invocation on the host (`pvesm status`, `pvesh get
  /nodes/<node>/storage`, `qm ...`, `pvestatd` polls) emits pages
  of `WARNING: VG name <name> is used by VGs <uuid1> and <uuid2>.
  Fix duplicate VG names with vgrename uuid, ...` lines. With many
  namespaces cloned from the same template the duplicates cascade
  into thousands of lines.
- `pvesh get /nodes/<node>/storage` slows enough to trip the 596s
  API timeout, which in turn breaks external tools that poll it
  (Veeam's backup/restore scans have been observed to fail here).
- Boot is slower because `pvscan --cache` runs on every namespace
  udev-add event.

Reference: GitHub issue [#4](https://github.com/truenas/truenas-proxmox-plugin/issues/4).

## The fix

Add a reject regex to LVM's `global_filter` in `/etc/lvm/lvm.conf`
so LVM stops scanning TrueNAS-served NVMe namespaces:

```
"r|/dev/disk/by-id/nvme-TrueNAS_.*|"
```

Why this works:

- `udev` populates `/dev/disk/by-id/nvme-<model>_<serial>` symlinks
  from the NVMe controller MODEL string. TrueNAS's NVMe controllers
  advertise `TrueNAS <hardware-model>` (e.g. `TrueNAS PowerEdge
  R730xd`, `TrueNAS FREENAS-MINI-2.0`), so every TrueNAS namespace
  ends up with a `nvme-TrueNAS_...` by-id name.
- `pve-manager` sets `obtain_device_list_from_udev = 1` (LVM's
  default), so LVM's scan enumerates by-id symlinks and matches
  them against `global_filter`.
- Local NVMe drives use their own vendor prefix (e.g.
  `nvme-Samsung_...`, `nvme-Intel_...`) and never match, so host
  LVM (root filesystem VGs, local-lvm, etc.) is untouched.

## Applying the fix

### Automated (recommended)

The plugin ships a helper at `/usr/sbin/truenas-plugin-lvm-filter`:

```
sudo truenas-plugin-lvm-filter --install
sudo pvscan --cache
sudo vgscan
```

`--install` appends the regex to the existing `global_filter` line
in `/etc/lvm/lvm.conf`, tagged with a comment marker
(`# truenas-proxmox-plugin issue #4`) for future removal. A
timestamped backup is written first, and `lvm dumpconfig` is run
after the edit to verify the file still parses.

Reversal:

```
sudo truenas-plugin-lvm-filter --uninstall
```

`--uninstall` removes only the regex the plugin added; anything
else in `global_filter` (including `pve-manager`'s default
`"r|/dev/zd.*|"` and `"r|/dev/rbd.*|"` for ZFS zvols and Ceph
rbds) is left in place.

Status check (does not need root):

```
truenas-plugin-lvm-filter --status
```

The script is idempotent: running `--install` when the filter is
already installed, or `--uninstall` when it isn't, is a no-op.

### Manual

Edit `/etc/lvm/lvm.conf`, find the `global_filter =` line inside
the `devices { ... }` section, and append `"r|/dev/disk/by-id/nvme-TrueNAS_.*|"`
to the list. Save. Refresh LVM:

```
sudo pvscan --cache
sudo vgscan
```

The `pve-manager`-provided line typically looks like:

```
global_filter = [ "r|/dev/zd.*|", "r|/dev/rbd.*|" ]
```

After the fix:

```
global_filter = [ "r|/dev/zd.*|", "r|/dev/rbd.*|", "r|/dev/disk/by-id/nvme-TrueNAS_.*|" ]
```

## Why this isn't done at package install

`/etc/lvm/lvm.conf` is a Debian conffile owned by the `pve-manager`
package. It is intentionally NOT modified by
`truenas-proxmox-plugin`'s postinst, because:

- Debian policy discourages one package silently editing another
  package's conffile.
- LVM configuration is systemically load-bearing; a silent change
  applied on plugin install could surprise operators debugging
  unrelated LVM issues later.

`truenas-plugin-lvm-filter --install` exists so the sysadmin can
opt in with a clear audit trail (backup + marker comment + explicit
consent).

## Scope and caveats

- **NVMe/TCP only.** iSCSI LUNs from TrueNAS appear as `/dev/sd*`
  and can hit the same class of duplicate-VG warnings; a similar
  filter for those would need a matcher against SCSI vendor/model
  (`/dev/disk/by-id/scsi-*TrueNAS*` for example). Not shipped
  because it has not been reported. Ask if you want it.
- **Requires `obtain_device_list_from_udev = 1`** (LVM's default).
  If a local operator has set `obtain_device_list_from_udev = 0`,
  LVM will not enumerate the by-id symlinks and the filter has no
  effect.
- **The right long-term home for this fix is `pve-manager` itself**,
  in the same default `global_filter` that already excludes ZFS
  zvols and Ceph rbds. Filing an upstream patch is a separate
  planned follow-up.
