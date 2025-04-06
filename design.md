# miniFS: A Simple File System

## Abstract

This document describes the design and implementation of miniFS, a simple file system that supports basic file operations, access rights management, and links. It explains the data structures and algorithms used to implement the file system.

## Introduction

miniFS is a simple file system implementation that supports basic file operations (create, delete, copy, move), directory operations (create, delete, navigate), access rights management, and both hard and symbolic links. The file system is implemented as a UNIX file with a simulated partition.

This document describes the design decisions, data structures, and algorithms used in the implementation of miniFS.

## File System Layout

The file system is organized into blocks of fixed size (4096 bytes by default). The layout of the file system is as follows:

- Block 0: Superblock
- Block 1: Inode bitmap (may span multiple blocks)
- Block N: Block bitmap (may span multiple blocks)
- Block M: Inode table (may span multiple blocks)
- Block P: Data blocks

### Superblock

The superblock contains metadata about the file system, including:

- Magic number to identify the file system
- Block size in bytes
- Total number of blocks
- Number of free blocks
- Total number of inodes
- Number of free inodes
- Index of the first data block
- Index of the root directory inode
- Block numbers for the inode bitmap, block bitmap, and inode table

### Inode Bitmap

The inode bitmap is used to track which inodes are free and which are in use. Each bit in the bitmap corresponds to one inode, with a value of 1 indicating that the inode is in use and 0 indicating that it is free.

### Block Bitmap

Similarly, the block bitmap tracks which blocks are free and which are in use. Each bit corresponds to one block, with 1 indicating in use and 0 indicating free.

### Inode Table

The inode table contains the inodes for all files, directories, and links in the file system. Each inode stores metadata about a file, including:

- File type and permissions
- Owner and group IDs
- File size in bytes
- Access, modification, and change times
- Number of blocks allocated
- Block pointers (direct and indirect)
- Number of hard links to this inode

### Data Blocks

The data blocks store the actual contents of files and directories. For regular files, the data blocks contain the file contents. For directories, they contain directory entries, which map file names to inode numbers.

## Data Structures

### Superblock

The superblock structure is defined as follows:

```c
typedef struct {
    uint32_t magic;               /* Magic number to identify the file system */
    uint32_t block_size;          /* Size of blocks in bytes */
    uint32_t num_blocks;          /* Total number of blocks in the file system */
    uint32_t num_free_blocks;     /* Number of free blocks */
    uint32_t num_inodes;          /* Total number of inodes */
    uint32_t num_free_inodes;     /* Number of free inodes */
    uint32_t first_data_block;    /* Index of the first data block */
    uint32_t root_inode;          /* Index of the root directory inode */
    uint32_t inode_bitmap_block;  /* Block containing the inode bitmap */
    uint32_t block_bitmap_block;  /* Block containing the block bitmap */
    uint32_t inode_table_block;   /* First block of the inode table */
} fs_superblock_t;
```

### Inode

The inode structure is defined as follows:

```c
typedef struct {
    uint32_t mode;                /* File type and permissions */
    uint32_t uid;                 /* User ID of owner */
    uint32_t gid;                 /* Group ID of owner */
    uint32_t size;                /* Size of file in bytes */
    time_t atime;                 /* Last access time */
    time_t mtime;                 /* Last modification time */
    time_t ctime;                 /* Last status change time */
    uint32_t blocks;              /* Number of blocks allocated */
    uint32_t direct_blocks[12];   /* Direct block pointers */
    uint32_t indirect_block;      /* Single indirect block pointer */
    uint32_t link_count;          /* Number of hard links to this inode */
} fs_inode_t;
```

### Directory Entry

The directory entry structure is defined as follows:

```c
typedef struct {
    uint32_t inode;               /* Inode number */
    uint8_t name_len;             /* Length of the name */
    uint8_t file_type;            /* Type of file */
    char name[FS_MAX_FILENAME];   /* Filename */
} fs_dir_entry_t;
```

### File System Handle

The file system handle structure is used to manage the mounted file system:

```c
typedef struct {
    int fd;                       /* File descriptor of the file system file */
    fs_superblock_t superblock;   /* Cached superblock */
    void* block_bitmap;           /* Cached block bitmap */
    void* inode_bitmap;           /* Cached inode bitmap */
} fs_t;
```

## Algorithms

### File System Creation

