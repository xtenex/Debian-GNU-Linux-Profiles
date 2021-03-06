# Systemtap  
Systemtap is a tool to detect kernel function. It helps to dynamically analyse kernel( run time).
- Test on Debian GNU/Linux 9 (stretch) 64-bit
## Prepare  
```
apt install systemtap `uname -r`-dbg
```
systemtap is a interpreter for *.stap  
\`uname -r\`-dbg provide kernel with debug information.  

## Grammer
Systemtap is a simple scripting language. Things we need to pay attention to is where to probe and what to print( [from this guide](https://sourceware.org/systemtap/tutorial.pdf)). Here, we have hello world example:
```  
root@debian:~/stap# cat hello.stp 
probe begin
{
    print ("hello world\n")
    exit ()
}	
root@debian:~/stap# stap hello.stp 
hello world

```
### Where to probe
`begin` is a probe point. It will be probe when the *.stap started, then "hello world" will be print. [Here are some probe point list at this tutorial](https://sourceware.org/systemtap/tutorial.pdf):

| probe point | explain |
|--------|--------|
|  begin  | The startup of the systemtap session.|
|end      |The end of the systemtap session.|
|kernel.function("sys_open") |The entry to the function named sys_open in the kernel.|
|timer.ms(200)|A timer that fires every 200 milliseconds.|
|timer.profile |A timer that fires periodically on every CPU.|
|perf.hw.cache_misses |A particular number of CPU cache misses have occurred.|
|procfs("status").read |A process trying to read a synthetic file.|
|process("a.out").statement("*@main.c:200") |Line 200 of the a.out program.|  

We can chose one or more points to be probe. When it is detect by stap, the things between { ... } will run.

### What to print
Here are some functions provide by stap from the [guide](https://sourceware.org/systemtap/tutorial.pdf). It determin what things to print.  

|function/value| explain |
|--------|--------|
|tid() |The id of the current thread.|
|pid() |The process (task group) id of the current thread.|
|uid() |The id of the current user.|
|execname() |The name of the current process.|
|cpu() |The current cpu number.|
|gettimeofday_s()| Number of seconds since epoch.|
|get_cycles()| Snapshot of hardware cycle counter.|
|pp()| A string describing the probe point being currently handled.|
|ppfunc() |If known, the the function name in which this probe was placed.|
|$$vars |If available, a pretty-printed listing of all local variables in scope.|
|print_backtrace() |If possible, print a kernel backtrace.|
|print_ubacktrace()| If possible, print a user-space backtrace.|

## Example  
Our example will print the information of the process when a kernel function was probed. The information include process name, pid, user stack, kernel stack.
```  
probe begin
/*print start when stap begin*/
{
printf ("stap started!\n")
}

probe kernel.function("kmalloc")
{
/*print process information*/
printf ("%s(%d) %s\n", execname(), pid(), pp())

/*print kernel stack dump*/
printf ("---------------------kernel--------------------\n")
print_backtrace()

/*print userspace stack dump*/
printf ("---------------------user-space----------------\n")
print_ubacktrace()
printf ("-----------------------------------------------\n\n")
}

probe end
/*print end when stap end*/
{
printf("stap end!")
}

```  
Here are some result, you may get different information on you machine.
```  
stap started!
kworker/0:1(6489) kernel.function("kmalloc@./include/linux/slab.h:478")
---------------------kernel--------------------
 0xffffffffb40f2ea4 : bio_alloc_bioset+0x1b4/0x240 [kernel]
 0x0 (inexact)
---------------------user-space----------------
<no user backtrace at kernel.function("kmalloc@./include/linux/slab.h:478")>
-----------------------------------------------

kworker/0:1(6489) kernel.function("kmalloc@./include/linux/slab.h:478")
---------------------kernel--------------------
 0xffffffffb40f2ea4 : bio_alloc_bioset+0x1b4/0x240 [kernel]
 0x0 (inexact)
---------------------user-space----------------
<no user backtrace at kernel.function("kmalloc@./include/linux/slab.h:478")>
-----------------------------------------------

WARNING: Missing unwind data for a module, rerun with 'stap -d /lib/x86_64-linux-gnu/libc-2.24.so'
bash(2402) kernel.function("kmalloc@./include/linux/slab.h:478")
---------------------kernel--------------------
 0xffffffffb400aec4 : alloc_pipe_info+0x24/0x160 [kernel]
 0x0 (inexact)
---------------------user-space----------------
 0x7fb3ae26fe07 [/lib/x86_64-linux-gnu/libc-2.24.so+0xdbe07/0x39b000]
-----------------------------------------------

bash(2402) kernel.function("kmalloc@./include/linux/slab.h:478")
---------------------kernel--------------------
 0xffffffffb40208c5 : alloc_fdtable+0x35/0xf0 [kernel]
 0x0 (inexact)
---------------------user-space----------------
 0x7fb3ae24c354 [/lib/x86_64-linux-gnu/libc-2.24.so+0xb8354/0x39b000]
-----------------------------------------------

```  
## Monitoring the VM(qemu) with systemtap
Systemtap can be used as a monitor of remote host. This example use a virtual machine with custom kernel  as a remote host.  
Step:  
1. Run your custom kernel on qemu with specified ssh port
2. Run systemtap with the script on your host

### Custom kernel & qemu
Compile custom kernel on your host( I use the default configure of debian):  
```  
make O=$(BUILD_TREE) bindeb-pkg
```  
After a long wait, several *.deb package will be generated, run `scp` to your VM and `dpkg -i *.deb` on your VM.  
The the follow command line to check your VM & kernel:  
```  
qemu-system-x86_64 -m 2048 -net nic -net user,host=10.0.2.10,hostfwd=tcp::$(SSH_PORT)-:22 -no-reboot -enable-kvm -hda $(IMAGE_OF_YOUR_VM)
```  
```  
ssh -p $(SSH_PORT) root@127.0.0.1
```  
Please make sure ssh can sucessfully connect to VM. 

### Prepare the debug info for systemtap
A self-built systemtap with new version may be necessary, please refer the documentations of source.  
The systemtap need the kernel debug info, which is locate /lib/modules/$(KERNEL_VERSION)/build( I am not sure if it can be specified in build time).So, run:
```  
cp -r $(BUILD_TREE) /lib/modules/`ssh -p $(SSH_PORT) root@127.0.0.1 uname -r`/build
```  
Then, run:
```  
stap --remote=root@127.0.0.1:$(SSH_PORT) $(PATH_TO_YOUR_SCRIPT)
```  

### Note
* Make sure the kernel version of the running and debug_info is the same, specify the custom kernel for qemu is support
* I met the error "frame size larger than *** byte" when the script print backtrace 
