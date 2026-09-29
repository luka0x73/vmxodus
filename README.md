# vmxodus

Migrates VMware VMs to Proxmox VE with minimal downtime, without copying disk data.

The disks are moved beforehand with Storage vMotion onto an NFS share that both ESXi and PVE mount. vmxodus then runs on a PVE node:

1. **prepare** (VM still running in VMware): reads the `.vmx`, creates an empty PVE VM with the same CPU, memory, firmware and NICs, and maps every portgroup to a PVE bridge.
2. Shut the VM down in vCenter.
3. **cutover**: renames the raw `-flat.vmdk` files into the PVE image layout (`mv` on the same filesystem, no copy, no conversion) and attaches them.

Afterwards, move the disks to their final storage with `qm disk move` or the GUI. vmxodus prints the commands but does not run them.

## Requirements

- Proxmox VE 9.x with `qm`, `pvesh`, `pvesm`, `qemu-img`, bash 5
- The NFS share configured as PVE storage of type `nfs` (content `images`) and mounted as an NFS v3 datastore on ESXi
- Thick or thin VMFS disks with one flat extent, no snapshots

## Tested with

- VMware ESXi 8.0 with vCenter 8.0, VMs on an NFS v3 datastore
- Proxmox VE 9.2 (pve-manager/9.2.20) with NFS storage
- Guests Debian 13 and Ubuntu 24.04: BIOS, pvscsi, vmxnet3, two disks

Windows and UEFI guests are supported by the code but not yet tested on real hardware.

## Before you migrate

In vSphere:

- Disable backup jobs for the VM until the cutover is done.
- Delete all snapshots and wait until the consolidation has finished. vmxodus refuses VMs with snapshots.
- Mount the export on ESXi as NFS v3. Lock detection relies on the `.lck-*` files ESXi creates on NFS v3.
- The export needs `no_root_squash` for the ESXi hosts.

In the guest:

- Uninstall VMware Tools and install `qemu-guest-agent`.
- Linux needs `virtio_scsi` in the initramfs.
- Windows needs the VirtIO drivers installed beforehand, or use `--disk-bus=sata`.
- Interface names change (e.g. `ens192` becomes `ens18`) while the MAC addresses are kept. Bind the network configuration to the MAC, not to the interface name.

## Installation

```bash
install -m 0755 vmxodus /usr/local/sbin/vmxodus
install -m 0644 vmxodus.conf.example /etc/vmxodus.conf
$EDITOR /etc/vmxodus.conf   # set NFS_STORE at least
```

`vmxodus.conf.example` lists every key with a short comment. `NFS_STORE` is required, vmxodus aborts without it.

## Usage

```
vmxodus list
vmxodus prepare <vm> [--vmid=<id>] [--disk-bus=<bus>] [--set key=value]... [--yes] [--dry-run]
vmxodus cutover <vm> [--start] [--wait[=<seconds>]] [--yes] [--dry-run]
vmxodus reset <vm> [--yes] [--dry-run]
```

`reset` undoes `prepare` as long as no flat file was moved: it archives the state and rollback file and prints the `qm destroy` command for the empty PVE VM without running it. Migrated VMs are refused.

`<vm>` is the VM folder name on the share. `--dry-run` prints every mutating command instead of running it.

Disks are attached on the bus from `DISK_BUS` or `--disk-bus=` (scsi, sata or virtio). Additional `qm create` options come from `EXTRA_OPTS` in the config, `--set key=value` or the interactive prompt after the summary. Precedence, low to high: defaults, config, values from the `.vmx`, `--set`/prompt. Disk slots, `boot`, the firmware (`bios`, always taken from the `.vmx`), `efidisk0` and `vmid` are managed by vmxodus and cannot be set.

Portgroup to bridge assignments are kept in `<STATE_DIR>/portgroups.map` (tab separated: key, bridge, tag, label; `-` means empty) and offered as defaults next time:

```
dvportgroup-1001	vmbr1	34	dmz-servers
VM Network	vmbr0	-	-
```

State, logs and rollback files live in `<STATE_DIR>` on the share. `cutover` must run on the node where `prepare` ran. Each completed `mv` is recorded in `<vm>.rollback` as its reverse `mv`; the file holds only the current cutover, earlier ones are archived as `<vm>.rollback.<timestamp>`.

## Safety

- Nothing is ever deleted, copied or converted. No `rm`, no `qm destroy`, nothing on the VMware side.
- Every check runs before the first `mv`: locks, snapshots (`.vmx`, descriptors, `.vmsd`), unchanged disk list since `prepare`, raw format, same filesystem, free targets.
- After each `mv` the result is verified on disk (source gone, target present with the expected size).
- The state file on the share is never sourced. A strict parser accepts only the exact format vmxodus writes.
- If `cutover` stops for any reason (error, failed command, signal), it reports the last step and what to do with the rollback file.

## Rollback

`cutover` writes `<STATE_DIR>/<vm>.rollback`. It contains one reverse `mv` per flat file that was moved in the current cutover, appended right after each move. Running it moves the flat files back into the VM folder, where the VMware VM expects them:

```bash
bash <STATE_DIR>/<vm>.rollback
```

- **Cutover aborted before `qm set`** (status stays `prepared`): the abort report says how many files were moved. Run the rollback file, then `vmxodus reset <vm>` to archive state and rollback file. `reset` refuses as long as any flat file is missing from the VM folder or a target exists in `images/<VMID>`. It prints `qm destroy <VMID>` for the now empty PVE VM but never runs it.
- **After a successful cutover** (status `migrated`): the disks are attached to the PVE VM. `reset` refuses. Stop the VM, detach the disks (`qm set <VMID> --delete scsi0,scsi1` keeps the files as unused disks), then run the rollback file. Only remove the PVE VM after the flat files are back in the VM folder.

**`qm destroy` deletes every disk volume of the VM, attached or unused.** Never run it, and never remove disks in the GUI, while migrated flat files are still in `images/<VMID>`.

Rollback only works:

- before the VM was started in PVE. Once the guest has run on PVE, its disks have changed and the guest was adapted to the new hardware.
- before the disks were moved to another storage. `qm disk move ... --delete 1` removes the raw files on the share.

## Limitations

- Not supported: vSAN, encrypted VMs, vTPM state, RDM disks, multi-extent or sparse descriptors, independent-nonpersistent disks.
- The smbios UUID is not preserved, PVE generates a new one.

## License and disclaimer

MIT, see [LICENSE](LICENSE).

vmxodus renames the only copy of your VM's disks. Test it with throwaway VMs first. Use at your own risk.
