# miniFS: Un système de fichiers simple

## Abstrait

Ce document se veut une description de nos choix de design pour l'impémentation de miniFS, un système de fichiers proposant des opérations de base sur les fichiers, la gestion des permissions et des liens. Il fait office de description technique des structures de données et algorithmes utilisés pour son fonctionnement.

## Introduction

miniFS propose l'implémentation d'un système de fichiers simple qui permet des opérations telles que la création et la suppression de fichiers et des repertoires, la copie et e déplacement de fichiers et la navigation dans une arborescence. Sont également proposés la gestion des droits d'accès, ainsi que les liens symboliques et en dur. Le système se base sur un fichier UNIX servant de simulation d'une partition. 

Dans le présent document, nous détaillons la raison derrière nos décisions techniques, en décrivant notamment les structures de données et algorithmes mis en place.

## Disposition du système de fichiers

Le système de fichiers est organisé en blocs de taille fixe (4096 octets/4 Ko par défaut - le plus commun dans les systèmes actuels). La disposition du système de fichiers est la suivante :

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

L'algorithme de création du système de fichiers s'exécute en respectant les étapes qui suivent:

1. Crée un fichier de la taille demandée
2. Procède à l'initialisation du superbloc avec les valeurs appropriées
3. Procède à l'initialisation de l'inode bitmap (toutes libres au début)
4. Procède à l'initialisation de la block bitmap (les blocs sytèmes sont marqués comme utilisés)
5. Crée le répertoire racine avec les entrées spéciales "." et ".."

### Mounting et Unmounting

L'algorithme de montage (fs_mount) s'exécute en respectant les étapes qui suivent :

1. Ouvrir le fichier associé au système de fichiers
2. Lire le superbloc et vérifier le numéro magique
3. Charger les block et inode bitmaps en mémoire
4. Initialiser la table des descripteurs de fichiers

L'algorithme de démontage (fs_unmount) s'exécute en respectant les étapes qui suivent :

1. Écrire les block et inode bitmaps sur le disque
2. Écrire le superbloc sur le disque
3. Libérer la mémoire utilisée par les bitmaps
4. Fermer le fichier associé au système de fichiers

### Actions sur les fichiers et répertoires

#### Création de fichiers

L'algorithme de création d'un fichier (fs_create_file) s'exécute en respectant les étapes qui suivent :

1. Résoudre le chemin du répertoire parent
2. Vérifier si le fichier n'existe pas déjà
3. Allouer un nouvel inode
4. Initialiser l'inode avec les valeurs adéquates
5. Écrire l'inode sur le disque
6. Ajouter une entrée dans le répertoire parent

#### Création de fichiers

L'algorithme de suppression d'un fichier (fs_delete) s'exécute en respectant les étapes qui suivent :

1. Résoudre le chemin du répertoire parent
2. Trouver le fichier dans le répertoire parent
3. Lire l'inode du fichier
4. Libérer tous les blocs utilisés par le fichier
5. Libérer l'inode
6. Supprimer l'entrée dans le répertoire parent

#### Lecture et écriture dans les fichiers

L'algorithme de lecture du contenu d'un fichier (fs_read) s'exécute en respectant les étapes qui suivent :

1. Vérifier le descripteur de fichier
2. Lire l'inode du fichier
3. Calculer le numéro du bloc et le décalage par rapport à la position actuelle
4. Lire les données depuis les blocs
5. Mettre à jour la timestamp d'accès

L'algorithme d'écriture de contenu dans un fichier (fs_write) s'exécute en respectant les étapes qui suivent :

1. Vérifier le descripteur de fichier et les permissions d'écriture
2. Lire l'inode du fichier
3. Calculer le numéro du bloc et le décalage pour la position actuelle
4. Allouer de nouveaux blocs si nécessaire
5. Écrire les données dans les blocs
6. Mettre à jour la taille du fichier et la timestamp de modification

#### Lecture du contenu d'un répertoire

Cet algorithme parcourt les blocs associés au répertoire eten lit les entrées, en restituant les informations sur chaque fichier ou sous-répertoire.

