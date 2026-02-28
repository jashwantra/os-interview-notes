# Chapter 8: Storage & Disk Management

## 8.1 Hard Disk Drive (HDD) Structure

Understanding the physical structure of a hard disk helps explain why disk scheduling algorithms exist and why certain access patterns are faster than others.

A hard disk consists of several components. **Platters** are circular disks coated with magnetic material where data is stored. Most drives have multiple platters stacked on a **spindle**, which is a motor that rotates the platters at high speed — typically 5400, 7200, 10000, or 15000 RPM. **Read/write heads** float just above the platter surface (about 3 nanometers) and read or write data by detecting or changing the magnetic orientation. The **actuator arm** moves the heads across the platter surface to access different tracks.

The disk geometry consists of **tracks**, which are concentric circles on each platter surface. **Sectors** are the smallest addressable units, traditionally 512 bytes but now often 4KB (Advanced Format). A **cylinder** is the set of tracks at the same position on all platters — all tracks that can be accessed without moving the heads.

Disk access time has three components. **Seek time** is the time to move the heads to the correct track, typically 5-15 milliseconds for a random seek. **Rotational latency** is the time waiting for the desired sector to rotate under the head, averaging half a rotation — about 4 milliseconds for a 7200 RPM drive. **Transfer time** is the time to actually read or write the data, typically less than 1 millisecond for a few sectors.

The total access time formula is: Access Time = Seek Time + Rotational Latency + Transfer Time. Seek time dominates, which is why disk scheduling algorithms focus on minimizing head movement.

## 8.2 Disk Scheduling Algorithms

When multiple I/O requests are pending, the disk scheduler decides which to service next. The goal is to minimize total head movement and provide fair service.

**FCFS (First-Come, First-Served)** services requests in the order they arrive. It's simple and fair — no request waits forever. However, it can result in large total head movement if requests are scattered across the disk. Imagine the head bouncing back and forth across the disk.

**SSTF (Shortest Seek Time First)** always services the request closest to the current head position. This minimizes seek time for each individual request and provides good throughput. However, it can cause **starvation** — requests at the edges of the disk may wait indefinitely while the head services a continuous stream of requests in the middle.

**SCAN (Elevator Algorithm)** moves the head in one direction, servicing all requests in that direction, then reverses and services requests in the other direction. It's like an elevator — it goes up, servicing all floors, then goes down. SCAN provides bounded waiting time and no starvation. The downside is that it goes all the way to the end of the disk even if there are no requests there.

**C-SCAN (Circular SCAN)** only services requests in one direction. When it reaches the end, it jumps back to the beginning without servicing requests on the return trip. This provides more uniform wait times — requests at the beginning of the disk don't have to wait for the head to come back from the end.

**LOOK and C-LOOK** are practical improvements on SCAN and C-SCAN. Instead of going all the way to the disk ends, they only go as far as the last request in each direction. This avoids unnecessary travel and is what most real systems use.

To compare these algorithms, consider a disk with 200 cylinders (0-199) and a head starting at cylinder 53, with pending requests at cylinders 98, 183, 37, 122, 14, 124, 65, 67. FCFS would service them in order, resulting in large total head movement. SSTF would service 65, 67, 37, 14, 98, 122, 124, 183 — minimizing each individual seek but potentially starving distant requests. LOOK would go 65, 67, 98, 122, 124, 183, then reverse to 37, 14 — systematic and fair.

## 8.3 RAID (Redundant Array of Independent Disks)

**RAID** combines multiple physical disks into a single logical unit to improve performance, reliability, or both. Different RAID levels offer different trade-offs.

**RAID 0 (Striping)** splits data across multiple disks without redundancy. If you have two disks, half of each file goes on each disk. This doubles read and write performance because both disks work in parallel. You get 100% of the combined capacity. However, there's no fault tolerance — if either disk fails, all data is lost. RAID 0 is actually less reliable than a single disk because you have two points of failure. Use it only for non-critical data where performance matters, like scratch space or game installations.

**RAID 1 (Mirroring)** duplicates all data on two disks. Every write goes to both disks. Read performance doubles because you can read from either disk. Write performance is the same as a single disk. You get 50% of the combined capacity. If one disk fails, the other has a complete copy. RAID 1 is simple and provides good redundancy, but it's expensive in terms of capacity. Use it for critical data like boot drives or databases where you can't afford data loss.

**RAID 5 (Striping with Distributed Parity)** stripes data across three or more disks with parity information distributed across all disks. Parity allows reconstruction of data if one disk fails. You lose one disk's worth of capacity to parity — with N disks, you get (N-1)/N capacity. Read performance is good because data is striped. Write performance is moderate because every write requires reading old data, computing parity, and writing both data and parity. RAID 5 can tolerate one disk failure. It's a good balance of performance, capacity, and redundancy for general-purpose storage.

**RAID 6 (Double Parity)** is like RAID 5 but with two parity blocks per stripe, allowing two simultaneous disk failures. You lose two disks' worth of capacity. Write performance is worse than RAID 5 due to double parity calculation. Use RAID 6 for large arrays where the probability of a second disk failing during rebuild is significant.

**RAID 10 (1+0, Mirrored Stripes)** combines mirroring and striping. Data is first mirrored (RAID 1), then the mirrors are striped (RAID 0). You need at least 4 disks. You get 50% capacity. Performance is excellent — reads can come from any mirror, writes go to both mirrors in parallel. It can tolerate one disk failure per mirror pair. RAID 10 is preferred for databases and high-performance applications where both speed and reliability matter.

