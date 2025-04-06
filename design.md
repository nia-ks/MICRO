# miniFS: Un système de fichiers simple

## Abstrait

Ce document se veut une description de nos choix de design pour l'impémentation de miniFS, un système de fichiers proposant des opérations de base sur les fichiers, la gestion des permissions et des liens. Il fait office de description technique des structures de données et algorithmes utilisés pour son fonctionnement.

## Introduction

miniFS propose l'implémentation d'un système de fichiers simple qui permet des opérations telles que la création et la suppression de fichiers et des repertoires, la copie et e déplacement de fichiers et la navigation dans une arborescence. Sont également proposés la gestion des droits d'accès, ainsi que les liens symboliques et en dur. Le système se base sur un fichier UNIX servant de simulation d'une partition. 

Dans le présent document, nous détaillons la raison derrière nos décisions techniques, en décrivant notamment les structures de données et algorithmes mis en place.

## Disposition du système de fichiers

Le système de fichiers est organisé en blocs de taille fixe (4096 octets par défaut). La disposition du système de fichiers est la suivante :

- Bloc 0 : Superbloc
- Bloc 1 : Inode bitmap (peut s'étendre sur plusieurs blocs)
- Bloc N : Block bitmap (peut s'étendre sur plusieurs blocs)
- Bloc M : Table des inodes (peut s'étendre sur plusieurs blocs)
- Bloc P : Blocs de données

### Superbloc

Le superbloc contient des métadonnées sur le système de fichiers, comprenant :

- Numéro magique pour identifier le système de fichiers
- Taille des blocs en octets
- Nombre total de blocs
- Nombre de blocs libres
- Nombre total d'inodes
- Nombre d'inodes libres
- Index du premier bloc de données
- Index de l'inode du répertoire racine
- Numéros de blocs pour l'inode bitmap, la block bitmap et la table des inodes


### Inode Bitmap
L'inode bitmap est utilisé pour suivre quels inodes sont libres et lesquels sont utilisés. Chaque bit dans la bitmap correspond à un inode, avec une valeur de 1 indiquant que l'inode est utilisé et 0 indiquant qu'il est libre.

### Block bitmap

De façon similaire, la block bitmap traque quels blocs sont libres et lesquels sont utilisés. Chaque bit correspond à un bloc, avec 1 indiquant qu'il est utilisé et 0 indiquant qu'il est libre.

### Table des inodes

La table des inodes contient les inodes de l'ensembe des fichiers, répertoires et liens dans le système de fichiers. Chaque inode est associée à des métadonnées, dont :

- Le type de fichiers et ses permissions
- Identifiant du propriétaire et du groupe
- La taille du fichier en octets
- Timestamp du dernier accès, modification et de mise à jour
- Nombre de blocs alloué
- Block pointers (direct and indirect)
- Pointeurs vers les blocs du fichier, directs ou indirects
- Nombre de liens en dur vers cet inode

### Blocs de données

Les blocs de données conservent le contenu réel des fichiers et des répertoires. Pour les fichiers simples, les blocs de données contiennent le contenu du fichier. Pour les répertoires, ils contiennent des entrées de répertoire, qui associent les noms de fichiers à leur numéro d'inodes.

## Structures de données

### Superblocs

La structure pour décrire un superbloc est la suivante :

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

La structure pour décrire un inode est la suivante :

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

### Entrée dans un répertoire 

La structure pour décire une entrée dans un répertoire est la suivante :

```c
typedef struct {
    uint32_t inode;               /* Inode number */
    uint8_t name_len;             /* Length of the name */
    uint8_t file_type;            /* Type of file */
    char name[FS_MAX_FILENAME];   /* Filename */
} fs_dir_entry_t;
```

### Gestionnaire du système de fichiers

La structure du gestionnaire du système de fichiers est utilisée pour gérer le système de fichiers "mounted", c'est-à-dire quand il est rendu accessibe à l'utilisateur :

```c
typedef struct {
    int fd;                       /* File descriptor of the file system file */
    fs_superblock_t superblock;   /* Cached superblock */
    void* block_bitmap;           /* Cached block bitmap */
    void* inode_bitmap;           /* Cached inode bitmap */
} fs_t;
```

## Algorithmes

### Création du système de fichiers

The file system creation algorithm (fs_create) performs the following steps:

1. Create a file of the specified size
3. Initialize the superblock with appropriate values
4. Initialize the inode bitmap (all free)
5. Initialize the block bitmap (mark system blocks as used)
6. Create the root directory with "." and ".." entries

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
