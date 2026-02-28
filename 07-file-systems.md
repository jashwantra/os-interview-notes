# Chapter 7: File Systems

## 7.1 What is a File System?

A **file system** is the method and data structure that an operating system uses to organize, store, and retrieve data on storage devices. Without a file system, data on a disk would be one large body of bits with no way to tell where one piece of data ends and another begins.

The file system provides several essential abstractions. It provides **naming**, allowing you to access data by human-readable names like "report.pdf" rather than disk addresses like "sector 1234, cylinder 56". It provides **organization** through hierarchical directories that let you group related files. It provides **persistence**, ensuring data survives power cycles and system restarts. It provides **sharing and protection**, controlling who can access what data.

When you save a file in your C++ program, you're interacting with the file system. The file system translates your high-level operations like "write this string to file.txt" into low-level disk operations like "write these bytes to these specific disk blocks".

## 7.2 File Concepts

A **file** is a named collection of related information stored on secondary storage. From the user's perspective, a file is the smallest unit of storage — you can't write to disk without creating a file.

Every file has **attributes** (metadata) that describe it. The **name** is the human-readable identifier. The **identifier** is a unique number the file system uses internally, like an inode number in Unix. The **type** indicates whether it's a regular file, directory, device file, or symbolic link. The **location** contains pointers to where the data is stored on disk. The **size** is the current size in bytes. **Protection** specifies access permissions — who can read, write, or execute. **Timestamps** record when the file was created, last modified, and last accessed. The **owner** identifies who owns the file.

The basic **file operations** that a file system must support include `create` to allocate space and create a directory entry, `open` to load metadata into memory and return a file descriptor, `read` to copy data from the file to a buffer, `write` to copy data from a buffer to the file, `seek` to move the current position within the file, `close` to release resources and flush buffers, and `delete` to remove the directory entry and free the space.

When you call `fopen()` in C or create an `std::fstream` in C++, the OS opens the file, loads its metadata into an in-memory structure, and returns a file descriptor (an integer) that you use for subsequent operations. This avoids repeatedly searching the directory for every read or write.

## 7.3 Directory Structure

A **directory** is a special file that contains a list of files and subdirectories. It maps file names to their metadata (or to inode numbers that point to metadata).

Early systems used a **single-level directory** where all files were in one directory. This caused name collisions — two users couldn't both have a file named "data.txt". It also made organization impossible with thousands of files.

**Two-level directories** gave each user their own directory, solving the name collision problem. But users still couldn't organize their own files into subdirectories.

Modern systems use **tree-structured (hierarchical) directories** where directories can contain subdirectories to arbitrary depth. This is what you're familiar with:

```
/
├── home/
│   └── user/
│       ├── documents/
│       │   └── report.txt
│       └── code/
│           └── main.cpp
├── etc/
└── bin/
```

Paths can be **absolute**, starting from the root like `/home/user/report.txt`, or **relative**, starting from the current directory like `../documents/report.txt`.

**Links** allow multiple names to refer to the same file. A **hard link** is another directory entry pointing to the same inode. The file has a reference count, and the data is only deleted when the last link is removed. Hard links can't cross file system boundaries because inode numbers are only unique within a file system. A **symbolic link** (symlink) is a special file containing the path to another file. It can cross file systems and can point to directories. If the target is deleted, the symlink becomes "dangling" — it points to nothing.

## 7.4 File Allocation Methods

The file system must decide how to allocate disk blocks to files. This is one of the most important design decisions, affecting performance and flexibility.

**Contiguous allocation** stores each file in a contiguous set of blocks. The directory entry contains the starting block and the length. This is simple and provides excellent performance for both sequential and random access — to read block N of a file, you just access starting_block + N. However, it suffers from external fragmentation as files are created and deleted. Files can't easily grow if there's no free space after them. You must know the file size when creating it. This method is used for CD-ROMs and some embedded systems where files don't change.

**Linked allocation** stores each file as a linked list of blocks. Each block contains a pointer to the next block. The directory entry contains the starting block. Files can grow easily — just add more blocks to the chain. There's no external fragmentation. However, random access is terrible — to read block N, you must follow N pointers. Pointer overhead wastes space in each block. If one pointer is corrupted, the rest of the file is lost.

The **File Allocation Table (FAT)** improves on linked allocation by storing all the pointers in a separate table at the beginning of the disk. The FAT can be cached in memory, making traversal much faster. FAT32 is still widely used for USB drives and memory cards because of its simplicity and compatibility.

**Indexed allocation** gives each file an index block containing pointers to all its data blocks. The directory entry points to the index block. This provides fast random access — to read block N, look up the Nth entry in the index. Files can grow easily. However, the index block is overhead, especially for small files. The index block size limits the maximum file size.

The **Unix inode structure** is a clever hybrid that handles both small and large files efficiently. An inode contains 12 direct pointers to data blocks. For small files (up to 48KB with 4KB blocks), this is all you need — fast access with no indirection. For larger files, there's a single indirect pointer pointing to a block of pointers, a double indirect pointer pointing to a block of pointers to blocks of pointers, and a triple indirect pointer for very large files. Most files are small, so they use only direct pointers. Large files pay the cost of indirection, but they're rare.

## 7.5 Free Space Management

The file system must track which blocks are free and which are allocated.

