# miniFS - A Simple File System

miniFS is a simple file system implementation that supports basic file operations, access rights management, and links.

## Features

- Basic file operations (create, delete, copy, move)
- Directory operations (create, delete, navigate)
- Access rights management (chmod, chown)
- Hard links and symbolic links
- Free space management after file deletion

## Installation

### Prerequisites

- Linux operating system
- GCC compiler
- Make

### Building from Source

1. Clone the repository:
   ```
   
   cd miniFS
   ```

2. Compile the project:
   ```
   make
   ```

3. Install the program:
   ```
   make install
   ```

   This will install the program to `/usr/local/bin` if you have write permissions, or to `~/.local/bin` otherwise.

### Building Documentation

If you have Doxygen installed, you can generate the HTML documentation:

```
make doc
```

The documentation will be available in the `docs/html` directory.

## Usage

### Creating a File System

To create a new file system, use the `mkfs` command:

```
minifs
> mkfs myfs.dat 10M
```

This creates a 10 MB file system in the file `myfs.dat`.

### Mounting a File System

To mount a file system, use the `mount` command:

```
minifs
> mount myfs.dat
```

### Basic Operations

Once a file system is mounted, you can perform various operations:

#### File Operations

- Create a file: `touch /path/to/file`
- Delete a file: `rm /path/to/file`
- Copy a file: `cp /source/file /dest/file`
- Move a file: `mv /source/file /dest/file`
- Display file contents: `cat /path/to/file`
- Write to a file: `write /path/to/file Some text to write`

#### Directory Operations

- Create a directory: `mkdir /path/to/dir`
- Remove a directory: `rmdir /path/to/dir`
- Change directory: `cd /path/to/dir`
- List directory contents: `ls [path]`

#### Access Rights

- Change permissions: `chmod 644 /path/to/file`

#### Links

- Create a hard link: `ln /target/file /link/name`
- Create a symbolic link: `symln /target/file /link/name`

### Unmounting a File System

To unmount a file system, use the `umount` command:

```
minifs
> umount
```

### Exiting the Program

To exit the program, use the `exit` command:

```
minifs
> exit
```

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Authors

- YOUCEF KHODJA Mastina
- BOUHENNI Nour El Houda 
- KESSAL Anais