The file system creation algorithm (fs_create) performs the following steps:

1. Create a file of the specified size
2. Initialize the superblock with appropriate values
3. Initialize the inode bitmap (all free)
4. Initialize the block bitmap (mark system blocks as used)
5. Create the root directory with "." and ".." entries

### Mounting and Unmounting

The mounting algorithm (fs_mount) performs the following steps:

1. Open the file system file
2. Read the superblock and verify the magic number
3. Load the block and inode bitmaps into memory
4. Initialize the file descriptor table

The unmounting algorithm (fs_unmount) performs the following steps:

1. Write the block and inode bitmaps back to disk
2. Write the superblock back to disk
3. Free memory used by the bitmaps
4. Close the file system file

### File and Directory Operations

#### File Creation

The file creation algorithm (fs_create_file) performs the following steps:

1. Resolve the parent directory path
2. Check if the file already exists
3. Allocate a new inode
4. Initialize the inode with appropriate values
5. Write the inode to disk
6. Add a directory entry to the parent directory

#### File Deletion

The file deletion algorithm (fs_delete) performs the following steps:

1. Resolve the parent directory path
2. Find the file in the parent directory
3. Read the file's inode
4. Free all blocks used by the file
5. Free the inode
6. Remove the directory entry from the parent directory

#### File Reading and Writing

The file reading algorithm (fs_read) performs the following steps:

1. Verify the file descriptor
2. Read the file's inode
3. Calculate the block number and offset for the current position
4. Read data from the blocks
5. Update the access time

The file writing algorithm (fs_write) performs the following steps:

1. Verify the file descriptor and write permissions
2. Read the file's inode
3. Calculate the block number and offset for the current position
4. Allocate new blocks if necessary
5. Write data to the blocks
6. Update the file size and modification time

#### Directory Listing

The directory listing algorithm traverses the directory blocks and reads the directory entries, displaying information about each file or subdirectory.

### Access Rights Management

The access rights checking algorithm (check_access) verifies that the user has the requested permissions based on the file mode and the user's identity.

The chmod and chown algorithms modify the mode, owner, and group of a file's inode.

### Link Management

#### Hard Links

The hard link creation algorithm (fs_link) performs the following steps:

1. Resolve the target file path to an inode
2. Verify that the target is a regular file
3. Add a directory entry to the parent directory of the new link
4. Increment the link count in the target inode

#### Symbolic Links

The symbolic link creation algorithm (fs_symlink) performs the following steps:

1. Allocate a new inode
2. Initialize the inode as a symbolic link
3. Allocate a block to store the target path
4. Write the target path to the block
5. Add a directory entry to the parent directory

### Free Space Management

Free space is managed using the block and inode bitmaps. When a block or inode is allocated, the corresponding bit in the bitmap is set to 1. When a block or inode is freed, the corresponding bit is cleared (set to 0).

The algorithms for allocating and freeing blocks and inodes use the bitmap functions to find free resources and mark resources as free or in use.

## Performance Considerations

### Caching

The file system caches the superblock, block bitmap, and inode bitmap in memory to reduce disk I/O. However, file data is not cached, which means each read or write operation requires disk access.

### Disk Layout

The disk layout is designed to minimize seek time by placing related structures close together. For example, the inode bitmap, block bitmap, and inode table are placed at the beginning of the disk, followed by the data blocks.

### Block Size

The default block size is 4096 bytes, which is a common choice for file systems. This size is a trade-off between space efficiency (larger blocks waste more space due to internal fragmentation) and I/O efficiency (larger blocks reduce the number of disk accesses).

## Limitations and Future Work

### Limitations

The current implementation has several limitations:

- No support for journaling or crash recovery
- Limited file size (maximum of 12 direct blocks + 1 indirect block)
- No support for file locking
- No support for file attributes beyond the standard UNIX attributes

### Future Work

Possible extensions to the file system include:

- Adding support for double and triple indirect blocks to allow larger files
- Implementing journaling for crash recovery
- Adding support for file locking for concurrent access
- Implementing file caching to improve performance
- Adding support for extended attributes

## Conclusion

miniFS is a simple but functional file system that provides the basic features required for file management. Its design is inspired by traditional UNIX file systems, with inodes, blocks, and bitmaps for free space management.

The implementation balances simplicity with functionality, providing a good foundation for understanding file system concepts and potentially extending the system with more advanced features. 