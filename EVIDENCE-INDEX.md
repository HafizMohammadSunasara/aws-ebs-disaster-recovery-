# Evidence index — Project 2: EBS Disaster Recovery

These are the **four original project screenshots** provided during the chat, saved as single **2× enlarged, lightly sharpened, privacy-masked PNGs**. We do not duplicate `raw/` and `enhanced/` folders.

| Filename | Screen | Observed proof | Limit |
|---|---|---|---|
| [`01-windows-ec2-and-attached-volumes.png`](../evidence/01-windows-ec2-and-attached-volumes.png) | EC2 `SRV-2A` in `eu-west-2`, running, attachments visible | Instance and three AWS disk attachments | Does not prove filesystem integrity |
| [`02-original-and-restored-ebs-volumes.png`](../evidence/02-original-and-restored-ebs-volumes.png) | EBS volumes list: 2 GiB original, 2 GiB restored, 30 GiB OS | Separate restored volume exists | Does not prove mount/restore data by itself |
| [`03-snapshot-completed.png`](../evidence/03-snapshot-completed.png) | EBS `D-Drive-Snapshot` at 100%/Completed | Snapshot completed | Does not prove application-consistent capture |
| [`04-before-vs-after-recovery.png`](../evidence/04-before-vs-after-recovery.png) | Windows D: (Before + After) versus E: (Before only) | Folder-level point-in-time restore behavior | Does not prove checksums, database recovery or zero post-snapshot loss |

## Image processing policy

- Upscaling is performed with Lanczos resampling to improve presentation size.
- Light contrast and sharpening are applied to make existing text slightly easier to read.
- AWS account-number UI text is masked; resource status, timestamps, drives and folder names remain unchanged.
- Enlarging images **does not recover details that were absent in the originals**.
- Arrow overlays from the original slideshow/screenshots remain as supplied; this is genuine screenshot evidence, not regenerated console output.

## Chronology note

Screenshots may be arranged in explanatory order, not necessarily strict capture-time order. The lab narrative comes from the previously published project description and visible states. No log transcript was supplied.

[Return to README](../README.md)
