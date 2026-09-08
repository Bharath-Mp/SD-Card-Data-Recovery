# SD Card Data Recovery — Recovering ~92% of Data from a Failing 32 GB SD Card

> A real-world data recovery case study involving a severely failing SD card, repeated USB disconnects, damaged filesystem/partition metadata, and successful recovery of approximately **16 GB from an estimated 17.5 GB of stored data**.

---

## 📌 Case Summary

What initially looked like a straightforward SD card recovery quickly turned into a race against failing hardware.

The SD card repeatedly disconnected during acquisition. At one point, the operating system could no longer reliably detect it.

The recovery strategy therefore changed:

> **Don't keep fighting the failing card. Preserve whatever can still be read, create a working image, and perform the rest of the investigation against the image.**

The final workflow looked like this:

```text
Failing SD Card
      │
      ▼
 GNU ddrescue
      │
      ▼
 Master Disk Image
      │
      ├── Sector Inspection
      │
      ├── TestDisk
      │      └── Partition reconstruction unsuccessful
      │
      ▼
     DMDE
      │
      ├── Full Scan
      ├── Filesystem reconstruction
      ├── Folder/file identification
      └── Bulk recovery
      │
      ▼
 Recovered Dataset
      │
      ▼
 Validation & Quantification
      │
      ▼
 ~92% of estimated stored data recovered
```

---

# 🔐 Client & Confidentiality Disclaimer

This recovery was performed for a **high-profile client** under circumstances where confidentiality and data security were critical.

To protect the client's identity and private information:

- No client-identifying information is published.
- Recovered personal data is not published.
- Sensitive filenames have not been disclosed.
- Screenshots should be sanitized before publication.
- The client cannot provide a public testimonial due to security and confidentiality requirements.

The purpose of this document is to demonstrate the **technical recovery methodology and problem-solving process**, not the client's identity or recovered content.

---

# 1. The Problem

The task:

> Recover as much usable data as possible from a failing 32 GB SD card.

The situation became progressively worse.

The SD card:

- Became unstable.
- Repeatedly disconnected from USB.
- Was intermittently detected by the operating system.
- Eventually became unavailable.
- Could not be treated as a reliable filesystem anymore.

At this point, repeatedly mounting or scanning the original card would only increase the risk of losing access to it completely.

The first priority became:

> **Acquire the media before it dies.**

---

# 2. The Recovery Environment

## Hardware

- 32 GB Samsung SD card
- USB SD card reader
- Host Windows machine
- Kali Linux running inside VirtualBox

## Software

- GNU ddrescue
- TestDisk 7.2
- DMDE
- `xxd`
- `fdisk`
- `strings`
- `sha256sum`

---

# 3. First Critical Decision — Image the Card

Instead of trying to recover files directly from the failing filesystem, a disk image was created.

The reason is simple:

> A failing storage device should be treated as an acquisition source, not as the workspace.

The objective was to read as much of the card as possible while recording which areas could not be read.

GNU ddrescue was chosen because it is specifically designed for recovering data from failing media and maintains a mapfile so the recovery can be resumed.

Example:

```bash
sudo ddrescue -n -d /dev/sdb \
/media/sf_Recovery/SD-Image1.img \
/media/sf_Recovery/SD-Image1.log
```

The important artifacts were:

```text
SD-Image1.img
SD-Image1.log
```

The `.img` file became the primary working copy.

The `.log`/mapfile preserved the recovery state.

---

# 4. Blocker #1 — The SD Card Kept Disconnecting

This was one of the biggest challenges.

The SD card would work for a while and then disappear.

At one point the device was no longer reliably available to Linux.

The situation became increasingly unstable as the imaging progressed.

Eventually, ddrescue reported:

```text
Can't open input file: No medium found
```

The card was effectively unavailable.

Fortunately, the previous ddrescue progress had already been written to the image.

This is one of the most important lessons from the entire recovery:

> **A resumable acquisition is far safer than repeatedly starting from scratch.**

---

# 5. The Final ddrescue Situation

Before the SD card became completely unavailable, ddrescue reached approximately:

```text
rescued:      29146 MB
pct rescued:  91.05%
bad areas:    1
read errors:  7
bad-sector:   512 B
non-tried:    2856 MB
```

The card was failing hard and progress had effectively stopped.

At this point, continuing to fight the original hardware was no longer justified.

The master image became the recovery target.

---

# 6. The Image Wasn't Looking Healthy

The next step was to inspect the image without touching the original SD card.

The image was approximately 30 GB:

```bash
ls -lh /media/sf_Recovery/SD-Image1.img
```

Output:

```text
-rwxrwx--- 1 root vboxsf 30G Aug 1 23:08 SD-Image1.img
```

Basic identification:

```bash
sudo file /media/sf_Recovery/SD-Image1.img
```

The result was not a reassuring filesystem identification:

```text
ISO-8859 text, with very long lines (65536), with no line terminators
```

That obviously wasn't enough to tell us what the original filesystem actually was.

So the investigation moved down to the sector level.

---

# 7. Sector Inspection

The beginning of the image was inspected with `xxd`:

```bash
sudo xxd -g1 -l 512 /media/sf_Recovery/SD-Image1.img
```

The beginning was dominated by:

```text
ff ff ff ff ff ff ff ff ...
```

Additional sectors showed the same pattern.

For example:

```bash
sudo dd if=/media/sf_Recovery/SD-Image1.img \
bs=512 skip=6 count=1 | xxd -g1
```

Again, the sector contained predominantly:

```text
ff ff ff ff ff ff ff ff ...
```

This meant the beginning of the image did not present the expected clean partition/filesystem metadata.

But there was an important clue.

---

# 8. Searching the Image for Filesystem Evidence

A strings search was performed:

```bash
strings -a /media/sf_Recovery/SD-Image1.img | grep FAT
```

This produced a large number of `FAT` occurrences.

However, this result had to be interpreted carefully.

Finding the text `FAT` inside a raw image does **not** automatically prove that the filesystem metadata is intact.

It simply told us that there was evidence of FAT-related data somewhere inside the image.

This was enough to justify deeper filesystem analysis.

---

# 9. TestDisk — First Recovery Attempt

The next step was TestDisk 7.2.

The image was opened directly:

```text
Disk /media/sf_Recovery/SD-Image1.img
```

TestDisk was then allowed to analyze the partition structure.

The partition table was not clean.

TestDisk reported:

```text
Partition sector doesn't have the endmark 0xAA55
```

This was a significant indication that the expected partition metadata was damaged or missing.

---

# 10. TestDisk Deeper Search

A deeper search was performed.

TestDisk proposed a FAT32 LBA partition:

```text
FAT32 LBA
Start: 115503 234 11
End:   229056 103 52
Size:  1824220734 sectors
```

But there was an obvious problem.

The image itself was only approximately:

```text
31 GB / 29 GiB
```

while the proposed partition was approximately:

```text
1.8 billion sectors
```

TestDisk therefore reported:

```text
The hard disk (31 GB / 29 GiB) seems too small!
(< 1884 GB / 1754 GiB)
```

The deeper search produced the same fundamental issue.

The proposed partition simply could not physically fit inside the image.

---

# 11. Was the Data Gone?

This was the point where the recovery could easily have been considered a failure.

But there was an important distinction:

> **A damaged partition table is not the same thing as destroyed file data.**

The next question became:

> Can we find surviving filesystem structures somewhere inside the image?

That led to DMDE.

---

# 12. DMDE — The Turning Point

DMDE was used to perform a **Full Scan** of the image.

RAW results were kept enabled during the scan.

The scan produced something much more interesting than the broken partition structure shown by TestDisk.

DMDE identified a reconstructed volume containing:

- Recognizable folders
- Original-looking filenames
- File extensions
- File sizes
- Directory hierarchy

This was the first strong indication that the underlying file data was still recoverable.

---

## Screenshot

![DMDE Full Scan](screenshots/03-dmde-full-scan.png)

*Sanitized screenshot showing the reconstructed filesystem/scan results.*

---

# 13. The First Recovery Test

Rather than immediately launching a large recovery operation, a single JPEG file was selected.

The file was recovered.

Then came the moment that changed the recovery:

> **The JPEG opened perfectly.**

This was no longer theoretical recoverability.

There was working file data.

The reconstructed volume was usable.

---

# 14. Why DMDE Became the Primary Recovery Tool

At this stage, the objective changed.

We were no longer asking:

> "Can anything be recovered?"

We were asking:

> "How much can we recover?"

The directory structure visible in DMDE was especially valuable.

Instead of receiving thousands of generic carved filenames, the reconstructed filesystem exposed meaningful folders and filenames.

The goal was therefore to preserve that structure during bulk recovery.

---

# 15. DMDE License

The free/limited workflow was not sufficient for the recovery approach I wanted.

I therefore purchased a **one-month DMDE license** specifically for this recovery.

This allowed the bulk recovery to be performed while retaining the available filesystem structure rather than manually rebuilding folders one by one.

The entire reconstructed volume was then selected for recovery.

---

# 16. Blocker #2 — Individual Files Were Damaged

During bulk recovery, DMDE encountered files that contained unreadable regions.

Instead of stopping the entire recovery process for individual problematic files, the affected files were separated into a `$Bad` location.

This gave us two advantages:

1. The main recovery could continue.
2. The damaged files remained available for later targeted analysis.

This is an important recovery principle:

> **Don't allow a small number of damaged files to compromise the recovery of the files that are still healthy.**

---

# 17. Maintaining Evidence Separation

The recovery data was kept separate from the original image.

Conceptually:

```text
Recovery Workspace
│
├── Master Image
│   └── SD-Image1.img
│
├── Acquisition Metadata
│   └── SD-Image1.log
│
├── Recovered Data
│   └── ~16 GB
│
└── Problematic Files
    └── $Bad
```

This separation made it easier to:

- Preserve the master image.
- Validate recovered files.
- Re-run analysis if necessary.
- Investigate problematic files independently.

---

# 18. Hashing

The master image was hashed using SHA-256:

```bash
sha256sum /media/sf_Recovery/SD-Image1.img
```

Hashing the image provides an integrity reference for the acquisition artifact.

The recovered data was kept separate from this master image.

---

# 19. Quantifying the Recovery

This was an important part of the project.

A 32 GB SD card does **not** mean that 32 GB of user files existed on it.

There is a difference between:

```text
Manufacturer capacity
        ↓
Usable filesystem capacity
        ↓
Actually stored user data
        ↓
Successfully recoverable data
```

The image/volume was approximately:

```text
29.7 GiB
```

But the reconstructed filesystem indicated approximately:

```text
17.5 GB
```

of actual stored data.

After recovery and cleanup:

```text
Recovered usable data ≈ 16 GB
```

Therefore:

```text
Recovery Rate = Recovered Data / Estimated Original Data × 100

               = 16 / 17.5 × 100

               ≈ 91.43%
```

Rounded for reporting:

# **≈92% Recovery**

---

# 20. Final Numbers

| Metric | Result |
|---|---:|
| SD card capacity | 32 GB |
| Approx. image/usable capacity | 29.7 GiB |
| Estimated original stored data | ~17.5 GB |
| Usable recovered data | ~16 GB |
| Estimated unrecovered data | ~1.5 GB |
| Final reported recovery rate | **≈92%** |
| ddrescue rescued | **≈91.05%** |

### Important distinction

The two percentages represent different things.

**91.05%** was the ddrescue acquisition progress against the failing physical source.

