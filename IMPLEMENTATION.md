# Implementation — evidence-backed reconstruction

> This document reconstructs the lab workflow from the supplied screenshots and LinkedIn project description. Steps and menu names are instructive reproductions, **not a claim that the commands below were captured in an execution transcript**.

## Prerequisites

- AWS account with required EC2/EBS privileges and awareness of running costs
- A disposable Windows Server EC2 instance (this project used `SRV-2A` in `eu-west-2`)
- A secondary EBS data volume (the example used `gp3`, 2 GiB), formatted and mounted in Windows
- Permission to create EBS snapshots, new volumes and EC2 volume attachments
- Windows Disk Management access

## Stage 1 — Establish the baseline

1. In the AWS EC2 console, open **Instances** and verify the Windows instance is running.
2. Open **Volumes** and identify the dedicated data volume. Do not confuse it with the Windows OS/root volume.
3. Confirm the original data volume is attached to the expected instance.
4. In Windows, inspect `D:` and create a harmless test folder called `Before-Snapshot`.
5. Ideally save a text file inside the folder and record a checksum; the current evidence confirms folder names, not file checksums.

**Evidence:** [Windows instance and volume attachments](../evidence/01-windows-ec2-and-attached-volumes.png)

## Stage 2 — Capture the snapshot

1. In EC2 → **Volumes**, select the correct 2 GiB data volume.
2. Choose **Actions → Create snapshot**.
3. Give it a meaningful name or tag such as `D-Drive-Snapshot`.
4. Open **Snapshots**, refresh until status becomes **Completed**.
5. Record the snapshot creation timestamp and volume association in a private change log.

**Observed:** snapshot `D-Drive-Snapshot` shows **100% / Completed** and the volume size as **2 GiB**. The visible full snapshot size is approximately **13 MiB**.

**Evidence:** [Completed snapshot](../evidence/03-snapshot-completed.png)

## Stage 3 — Introduce controlled data drift

On the existing `D:` drive, create another folder: `After-Snapshot`.

Do this **after** the snapshot request was made so the backup represents a state before this change. Avoid real or sensitive business data in a disposable lab.

For illustration only (not a screenshot-backed command transcript):

```powershell
New-Item -ItemType Directory -Path 'D:\Before-Snapshot' -Force
# Take the snapshot at this point, then:
New-Item -ItemType Directory -Path 'D:\After-Snapshot' -Force
```

## Stage 4 — Rebuild volume from backup

1. Open EC2 → **Snapshots** and select `D-Drive-Snapshot`.
2. Choose **Create volume from snapshot**.
3. Select the AZ of the target EC2 instance; a volume cannot attach across AZs.
4. Create a new **2 GiB** restored volume. The screenshot lists a restored volume named `D-Drive-Restore`.
5. In **Volumes**, confirm original and new volumes exist.

**Evidence:** [Original and restored EBS volumes](../evidence/02-original-and-restored-ebs-volumes.png)

## Stage 5 — Attach and mount safely

1. Select the restored volume and attach it to `SRV-2A`.
2. On Windows, open **Disk Management** and locate the newly attached disk carefully by size and state.
3. If Windows marks it offline, investigate **why**; do not format or initialize a restored disk containing expected data.
4. Bring the verified restored disk online using the appropriate non-destructive procedure and expose it as `E:`.
5. Confirm `D:` still points to the original volume.

**Caution:** Console-assigned Linux-style device names such as `xvdb`/`xvdc` are not necessarily the same as Windows disk numbers or letters.

## Stage 6 — Final comparison

Open File Explorer windows side-by-side:

| Drive | Snapshot-time folder | After-snapshot folder |
|---|---|---|
| Original `D:` | `Before-Snapshot` visible | `After-Snapshot` visible |
| Restored `E:` | `Before-Snapshot` visible | Not visible |

**Evidence:** [Before vs after comparison](../evidence/04-before-vs-after-recovery.png)

This is a visual, point-in-time **folder-level recovery check**, not a byte-level full-data validation.

## Stage 7 — Optional stronger validation

For disposable files that exist in both locations, add SHA-256 checksums and compare their values. Do not claim this was carried out in the original experiment unless you have execution evidence.

```powershell
# EXAMPLE: choose a file that really exists on both drives
Get-FileHash 'D:\Before-Snapshot\example.txt' -Algorithm SHA256
Get-FileHash 'E:\Before-Snapshot\example.txt' -Algorithm SHA256
```

## Stage 8 — Clean up responsibly

Check AWS EC2 volumes, snapshots and instances. Remove only disposable resources that are no longer required, after confirming they are not needed for recovery. EC2 stopping and EBS/snapshot billing are separate concerns.

[Validation](VALIDATION.md) · [Troubleshooting](TROUBLESHOOTING.md) · [Project home](../README.md)
