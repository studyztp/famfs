# FAMFS with File Removal and Non-Contiguous Allocation Support  

## Overview  
This repository improves upon **FAMFS (Fault-Tolerant and Adaptive Multi-Node File System)** by adding **file removal functionality** and **non-contiguous allocation**. These enhancements allow FAMFS to properly free and reuse storage space after file deletions, preventing false "Out of Space" errors.  

## Improvements  

### 1. File Removal Functionality  
We modified the FAMFS library to support file removal while ensuring proper synchronization and consistency across client nodes. Key changes include:  
- **Log Entry Modification**: When a file is deleted, its log entry is updated to reflect its removal by setting its size and extent count to zero.  
- **Bitmap Creation Updates**: The system now skips removed file extents when rebuilding the bitmap.  
- **Log Play Adjustments**: Removed files are skipped during log playback to prevent their recreation on client nodes.  
- **Master Node Handling**: The master node ensures that deleted files are unlinked properly.  

### 2. Non-Contiguous Allocation  
Previously, FAMFS only allocated files contiguously, which caused false **"Out of Space"** errors when free space was fragmented. To address this, we introduced:  
- **New Allocation Functions**:  
  - `bitmap_alloc_noncontiguous()`: Identifies and returns available data block lists.  
  - `famfs_alloc_noncontiguous()`: Maps files to these blocks and records their locations in the file creation log.  
- **Space Reuse**: After file removal, the freed space is now utilized efficiently, improving overall storage efficiency.  

## How It Works  
1. **File Removal Process**  
   - The master node calls `famfs rm`, triggering `do_famfs_cli_rm()` → `famfs_rm()`.  
   - `famfs_rm()` verifies the file’s existence and calls `famfs_free_file_memory()`.  
   - The log entry is updated to reflect the deletion, overwriting the previous entry.  
   - The file is then unlinked from the master’s filesystem.  

2. **Non-Contiguous Allocation**  
   - When a new file is created, available non-contiguous blocks are identified.  
   - The file is mapped to these blocks and its location is stored in the log.  
   - Freed space from removed files is now efficiently reused.  

## Benefits  
- **Improved Storage Efficiency**: Eliminates wasted space due to fragmentation.  
- **Prevents False "Out of Space" Errors**: Ensures files can be allocated even in fragmented storage.  
- **Seamless Integration**: Works within FAMFS's existing logging and synchronization system.  

## Future Work  

### 1. Log Entry Removal Instead of Alteration  
Currently, our file removal implementation **modifies** log entries instead of removing them. However, since FAMFS has a **fixed number of log entries**, continuously altering logs without removing them may eventually lead to a situation where the system **runs out of log entries**, even if there are available data blocks for new files. A future improvement would be to implement **log entry removal**, ensuring better log space management.

### 2. Expanding `famfs rm` Functionality  
At present, `famfs rm` can only remove **one file at a time** and **cannot remove directories**. While this does not critically impact the system—since multiple `famfs rm` calls can be made sequentially—it would be beneficial to extend its functionality to **batch file deletions and directory removals** for greater usability.

### 3. Adding `famfs append` and `famfs truncate`  
The current improvements to the FAMFS library **lay the groundwork** for implementing `famfs append` and `famfs truncate`. However, these operations were not added due to time constraints. Implementing them would primarily require **adding the corresponding CLI functions**, as the underlying library changes already support these operations.

### 4. Dynamic Runtime Operations and Synchronization  
A major future direction for this work is **dynamic runtime file operations**, where file allocation changes can be handled efficiently **without excessive synchronization overhead**. This is challenging because:  
- All nodes must remain **synchronized** to maintain consistency, which can introduce **latency** and **performance overhead**.  
- Shared memory communication can **degrade application performance**, making runtime updates costly.  
- Concurrent modifications across multiple users could lead to **data corruption or unintended behavior**.  

To **reduce synchronization costs**, a possible approach is **decoupling file removal and data access**:  
- Each node could remove a file from its own memory space by adjusting the file pointer at the kernel level.  
- A **metadata tracking system** would manage file statuses across nodes, ensuring that once all nodes remove a file, the space is freed.  

For **file appending**, synchronization remains critical:  
- The **writer node** must **stall all nodes** until the append operation completes.  
- After appending, all nodes must **synchronize their file pointers** to maintain correctness.  
- Importantly, **only the writer-exclusive node** should issue an append operation to prevent inconsistencies.  

While this approach could **minimize synchronization overhead**, its practical implementation might introduce unforeseen challenges and require additional optimizations.

## Evaluation

We evaluated our improvements to FAMFS using QEMU for functional testing. The evaluation consists of three cases:  

1. **Reusing free space after file removal**  
2. **Replaying filesystem events after removal and space reuse**  
3. **Verifying file correctness for a non-contiguously allocated file using logplay**  

### Case 1: Reusing Free Space After Removal  

To verify that free space can be reused after file removal, we performed the following steps:  

