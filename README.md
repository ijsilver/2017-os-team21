# Synopsis
proj1 implements a syscall 'ptree' at syscall number 380 that gets a buffer and size of that buffer from user to return number of processes now running with buffer filled with information of the processes. Int value to specify size of the buffer would also be updated.

# Files Changed
## Preparation
- include/linux/prinfo.h 

  define struct of prinfo

## Define Syscall of ptree

- include/linux/syscalls.h

  define sys_ptree
  
  include linux/prinfo.h
    
- arch/arm/include/asm/unistd.

  enlarge number of syscalls from 380 to 384, in alignment of 4 bytes

- arch/arm/include/uapi/asm/unistd.h <br/>

  define ptree as syscall_base+380

- arch/arm/kernel/calls.S  <br/>
  
  include call of sys_ptree

## Make ptree Logic

- kernel/ptree.c <br/>

  to be explained.
  
## Test Code

- artik/main.c <br/>

  print out process tree

- artik/test.c  <br/>
  
  test for developers to check if kernel code is working correctly

# How ptree Is Implemented

sys_ptree gets two arguments: struct prinfo * buf and int * nr.  <br/>

ptree checks errors of  <br/>
1. EINVAL: buf, nr null pointers 
2. EFAULT: buf, nr not in user address space
3. ENOMEM: unable to allocate kernel heap memory  <br/>

It locks tasklist_lock and runs dfs starting from init_task. <br/>
While running dfs, it copies process information into kernel buffer kbuf. <br/>
After dfs, it unlocks tasklist_lock and copy kernel buffer entries into user buffer buf. <br/>
Then it updates nr to match the number of entries copied from kbuf to buf and returns process count. <br/>

## Functions and Fuctionalities
- first_child_task(&struct task_struct)  <br/>

  macro: exploits list_first_entry macro. only used when child exists.

- next_sibling_task(&struct task_struct) <br/>
   
  macro: exploits list_first_entry macro. only used when next sibling exists.

- child_pid(struct task_struct *) <br/>
  
  inline: exploits list_empty and first_child_task macro. Used for prinfo.

- sibling_pid(struct task_struct *) <br/>
  
  inline: exploits list_is_last and next_sibling_task macro. Used for prinfo.

- void print_prinfo(struct task_struct *) <br/>
  
  function: debugging purpose

- void __write_prinfo (struct prinfo *, struct task_struct *) <br/>

  function: writes task infomation to prinfo  
- void dfs_task_rec(struct task_struct *) <br/>
  
  function: recursive dfs. Implemented but not used.

- int dfs_init_task(struct prinfo *, int, int *)  <br/>
  
  function: runs dfs starting from init_task, without stack but with if.  <br/>
  If current task has child task (!list_empty(&task->children) it moves on to that child.  <br/>
  If current task has no child but next sibling task(!list_is_last(&task->sibling, &task->real_parent->children), then it moves to that sibling.  <br/>
  If current task has no child and no sibling, it moves to its ancestors who have next sibling.  <br/>
  If current task comes all the way back to init_task, then it breaks.  <br/>
  While doing so, for each task current task has moved through, task information is copied to kbuf until buffer is full or it dfs through all processes.  <br/>
  Count of copied entries of buffer is updated to int *. <br/>
  Returns count of all processes running. <br/>
- int do_ptree(struct prinfo *, int *) <br/>
  function: checks for errors, define and malloc kbuf, lock tasklist_lock, call dfs, unlock tasklist_lock, copy from kbuf to buf, knr to nr, free kbuf and return result.
- SYSCALL_DEFINE2(ptree, struct prinfo*, buf, int*, nr) <br/>
  syscall: calls do_ptree.
