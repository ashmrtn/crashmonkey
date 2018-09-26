# Automatic Crash Explorer (Ace) #

### Overview ###
Ace is a bounded, and exhaustive workload generator for POSIX file systems. A workload is simply a sequence of file-system operations. Ace comprises of two main components

1. **High-level workload generator** : This is responsible for exhaustively generating workloads within the defined bounds. The generated workloads are represented in a high-level language which resembles the one below.
```
mkdir B 0777
open Bfoo O_RDWR|O_CREAT 0777
fsync Bfoo
checkpoint 1
close Bfoo
```

2. **CrashMonkey adapter** : Workloads represented in the high-level language have to be converted to a [format](workload.md) that CrashMonkey understands. This is taken care of by the adapter. For example, the above workload is converted to the run method of CrashMonkey as follows.

```c++
virtual int run( int checkpoint ) override {

    test_path = mnt_dir_ ;
    B_path =  mnt_dir_ + "/B";
    Bfoo_path =  mnt_dir_ + "/B/foo";

    int local_checkpoint = 0 ;

    if ( mkdir(B_path.c_str() , 0777) < 0){
      return errno;
    }

    int fd_Bfoo = cm_->CmOpen(Bfoo_path.c_str() , O_RDWR|O_CREAT , 0777);
    if ( fd_Bfoo < 0 ) {
      cm_->CmClose( fd_Bfoo);
      return errno;
    }

    if ( cm_->CmFsync( fd_Bfoo) < 0){
      return errno;
    }

    if ( cm_->CmCheckpoint() < 0){
      return -1;
    }
    local_checkpoint += 1;
    if (local_checkpoint == checkpoint) {
      return 1;
    }

    if ( cm_->CmClose ( fd_Bfoo) < 0){
      return errno;
    }

    return 0;
}
```

---
### Bounds used by Ace ###

Ace currently generates workloads of sequence length 1, 2, and 3. We say a workload has sequence length 'x' if it has 'x' core file-system operations in it. Currently, Ace places the following bounds.

1. **Sequence length** : Only sequences of length up to 3 have been tested so far.

2. **File-system operations** : Ace currently supports the following system calls :

    `creat, mkdir, falloc, write, direct write, mmap write, link, unlink, remove, rename, fsetxattr, removexattr, truncate, fdatasync, fsync, sync, symlink`

    However, be cautious of the number of workloads that can be generated in each sequence length, if you decide to test this whole set of operations. As you go higher up the sequence length, it is advisable to narrow down the operation set, or increase the compute available to test.

3. **File set** : Ace supports only a pre-defined file set. You have the freedom to restrict it to a smaller set. Currently, we support 4 directories and 2 files within each directory.

    ```
    dir : test (files: foo, bar)
          |
          |__ dir : A (files: foo, bar)
          |         |
          |         |_ dir C (files: foo, bar)
          |
          |__ dir : B (files: foo, bar)
    ```

However, by default, we only use directory A, B, and the files under them. To include a level of nesting, add the files under test dir or C to the file set.

4. **Persistence operations** : Ace supports four types of persistence operations to be included in the workload

    `sync, fsync, fdatasync, none`
    Since by default `fdatasync` is one of the file-system operations under test, it is excluded from the list of persistence operations. However you could add it to the persistence operation set in the ace script.

5. **Fileset for persistence operations** : Ideally, the file set for persistence operations should be the same as the above defined file set. However, there is no point persisting a file that was not touched by any of the system calls in the workload. Hence, we reduce the space of workloads further by restricting the fileset for persistence operations:
      1. Range of used files : In this mode, in addition to the files/directories involved in the workload, we allow persisting sibling files and parent directory. This is the default for seq-1 and seq-2.
      2. Strictly used files : In this mode, only files/directories acted upon by the workload can be persisted. We default to this mode in seq-3, to restrict the workload set.

___
### Generating workloads with Ace ###

Generating workloads with Ace is a two-step process.

  1. Set the bounds that you wish to explore, using the guidelines and advices in the [previous](#bounds-used-by-ace) section.

  2. Start workload generation.

      ```python
      cd ace
      python ace.py -l <seq_length> -n <True|False> -d<True|False>
      ```
For example, to generate seq-2 workloads with no additional nested directory, run :
      ```python
      python ace.py -l 2 -n False -d false
      ```

      **Flags:**

      * `-l` - Sequence length of the workload, i.e., the number of core file-system operations in the workload.
      * `-n` - If True, provides an additional level of nesting to the file set. Adds a directory `A/C` and two files `A/C/foo` and `A/C/bar` to the set of files.
      * `-d` - Demo workload. If true, simply restricts the workload space to test two file-system operations `link` and `fallocate`, allowing the persistence of used files only. The file set is also restricted to just `foo` and `A/bar`

___
### Generalizing Ace ###