1. Created two files (`test-file-1.txt` and `test-file-2.txt`), each 100 MB in size.  
2. Removed `test-file-1.txt` to create a **non-contiguous** free space.  
3. Created a new `test-file-1.txt` with a size of 1 GB, forcing the system to allocate it non-contiguously.  

#### Output:  
The output confirms that the newly created 1 GB file successfully reused the space freed from the removed file:  

```sh
➜ famfs creat -s 100M -r /mnt/famfs-mount/test-file-1.txt
offset: 5, len: 50
famfs_file_alloc: nextents 1
ext_list[0].offset = 10485760
ext_list[0].len    = 104857600

➜ famfs creat -s 100M -r /mnt/famfs-mount/test-file-2.txt
offset: 55, len: 50
famfs_file_alloc: nextents 1
ext_list[0].offset = 115343360
ext_list[0].len    = 104857600

➜ famfs rm /mnt/famfs-mount/test-file-1.txt
File /mnt/famfs-mount/test-file-1.txt removed successfully.

➜ famfs creat -s 1G -r /mnt/famfs-mount/test-file-1.txt
offset: 5, len: 50
offset: 105, len: 462
famfs_file_alloc: nextents 2
ext_list[0].offset = 10485760
ext_list[0].len    = 104857600
ext_list[1].offset = 220200960
ext_list[1].len    = 968884224
```

This confirms that our non-contiguous allocation successfully reuses free space.  

---

### Case 2: Filesystem Replay After Removal and Space Reuse  

Due to experimental setup limitations, we simulated logplay by:  

1. Unmounting the current FAMFS mount point.  
2. Remounting FAMFS to force a **filesystem rebuild using logplay**.  

#### Output:  
```sh
➜ sudo umount /mnt/famfs-mount
➜ ls /mnt/famfs-mount
➜ sudo famfs mount /dev/dax0.0 /mnt/famfs-mount/
famfs_module_loaded: YES
ext_list[0].offset = 0
ext_list[0].len    = 2097152
ext_list[0].offset = 2097152
ext_list[0].len    = 8388608
famfs_mkmeta: Meta files successfully created
ext_list[0].offset = 115343360
ext_list[0].len    = 104857600
ext_list[0].offset = 10485760
ext_list[0].len    = 104857600
ext_list[1].offset = 220200960
ext_list[1].len    = 968884224
famfs_logplay: processed 3 log entries; 2 new files; 0 new directories
```

This confirms that:  
- The removed file **does not reappear** after logplay.  
- The log entry for the removed file **remains**, as expected, since we overwrite the creation log instead of deleting it.  

---

### Case 3: File Correctness Verification After Logplay  

To ensure that non-contiguous allocation does not corrupt data, we:  

1. Created a copy (`test-file-1-cp.txt`) of a non-contiguously allocated file.  
2. Unmounted and remounted the filesystem.  
3. Compared the copied file with the original after logplay.  

#### Output:  
```sh
➜ famfs cp /mnt/famfs-mount/test-file-1.txt /mnt/famfs-mount/test-file-1-cp.txt
offset: 567, len: 512
famfs_file_alloc: nextents 1
ext_list[0].offset = 1189085184
ext_list[0].len    = 1073741824

➜ diff test-file-1.txt test-file-1-cp.txt
➜ diff test-file-1.txt test-file-2.txt
Binary files test-file-1.txt and test-file-2.txt differ

➜ sudo umount /mnt/famfs-mount
➜ sudo famfs mount /dev/dax0.0 /mnt/famfs-mount/
famfs_module_loaded: YES
ext_list[0].offset = 0
ext_list[0].len    = 2097152
ext_list[0].offset = 2097152
ext_list[0].len    = 8388608
famfs_mkmeta: Meta files successfully created
ext_list[0].offset = 115343360
ext_list[0].len    = 104857600
ext_list[0].offset = 10485760
ext_list[0].len    = 104857600
ext_list[1].offset = 220200960
ext_list[1].len    = 968884224
ext_list[0].offset = 1189085184
ext_list[0].len    = 1073741824
famfs_logplay: processed 4 log entries; 3 new files; 0 new directories

➜ sudo chown -R user /mnt/famfs-mount
➜ diff test-file-1.txt test-file-1-cp.txt
```

Since `diff` does not return any differences between the copied and original files, this confirms that:  
- **Logplay correctly reconstructs files** after remounting.  
- **Non-contiguous allocation preserves file integrity**, ensuring that data remains unchanged even after log replay.  

---

### Summary  

| Case | Test Description | Expected Result | Outcome |
|------|-----------------|----------------|---------|
| **Case 1** | Reuse free space after removal | New file is allocated non-contiguously | ✅ Success |
| **Case 2** | Replay filesystem events after removal | Removed files do not reappear | ✅ Success |
| **Case 3** | Verify file correctness after logplay | Copied file matches original after remount | ✅ Success |

Our evaluation demonstrates that:  
- **File removal properly frees storage for reuse.**  
- **Filesystem replay correctly maintains state after unmounting and remounting.**  
- **Non-contiguous allocation does not impact data integrity.**  

These results validate the functionality and effectiveness of our modifications to FAMFS. 