## 8.4 SSD vs HDD

**Solid State Drives (SSDs)** use NAND flash memory with no moving parts. This fundamentally changes the performance characteristics compared to HDDs.

HDDs have mechanical components — spinning platters and moving heads. Seek time (5-15ms) and rotational latency (~4ms) dominate access time. Sequential access is much faster than random access because sequential reads don't require seeks. Typical sequential throughput is 100-200 MB/s. Random access is about 100 IOPS (I/O operations per second). HDDs are cheap per gigabyte and have essentially unlimited write endurance.

SSDs have no moving parts, so there's no seek time or rotational latency. Access time is about 0.1ms for SATA SSDs and 0.02ms for NVMe SSDs. Random access is nearly as fast as sequential access. SATA SSDs achieve about 550 MB/s sequential and 90,000 IOPS random. NVMe SSDs achieve 3500+ MB/s sequential and 500,000+ IOPS random. SSDs are more expensive per gigabyte and have limited write endurance due to flash cell wear.

The implications for system design are significant. For HDDs, disk scheduling algorithms matter a lot because minimizing seeks dramatically improves performance. Defragmentation helps by making files contiguous. Sequential access patterns should be preferred.

For SSDs, disk scheduling is less important because there's no seek penalty. Defragmentation is actually harmful — it causes unnecessary writes that wear out the flash cells without improving performance. Random access is nearly as fast as sequential, so data layout matters less.

**TRIM** is an important SSD concept. When you delete a file, the file system marks the blocks as free but doesn't tell the SSD. The SSD still thinks those blocks contain valid data. TRIM is a command that tells the SSD which blocks are no longer in use, allowing the SSD to erase them in the background. This improves write performance and extends SSD life.

**Wear leveling** is how SSDs distribute writes evenly across all flash cells. Each cell can only be written a limited number of times (typically thousands to hundreds of thousands). Without wear leveling, frequently written areas would wear out quickly. The SSD controller tracks write counts and moves data around to even out the wear.

## 8.5 Disk Management

**Formatting** prepares a disk for use. Low-level formatting creates the physical sector structure and is done at the factory. High-level formatting creates the file system structures — the superblock, inode tables, free space bitmaps, and root directory.

**Partitioning** divides a disk into separate logical sections. **MBR (Master Boot Record)** is the legacy partitioning scheme, supporting up to 4 primary partitions and a maximum disk size of 2TB. **GPT (GUID Partition Table)** is the modern scheme, supporting 128+ partitions and disk sizes up to 9.4 ZB (zettabytes). GPT also stores backup partition tables for reliability.

**Bad block management** handles defective sectors. **ECC (Error Correcting Code)** detects and corrects minor errors. When a sector becomes unreliable, the disk remaps it to a spare sector transparently. **SMART (Self-Monitoring, Analysis and Reporting Technology)** monitors disk health and can predict failures before they happen.

## 8.6 Common Interview Questions

**Q: Why does disk scheduling matter for HDDs but not SSDs?**

HDD access time is dominated by seek time — physically moving the heads. Minimizing head movement dramatically improves throughput. SSDs have no moving parts, so there's no seek penalty. Random access is nearly as fast as sequential, making scheduling less important.

**Q: Explain the difference between RAID 5 and RAID 10.**

RAID 5 uses striping with distributed parity — data and parity spread across all disks. It can tolerate one disk failure and gives (N-1)/N capacity. RAID 10 uses mirrored stripes — data is mirrored first, then striped. It can tolerate one failure per mirror pair and gives 50% capacity. RAID 10 has better write performance and faster rebuilds, making it preferred for databases.

**Q: Why shouldn't you defragment an SSD?**

Defragmentation rearranges files to be contiguous, which helps HDDs by reducing seeks. SSDs have no seek penalty, so contiguity doesn't help performance. Defragmentation causes many unnecessary writes, which wear out the flash cells. It's all cost and no benefit for SSDs.

**Q: What is TRIM and why is it important?**

TRIM tells the SSD which blocks are no longer in use. Without TRIM, the SSD doesn't know when files are deleted — it still thinks those blocks contain valid data. This forces the SSD to do read-modify-write operations when reusing those blocks. With TRIM, the SSD can erase unused blocks in the background, improving write performance and extending lifespan.

## Key Takeaways

Understanding storage is essential for system design and performance optimization. HDD access time is dominated by seek time, which is why disk scheduling algorithms exist. LOOK and C-LOOK are the most practical algorithms, avoiding unnecessary travel while preventing starvation.

RAID provides redundancy and/or performance by combining multiple disks. RAID 0 is for speed without redundancy. RAID 1 is simple mirroring. RAID 5 balances capacity, performance, and redundancy. RAID 10 is best for databases requiring both speed and reliability.

SSDs fundamentally change the performance model. Without seek time, random access is nearly as fast as sequential. Disk scheduling becomes less important. Defragmentation is harmful. TRIM support is essential for maintaining performance and longevity.

When designing systems, consider the storage characteristics. For HDDs, prefer sequential access and batch operations. For SSDs, random access is fine, but be mindful of write amplification and wear. Choose the appropriate RAID level based on your reliability and performance requirements.
