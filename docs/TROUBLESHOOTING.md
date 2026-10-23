# Troubleshooting — AWS-to-Windows recovery boundary

**Scenario:** A restored EBS volume is visible in EC2 but not usable as a Windows drive; the learner also reported a disk-signature/offline-disk problem during the lab.

> **Evidence disclaimer:** No screenshot of the Windows error or exact repair command was supplied. This is a reasoned recovery runbook plus a clearly attributed report, not a fabricated incident transcript.

## Diagnostic sequence

### 1. Volume not listed in AWS

**Check:** EC2 → Volumes, current Region and filters.  
**Possible cause:** Wrong Region, creation still in progress, permission issue.  
**Next action:** Locate correct volume and verify `available` or `in-use`.

### 2. Volume attached to wrong Availability Zone or instance

**Check:** Volume AZ, EC2 instance AZ, attachment details.  
**Rule:** An EBS volume must be attached to an EC2 instance in the same AZ.  
**Next action:** Re-create volume from snapshot in desired target AZ if necessary.

### 3. AWS shows In-use but no E: in Windows

**Check:** Disk Management → disks, offline status, partitions and volumes.  
**Possible cause:** Windows has not brought the disk online or assigned a letter.  
**Next action:** Identify the disk non-destructively; confirm expected size, identifiers and filesystem before any action.

### 4. Disk offline due to signature collision

A restored disk can resemble its source closely enough to confuse OS-level disk identification. Windows may protect against collisions by keeping one disk offline. The exact recovery steps depend on MBR/GPT disk identity and Windows storage configuration.

**Safe process:**

- Distinguish original `D:` from restored disk using AWS attachment metadata and Windows disk properties.
- Record disk number, capacity, layout and status.
- Confirm Windows' actual reason for the offline state.
- Choose the least destructive remedy appropriate to your disposable lab setup.
- Never run `clean`, initialize or format a data-bearing restored volume as a generic fix.

Avoid publishing claims like “switching to Basic Disk always fixes signature collisions”; there are multiple different root causes.

### 5. Drive mounted but content missing

**Checks:** Correct snapshot selected? Correct volume mounted? Correct directory? Was content written before or after snapshot? Was data buffered but not flushed?

If `After-Snapshot` is absent on `E:`, that is **expected** in this project. Do not repair a correct restore as if it were a fault.

### 6. Restore is slower than expected

Volumes created from snapshots can experience first-read latency while blocks initialize. Evaluate initialization options and use measured timings before deciding on an RTO claim.

## Root cause vs evidence: a lesson

The Windows disk-identity problem is a good example of **cross-layer diagnosis**. AWS-side attachment state and Windows-side mount state are not interchangeable.

What was shown in provided screenshots: restored volume and accessible files.  
What was additionally reported: disk signature/offline disk troubleshooting.  
What is not available: log lines, exact error code, remediation commands or before/after screenshots of the Windows error.

[Home](../README.md) · [Implementation](IMPLEMENTATION.md)
