# Validation — what the evidence actually proves

## Verified evidence matrix

| ID | Screenshot | Visible evidence | Supported conclusion |
|---|---|---|---|
| V1 | [EC2 and attachments](../evidence/01-windows-ec2-and-attached-volumes.png) | Windows `SRV-2A` running, system and two data volumes attached | EC2 and data-volume attachment state |
| V2 | [EBS volumes](../evidence/02-original-and-restored-ebs-volumes.png) | Original and restored gp3 2 GiB volumes | Separate source and restored volumes exist |
| V3 | [Snapshot](../evidence/03-snapshot-completed.png) | `D-Drive-Snapshot` status `Completed`, progress `100%` | Snapshot operation succeeded |
| V4 | [Folder comparison](../evidence/04-before-vs-after-recovery.png) | D: `Before-Snapshot` + `After-Snapshot`; E: `Before-Snapshot` only | Restored folder view matches snapshot-time state |

## Claim taxonomy

### Demonstrated

- A regional AWS EBS snapshot completed.
- A separate EBS volume restored from snapshot was present and attached.
- Windows Explorer showed the expected pre/post-snapshot folder separation.

### Reported from user's project notes, without direct screenshot of error

- An offline disk/signature collision issue occurred and was troubleshot during the Windows restore workflow.

### Not demonstrated

- File checksum equality, 100% byte-level restore integrity
- No data loss since time of failure or zero RPO
- Snapshot application consistency via VSS
- Automated backup or recovery
- Tested failover to new instance/AZ
- Measured RTO or RPO against a business requirement
- Full database/application recovery

## Stronger future validation plan

1. Store known small files with SHA-256 manifests before snapshot.
2. Time each restore stage (new volume, attachment, Windows mount, data check).
3. Compare all expected files by relative path and hash.
4. Verify restore into a fresh instance, not only the original instance.
5. Validate application usability for a real workload, not just folder visibility.

**Why this honesty matters:** a recruiter can distinguish a real, narrow test from a sweeping claim. Evidence is most credible when its boundaries are clear.

[Evidence index](EVIDENCE-INDEX.md) · [Home](../README.md)