**~92%** is the estimated percentage of the actual stored user data that was ultimately recovered into usable files.

They should not be treated as the same metric.

---

# 21. What Went Wrong?

Several things went wrong during the recovery:

- The SD card was physically failing.
- The card repeatedly disconnected.
- The operating system eventually stopped detecting it reliably.
- The initial filesystem/partition metadata was damaged.
- TestDisk could not reconstruct a physically valid partition.
- Some individual files contained unreadable regions.
- The original media eventually became unavailable before a perfect acquisition could be completed.

---

# 22. What Went Right?

A number of decisions made the recovery possible:

### 1. Imaging before aggressive recovery

The original card was not used as the main workspace.

### 2. Using ddrescue

The mapfile allowed the acquisition to be resumed rather than restarted.

### 3. Switching from filesystem recovery to image analysis

Once the card became unavailable, the master image became the primary evidence source.

### 4. Not giving up after TestDisk failed

TestDisk's inability to reconstruct the partition did not prove that the file data was gone.

### 5. Using DMDE

DMDE exposed a reconstructed volume with meaningful directory and file information.

### 6. Testing before bulk recovery

A single JPEG was recovered first and opened successfully.

### 7. Separating bad files

Problematic files were moved to `$Bad` instead of allowing them to interfere with the broader recovery.

---

# 23. Recovery Decision Tree

The final workflow can be summarized as:

```text
                 Failing SD Card
                       │
                       ▼
                Is it detectable?
                       │
                       ▼
                  GNU ddrescue
                       │
                       ▼
                  Master Image
                       │
                       ▼
             Sector / Image Analysis
                       │
                       ▼
                 TestDisk Search
                       │
             ┌─────────┴─────────┐
             │                   │
       Valid structure?       Damaged structure
             │                   │
             ▼                   ▼
       Filesystem tools        DMDE
                                 │
                                 ▼
                           Full Scan
                                 │
                                 ▼
                       Reconstructed Volume
                                 │
                                 ▼
                         Test File Recovery
                                 │
                                 ▼
                         Bulk Recovery
                                 │
                    ┌────────────┴────────────┐
                    │                         │
              Healthy files              Bad files
                    │                         │
                    ▼                         ▼
             Recovered Dataset              $Bad
                    │                         │
                    └────────────┬────────────┘
                                 ▼
                           Validation
                                 │
                                 ▼
                         Recovery Quantification
```

---

# 24. Key Linux Commands

## Identify the storage device

```bash
lsblk -o NAME,SIZE,MODEL,TRAN
```

---

## Acquire the failing device

```bash
sudo ddrescue -n -d /dev/sdb \
/media/sf_Recovery/SD-Image1.img \
/media/sf_Recovery/SD-Image1.log
```

---

## Inspect the image header

```bash
sudo xxd -g1 -l 512 \
/media/sf_Recovery/SD-Image1.img
```

---

## Inspect the partition structure

```bash
fdisk -l \
/media/sf_Recovery/SD-Image1.img
```

---

## Search for FAT-related strings

```bash
strings -a \
/media/sf_Recovery/SD-Image1.img | grep FAT
```

---

## Inspect a specific sector

```bash
sudo dd \
if=/media/sf_Recovery/SD-Image1.img \
bs=512 \
skip=6 \
count=1 | xxd -g1
```

---

## Hash the master image

```bash
sha256sum \
/media/sf_Recovery/SD-Image1.img
```

---

# 25. Tools Used

| Tool | Purpose |
|---|---|
| **GNU ddrescue** | Acquisition from failing media |
| **TestDisk 7.2** | Partition/filesystem analysis |
| **DMDE** | Full scan, filesystem reconstruction and bulk recovery |
| **xxd** | Sector-level inspection |
| **fdisk** | Partition structure inspection |
| **strings** | Searching raw image contents |
| **sha256sum** | Image integrity verification |

---