A **bitmap** (or bit vector) uses one bit per block — 1 for allocated, 0 for free. Finding a free block means finding a 0 bit. Finding N contiguous free blocks means finding N consecutive 0 bits. Bitmaps are simple, efficient, and can be cached in memory. For a 1TB disk with 4KB blocks, the bitmap is about 32MB.

A **linked list** chains all free blocks together. The first free block contains a pointer to the second, and so on. This is simple but inefficient for finding contiguous blocks — you'd have to traverse the list.

**Grouping** stores addresses of N free blocks in the first free block. One of those addresses points to another block containing N more addresses, and so on. This is more efficient than a simple linked list.

**Counting** stores (starting_block, count) pairs for contiguous free regions. This is efficient when free space tends to be contiguous.

Most modern file systems use bitmaps because they're simple, efficient, and work well with caching.

## 7.6 I/O Methods

Understanding how the CPU communicates with I/O devices helps understand file system performance.

**Programmed I/O (Polling)** has the CPU actively wait for the device. The CPU repeatedly checks the device status register until the operation completes. This is simple but wastes CPU cycles — the CPU does nothing useful while waiting.

**Interrupt-driven I/O** has the device interrupt the CPU when the operation completes. The CPU starts the I/O operation and then does other work. When the device finishes, it sends an interrupt, and the CPU handles the completion. This is more efficient but still involves the CPU for every byte or word transferred.

**Direct Memory Access (DMA)** has a DMA controller transfer data directly between the device and memory without CPU involvement. The CPU sets up the transfer (source, destination, count) and then does other work. The DMA controller handles the transfer and interrupts the CPU only when done. This is essential for high-speed devices like disks — the CPU would be overwhelmed transferring every byte.

## 7.7 Buffering and Caching

**Buffering** handles the speed mismatch between producers and consumers. When you write to a file, the data goes to a buffer first. The buffer is flushed to disk when full or when you close the file. This allows the application to continue without waiting for slow disk writes.

**Caching** keeps frequently accessed disk blocks in memory. The **buffer cache** (or page cache in Linux) stores recently accessed blocks. When you read a file, the OS checks the cache first. If the block is there (cache hit), no disk access is needed. If not (cache miss), the block is read from disk and added to the cache.

For writes, there are two strategies. **Write-through** writes to both cache and disk immediately — safe but slow. **Write-back** writes only to cache and flushes to disk later — fast but data can be lost on crash. Most systems use write-back with periodic flushing.

Caching is extremely effective because of locality — programs tend to access the same files and blocks repeatedly. The cache hit rate is often 90% or higher, dramatically improving performance.

## 7.8 Journaling

A **journaling file system** maintains a log (journal) of changes before applying them to the main file system. This enables crash recovery.

The process works as follows. Before modifying the file system, write the intended changes to the journal. Then apply the changes to the file system. Finally, mark the journal entry as complete.

If the system crashes during step 2, recovery is straightforward. On reboot, check the journal. If there are incomplete entries, either replay them (redo) or discard them (undo), depending on the journaling mode.

**Metadata journaling** only journals metadata changes (directory entries, inodes, bitmaps), not file data. This is faster but file contents might be inconsistent after a crash. **Full journaling** journals both metadata and data — safer but slower because data is written twice (journal then file system).

Most modern file systems use journaling: ext3/ext4 on Linux, NTFS on Windows, HFS+ and APFS on macOS. Without journaling, a crash could leave the file system in an inconsistent state requiring a lengthy `fsck` (file system check) on reboot.

## 7.9 Common Interview Questions

**Q: What's the difference between hard links and symbolic links?**

A hard link is another directory entry pointing to the same inode — it's essentially another name for the same file. The file's data is only deleted when all hard links are removed. Hard links can't cross file systems. A symbolic link is a special file containing a path to another file. It can cross file systems and can point to directories. If the target is deleted, the symlink becomes dangling.

**Q: Explain the Unix inode structure.**

An inode contains file metadata and pointers to data blocks. It has 12 direct pointers for small files, plus single, double, and triple indirect pointers for larger files. This design optimizes for the common case — most files are small and use only direct pointers. Large files pay the cost of indirection but are rare.

**Q: Why is journaling important?**

Journaling enables crash recovery. Without it, a crash during a file system update could leave the file system inconsistent — for example, a block marked as allocated but not pointed to by any file. With journaling, we can replay or undo incomplete operations after a crash, ensuring consistency.

**Q: What's the difference between buffering and caching?**

Buffering handles speed mismatch — data is temporarily held in a buffer before being written to a slower device. Caching keeps frequently accessed data in fast memory to avoid repeated slow accesses. Both improve performance, but buffering is about smoothing out speed differences while caching is about avoiding repeated accesses.

## Key Takeaways

A file system provides the abstraction of named, persistent storage over raw disk blocks. It handles naming, organization, persistence, and protection.

Hierarchical directories are the modern standard, allowing arbitrary organization. Hard links share the same inode, while symbolic links contain a path and can cross file systems.

File allocation methods trade off between simplicity, performance, and flexibility. Contiguous allocation is fast but inflexible. Linked allocation is flexible but slow for random access. Indexed allocation provides good random access. The Unix inode structure is a clever hybrid optimized for the common case of small files.

Free space is typically managed with bitmaps — simple, efficient, and cacheable. DMA is essential for high-speed I/O, freeing the CPU from byte-by-byte transfers.

Caching dramatically improves performance by keeping frequently accessed blocks in memory. Journaling enables crash recovery by logging changes before applying them. These two features are essential in modern file systems.
