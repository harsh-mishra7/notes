# Object Storage

## TL;DR

**Object storage is a way to store huge amounts of files ("objects") as simple key → blob pairs, accessed over HTTP.**

You don't get folders on a disk or a mountable drive. You get a flat, practically unlimited bucket where you `PUT` a file under a name and `GET` it back by that name. It's cheap, extremely durable, and scales to billions of files.

Examples: **Amazon S3**, Google Cloud Storage, Azure Blob Storage, Cloudflare R2, MinIO (self-hosted).

In system design, it's the default answer to **"where do the images, videos, and files go?"**

---

## The analogy that makes it click

**Object storage is a valet coat check.**

You hand over your coat (the file), and you get a ticket number (the key). You don't know or care which rack it's on. When you want it back, you show the ticket and get your coat. You can also pin a note to it — "blue, wool, belongs to Priya" (metadata).

- You can't alter a sleeve while it's hanging there. To change it, you hand in a whole new coat. (objects are replaced whole, not edited in place)
- The coat check can hold millions of coats, and it keeps copies in several buildings in case one burns down. (durability)

Compare that to **block storage** (your own closet, where you arrange everything shelf by shelf) and **file storage** (a shared office closet with labeled folders everyone navigates together).

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Bucket** | A top-level container for objects (like `myapp-user-uploads`) |
| **Object** | The file itself (the bytes) plus its metadata |
| **Key** | The object's unique name inside the bucket, e.g. `users/42/avatar.jpg` |
| **Metadata** | Key-value info attached to the object: content type, size, custom tags |
| **Pre-signed URL** | A temporary URL that grants access to one object without sharing credentials |

Note: `users/42/avatar.jpg` *looks* like folders, but it's really just one long key string in a flat namespace. The `/` is a naming convention that tools display as folders.

---

## Block vs file vs object storage

```
BLOCK STORAGE                FILE STORAGE                  OBJECT STORAGE

[blk][blk][blk][blk]         /                             bucket: photos
[blk][blk][blk][blk]         ├── home/                       "cat.jpg"      → bytes + metadata
 raw chunks, a server        │   └── notes.txt               "trip/beach.png" → bytes + metadata
 formats it with a           └── shared/                     "video/1.mp4"  → bytes + metadata
 filesystem                      └── report.pdf
                                                           flat, accessed via HTTP API
 like a raw hard drive       like a shared network drive
```

| | **Block** | **File** | **Object** |
|---|---|---|---|
| Unit | Fixed-size blocks | Files in folders | Objects (data + metadata) with a key |
| Access | Mounted as a disk by one server | Mounted over the network (NFS, SMB) by many servers | HTTP API (`PUT`, `GET`, `DELETE`) |
| Structure | None — OS adds a filesystem | Hierarchical folders | Flat namespace |
| Edit part of a file? | ✅ Yes, low-level | ✅ Yes | ❌ No — replace the whole object (appends/multipart uploads aside) |
| Latency | Lowest | Low | Higher (tens of ms per request) |
| Scale | One volume, TBs | Moderate | Practically unlimited |
| Cost | Highest | Medium | Lowest |
| Good for | Databases, OS disks, VMs | Shared files between servers, legacy apps | Images, videos, backups, logs, static sites, data lakes |
| Examples | AWS EBS, local SSD | AWS EFS, NFS servers | AWS S3, GCS, Azure Blob |

**Why not store images in the database?** Big binary blobs bloat the DB, make backups slow, eat expensive DB storage and memory, and slow down queries. Databases are built for small structured rows, not 50 MB videos.

---

## Why object storage is used for images, videos, and backups

| Reason | Why it matters |
|---|---|
| **Practically unlimited scale** | Store billions of objects without planning disk capacity |
| **Cheap** | Fractions of a cent per GB per month, plus cheaper "cold" tiers for rarely-read data |
| **Extremely durable** | Data is copied across multiple devices and data centers automatically |
| **Accessible over HTTP** | Browsers and CDNs can fetch files directly — your app servers don't have to stream them |
| **Storage tiers** | Move old data to cheaper classes (e.g. S3 Standard → Infrequent Access → Glacier) with lifecycle rules |
| **Built-in features** | Versioning, lifecycle rules, access policies, encryption |

Trade-offs: higher latency than a local disk, no partial in-place edits, and not usable as a database or a normal filesystem.

---

## Durability: "eleven nines"

Amazon S3 is designed for **99.999999999% (11 nines) durability** of objects over a year.