# 26. The Most Important Lesson

The most important lesson from this recovery was not a particular command.

It was the sequence of decisions.

> **Acquire first. Analyze second. Recover third. Validate last.**

A failing storage device should not be treated like a normal filesystem.

And a broken partition table should not immediately be interpreted as:

> "The data is gone."

In this case:

```text
TestDisk: Unable to reconstruct a valid partition
                 ↓
          Does NOT mean
                 ↓
        Files are destroyed
                 ↓
        DMDE Full Scan
                 ↓
      Filesystem structures found
                 ↓
       Test JPEG opens correctly
                 ↓
       Bulk recovery succeeds
```

The filesystem structure was damaged.

The data was still there.

---

# 27. Future Digital Media Recovery Toolkit

This project also reinforced the value of maintaining a dedicated recovery toolkit.

My planned toolkit includes:

- GNU ddrescue
- TestDisk
- PhotoRec
- DMDE
- FTK Imager
- Autopsy
- Sleuth Kit
- dcfldd
- ewf-tools
- Hex editor
- ExifTool

The objective is not to have ten tools that do the same thing.

The objective is to have different tools available for different failure modes.

---

# 28. What I Would Do Differently Next Time

A few improvements would make future recoveries even cleaner:

### Acquire as early as possible

If a device is showing signs of physical instability, imaging should become the priority immediately.

### Keep the master image immutable

Once acquisition is complete, all further investigation should happen against a copy/image rather than the original source.

### Capture evidence systematically

Record:

- Device identification
- ddrescue statistics
- Mapfile
- Image hash
- Partition analysis
- Recovery statistics
- Problematic file list

### Separate acquisition, analysis and recovery

Keeping these stages separate makes the investigation easier to reproduce and document.

---

# 29. Final Outcome

After dealing with:

- A physically failing SD card
- Repeated USB disconnects
- Incomplete acquisition
- Damaged partition metadata
- TestDisk reconstruction failure
- Individual unreadable files

the recovery ultimately produced approximately:

> **16 GB of usable recovered data from an estimated 17.5 GB of stored data.**

That gives an estimated:

# **≈92% Recovery Rate**

The most satisfying part wasn't the percentage.

It was seeing the first recovered JPEG open perfectly after the filesystem appeared to be badly damaged.

That single file changed the question from:

> "Is recovery possible?"

to:

> **"How much can we recover?"**

---

# 📚 Project Takeaway

This recovery reinforced a principle that applies far beyond SD cards:

> **When storage media starts failing, the first objective isn't to recover files. It's to preserve the evidence and maximize the amount of readable data before the source disappears.**

Once the data was safely represented in an image, the investigation became a puzzle rather than a race against failing hardware.

---

## 🔒 Client Confidentiality

No client-identifying information, private files, personal information or sensitive recovered content is included in this publication.

The absence of a client testimonial is intentional and is a consequence of the security and confidentiality requirements surrounding the engagement.

---

## 🤝 Recovery Assistance

I currently have an **active one-month DMDE license** following this recovery project.

If you have a failing SD card, USB drive, HDD, SSD or similar storage device and need assistance with data recovery, I am open to discussing suitable cases.

**Important:** Recovery success depends heavily on the condition of the media. No responsible recovery practitioner can guarantee recovery before examining the source.

---

## ⚠️ Important Recovery Advice

If your storage device is failing:

**Do not keep repeatedly plugging it in, formatting it, running CHKDSK/fsck, or attempting random recovery operations.**

Every additional operation can potentially make a difficult recovery harder.

If the data is important, prioritize:

```text
Stop unnecessary writes
        ↓
Preserve the source
        ↓
Create an image
        ↓
Work from the image
```

---

## 📎 Repository Disclaimer

This repository documents a real recovery workflow for educational and professional purposes.

Actual client data, full disk images, private filenames, identifying information and sensitive screenshots are intentionally excluded.
