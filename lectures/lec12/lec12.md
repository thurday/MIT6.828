# Lec 12

## File system overview

Importance. Challenges.



## Disk

IDE: a standard to control devices. 

HDD and SSD. Sequential access much faster than random access. 

On-disk layout: bootloader, superblock, logs, inodes, bitmap, data blocks. This layout is generated by the `mkfs` program.

```
0: 		  booloader, not used by the file system
1: 		  super block
2: 		  log for transactions
32 - 57:  array of inodes, packed into blocks
58: 	  block in-use bitmap (0=free, 1=used)
59 - end: file/dir content blocks
```

Direct vs indirect blocks. 

Transfer inode number to disk address of the inode data structure. 

The file system can be viewed as an on-disk data strucure, with 2 allocation pools: inodes and blocks. 

### Links

Symbolic/soft link: a file `foo` has a reference to a file `bar`. If `bar` is deleted, using `foo` is error.

Hard link: `foo` and `bar` both refer to the same file. If one is deleted, the other still refers to the file.



## Xv6 code

### Create a file

Call graph for `$ echo > a`.

```
sys_open      sysfile.c
  create      sysfile.c
    ialloc    fs.c		 	(Write to block 34. mark inode allocated)
    iupdate   fs.c			(Write to block 34, update fields of inode)
    dirlink   fs.c
      writei  fs.c			(Write to block 59, write to inode)
```

- What's in block 34? 

  We know block 32-57 are for inodes. Why write to 34, instead of 32 or 33? Answer: there're other files created before `a`. Block 34 contains the inode for file `a`.

- Why **two** writes to block 34?

  The first write in in `ialloc`, allocating an inode. The second write is in `writei`, writing to inode to update the ref count. The reason we cannot do these two actions in one step is that `ialloc` returns an *unlocked* inode, thus `ilock` should be called before calling `iupdate`. So we have to do the write in two steps and insert the call to `ilock`.

- What's in block 59?

  We know blocks starting from block 58 are data blocks. So block 59 is the content of `a`. 

- What if there're concurrent calls to `ialloc`? Would they get the same inode? 

  No. `bread` is called to read the inode block from disk to buffer. `bread` locks the block, so only one call to `ialloc` would get access to the inode block. 



### Write to a file

For `$ echo x > a`:

```
sys_write       sysfile.c
  filewrite     file.c
    writei      fs.c
      bmap      fs.c    
        balloc	fs.c	
          bzero fs.c
      iupdate	fs.c
      
write 58 	balloc  (from bmap, from writei, allocating block 508)
write 508 	bzero	(write 'x')
write 508 	writei  (from filewrite file.c)
write 34 	iupdate (from writei, updating the size, addrs)
write 508 	writei	(write '\n')
write 34 	iupdate (from writei, updating the size, addrs)
```

- What's in block 58?
  The bitmap block.

- What's in block 508?

  The data block of file `a`.

- What's in block 640?

  ```
  x\n\0\0\0\0...\0
  ```

- Why the iupdate?
  File length and addrs[]

- Why *two* `writei `+ `iupdate`?
  `echo` calls write() twice, 2nd time for the newline. `echo` writes one character at a time.



### Delete a file

For `$ rm a`:

```
sys_unlink
  writei
  iupdate
  iunlockput
    iput
      itrunc
        bfree
        iupdate
      iupdate

write 59 writei 	(from sys_unlink; clear directory content)
write 34 iupdate 	(from sys_unlink; update link count of file)
write 58 bfree  	(from itrunc, from iput)
write 34 iupdate 	(from itrunc; zeroed length)
write 34 iupdate 	(from iput; marked free)
```

- What's in block 59?

  The directory contain `a`.

- What's in block 34?

  The inode for `a`.

- What's in block 58?

  The bitmap. 

- Why three `iupdate`s?

  The first `iupdate` is updating the content of the directory. The second `iupdate` is updating the inode of the unlinked file `a`. The third update is to free the on-disk space. 



## Block Cache

Two levels of caching: `bcache.lock` protects the description of what's in the cache, `buf->lock` protects the content of the buffer. 

Block cache replacement policy: `bget` re-uses `bcache.head.prev`, the in used blocks. `brelse` moves the released block to `bcache.head.next`.



