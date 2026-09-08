
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