### Gestion des droits d'accès

L'algorithme de vérification des droits d'accès (check_access) vérifie que l'utilisateur dispose des permissions demandées en fonction du mode du fichier et de l'identité de l'utilisateur.

Les algorithmes `chmod` et `chown` modifient le mode, le propriétaire et le groupe dans l'inode d'un fichier.

### Gestion des liens

#### Liens en dur

The hard link creation algorithm (fs_link) performs the following steps:

1. Resolve the target file path to an inode
2. Verify that the target is a regular file
3. Add a directory entry to the parent directory of the new link
4. Increment the link count in the target inode

#### Liens symboliques

L'algorithme de création d'un lien symbolique pour un fichier (fs_write) s'exécute en respectant les étapes qui suivent :

1. Allouer un nouvel inode
2. Initialiser le type dans l'inode comme un lien symbolique
3. Allouer un bloc pour stocker le chemin cible du lien
4. Écrire le chemin dans le bloc
5. Ajouter une entrée dans le répertoire parent

### Gestion de l'espace libre

L'espace libre est géré à l'aide des bitmaps des blocs et des inodes. Lorsqu'un bloc ou un inode est alloué, le bit correspondant dans la bitmap est défini sur 1. Lorsqu'un bloc ou un inode est libéré, le bit correspondant est réinitialisé (défini sur 0).

Les algorithmes pour allouer et libérer des blocs et des inodes utilisent les fonctions de la bitmap pour trouver les ressources libres et marquer les ressources comme libres ou en cours d'utilisation.

## Performance

### Cache

Le système de fichiers met en cache le superbloc, et la block et inode bitmap en mémoire pour réduire les opérations d'entrée/sortie sur disque. Cependant, les données des fichiers ne sont pas mises en cache, ce qui signifie que chaque opération de lecture ou d'écriture nécessite un accès au disque.

### Organisation sur le disque

La disposition du disque est conçue pour minimiser le temps de recherche en plaçant à proximité les structures liées. Par exemple, la bitmap des inodes, la bitmap des blocs et la table des inodes sont  au début du disque, suivies des blocs de données.

### Taille des blocs

La taille par défaut d'un bloc est de 4096 octets, ou 4 Ko, un choix courant pour beaucoup de système de fichiers et SE. Elle se veut un compromis entre l'efficacité liée à 'espace (des blocs plus lourds ont tendance à en gaspiller en raison de la fragmentation interne, moins de chance d'avoir une partie inutilisée du bloc) et l'efficacité des entrées/sorties (des blocs plus lourds réduisent le nombre d'accès au disque, évitant de devoir gérer de nombreux petits blocs). Ele est donc à mis-chemin et convient à de nombreux types de fichiers.

## Limites et pistes d'amélioration

### Limites

L'implémentation actuelle présente plusieurs limites :

- Pas de prise en charge de la journalisation
- Taille de fichier limitée (maximum de 12 blocs directs + 1 bloc indirect)
- Pas de support pour les attributs de fichier au-delà des attributs UNIX standards

### Pistes d'amélioration

Les extensions possibles du système de fichiers incluent :

- Ajouter les blocs indirects doubles et triples pour permettre des fichiers plus volumineux
- Implémenter la journalisation pour la récupération après crash
- Ajouter le verrouillage de fichier pour un accès concurrent
- Implémenter la mise en cache des fichiers pour améliorer les performances
- Ajouter la prise en charge d'attributs étendus

## Conclusion

miniFS est un système de fichiers simple mais fonctionnel qui fournit les fonctionnalités de base requises pour la gestion des fichiers. Sa conception est inspirée des systèmes de fichiers UNIX traditionnels, avec des inodes, des blocs et des bitmaps pour la gestion de l'espace libre.

L'implémentation actuelle se veut à la fois simple dans son utilisation et fonctionnement et fonctionnelle, offrant une bonne base pour comprendre les concepts des systèmes de fichiers et potentiellement étendre le système avec des fonctionnalités plus avancées.