What that means in plain English: if you store **10 million objects**, you'd expect to lose **one object roughly every 10,000 years** on average.

It achieves this by storing each object redundantly across **multiple devices in at least three Availability Zones** (separate data centers) and constantly checking for and repairing corruption.

| Term | Means | S3 Standard (design target) |
|---|---|---|
| **Durability** | Will my data still exist? (not lost) | 99.999999999% (11 nines) |
| **Availability** | Can I access it right now? | 99.99% |

Don't mix them up: data can be temporarily unreachable (availability) without being lost (durability).

---

## Pre-signed URLs: uploads and downloads without your server in the middle

Problem: users upload 200 MB videos. If every upload streams **through your app servers**, those servers waste bandwidth and CPU just shoveling bytes.

Solution: the app hands the client a **pre-signed URL** — a temporary, signed link that allows one specific action (e.g. `PUT` to one key) for a limited time (e.g. 15 minutes). The client talks to object storage **directly**.

```
1. Client ──► App server:  "I want to upload a video"
2. App server checks auth, generates a pre-signed URL:
      https://bucket.s3.amazonaws.com/videos/abc.mp4?X-Amz-Signature=...&X-Amz-Expires=900
3. App server ──► Client:  here's the URL
4. Client ══ PUT 200 MB ══► S3 directly     (app server not involved ✅)
5. Client ──► App server:  "upload done"  (or S3 sends an event notification)
6. App server saves metadata in the DB
```

Same idea for **downloads** of private files: generate a short-lived `GET` URL so only authorized users can fetch that object, and the bucket itself stays private.

---

## Pairing object storage with a CDN

Object storage is the **origin**; the CDN caches files close to users.

```
                  (miss only)
User ──► CDN edge ──────────────► Object storage bucket (origin)
    ◄── cached file (fast) ✅
```

- Users get files from a nearby edge — much lower latency.
- Fewer requests hit the bucket — lower cost (egress from a CDN is often cheaper).
- The bucket can stay private, readable only by the CDN (e.g. CloudFront with Origin Access Control).

This `CDN → S3` pair is the standard way to serve images, video segments, and static websites.

---

## The core pattern: metadata in the DB, blobs in object storage

Split every file into two parts:

```
                 ┌──────────────────────────────────────┐
                 │ Database  (small, structured, queryable)
                 │ photos table:
                 │  id | user_id | s3_key                | size  | created_at
                 │  91 | 42      | photos/42/91.jpg      | 2.1MB | 2026-09-30
                 └──────────────────────────────────────┘
                                   │ s3_key points to ▼
                 ┌──────────────────────────────────────┐
                 │ Object storage  (big, cheap, durable)
                 │  photos/42/91.jpg  →  [ 2.1 MB of JPEG bytes ]
                 └──────────────────────────────────────┘
```

| Goes in the **database** | Goes in **object storage** |
|---|---|
| Who owns the file, title, tags, likes | The actual image/video/PDF bytes |
| The object's key or URL | Thumbnails and transcoded versions |
| Upload status, size, content type | Backups and log archives |

Now you can query "all photos by user 42 from last week" in the DB fast, and fetch the actual files from object storage/CDN only when needed.

**Tip:** use unique, non-guessable keys (e.g. UUIDs) rather than user-provided filenames, to avoid collisions and people guessing other users' file URLs.

---

## Where this shows up in HLD

- **Instagram, YouTube, Dropbox, WhatsApp media, Netflix** — any design with user-uploaded files uses object storage for the bytes and a DB for metadata.
- **Uploads via pre-signed URLs** is a strong interview point — it shows you know how to keep large payloads off your app servers.
- **Video platforms**: raw upload → object storage → message queue triggers transcoding workers → processed versions back to object storage → served through a CDN.
- **Backups, logs, and data lakes** land in object storage because it's cheap and durable, with lifecycle rules moving old data to cold tiers.
- **Static website hosting**: HTML/CSS/JS in a bucket behind a CDN.

---

## Key takeaways

- Object storage keeps files as **key → object (bytes + metadata)** in flat buckets, accessed over HTTP.
- Block storage is a raw disk (databases), file storage is a shared filesystem, object storage is cheap, massive, and durable for blobs.
- S3 is designed for **11 nines of durability** by replicating across multiple data centers.
- Use **pre-signed URLs** so clients upload/download directly without passing big files through your servers.
- Standard pattern: **metadata in the database, bytes in object storage, served through a CDN.**
