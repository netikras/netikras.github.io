# 16 Killed java $JVM_OPTS -jar app.jar

That is definitely not something you'd like to see in your container logs. And for a good reason. Even worse, if you see thismessage in your logs, you most likely won't see any other error-like errors suggesting what went wrong. And the detective's work begins. 
- Who's the killer?
- What's the motive?
- What's the weapon?
- How could we prevent similar murders in the future?

## The murder mystery

### The weapon
This message, one would not dare to call verbose, is printed when the JVM receives a signal and has no other choice but to die. There is only 1 signal this powerful - it's **SIGKILL**. It's the grim reaper of the processes and full-size snickers can only buy you this much time - not really even enough to say goodbies to all your relatives.

### The killer
The SIGKILL could have several origins:
- someone issued a `kill -9 <pid>` 
- some other process invoked the `signal()` syscall and sent the SIGKILL signal to your JVM
- OOMKiller

### The motive
As in real life, each killer might have a different motive.
- `kill -9` could be a developer, a sysadmin or a malicious user logged in to the system's shell running the command. Why? Ask hm/her. Maybe it was a mistake and a wrong PID was passed to the command. Maybe the JVM appeared to be stuck/frozen and needed to be restarted. Maybe the JVM used up a bit too many resources (especially seen in shared environments) and left no breathing room for other applications. Maybe there was some maintenance happening in the server. There could be other reasons. So log in, run the `last` command and see who was logged in to the system at that time. And give him/her a good lecture on why one should avoid using SIGKILL (kill -9)  (a simple kill or kill -15 should be enough).

- `signal()` that's quite unusual and unlikely to happen. However, unlikely is not impossible. If you have other applications running in that system, chances are some of them might have a bug and pick a wrong pid as the signal receiver. I've only encounered such a case once in my IT career and it was a monitoring daemon to blame.

- OOMK (OutOfMemoryKiller) has only 1 motive - the OS felt threatened it will not have enough memory left to breathe and summoned the OOMK to do some purging. This is the most likely cause of your outages. If you have accesses, youcan confirm this theory by examining the last entries in the `dmesg` output. If you see the 'Killed process 4832 (java)' entry there along with some memory layout tables then you've got your killer.

## OOMKiller - the Grim Reaper from the Kernelland

### The OS (Operating System)
It's the _magic_ that runs in your computer. OS is the interaction layer between the computer's input devices and the output devices. When you type a letter 'a' on your keyboard, it's up to the OS to decide what happens on your screen, CD-ROM and your speakers. One might argue it's up to the application that's open and one would be also correct. Thing is, the term OS is so vast that it includes pretty much all the software: applications, drivers, the terminal, installers and so on. There are certain types of software in your computer that are not a part of the OS (microcode/firmware, BIOS/EFI and similar). The OS is divided into 2 main parts: Kernel and user space.

### The Kernelland
Kernelspace is the Mount Olympus of the OS. It's where the gods live. In a way...
This is the core of the OS. It's a self-sustained system that translates hardware to software and back. It is a complicated layered system that detects and uses your hardware components to do the things your software asks them to do. Kernel is 'the guy' who can give you the resources you need for your applications, like memory, CPU time, network, etc. If the kernel crashes - the whole system crashes.

### The Userland
Userspace is where all the applications are running: MS Word, java, cstrike16.exe, chrome.exe et al. You can list those applications with `ps` or `top`, monitor their activities, even spy on them using `strace` and `ltrace`! This is also where the `root` and the other users live in.

### The problem
The problem is that both the Kernel and the Userspace applications are sharing the same pool of resources. The memory regions are not strictly divided (for the most of it) and it is possible that a memory leak in some userapsce application will consume all the memory. It's also possible (although far, far less likely to happen) that a memory leak in some kernel module (e.g. some driver) will consume the RAM. Either way, both the userspace and the kernelspace will starve. If userspace starves - the applications freeze. But if the kernelspace starves - the whole system crashes. There are methods to manage the memory starvation, like using swap as an expansion of RAM, but these methods do not solve the actual problem.

### Kernel hates fasting
In fact, it hates it so much that it's ready to actually kill if it feels this threat.
As the memory is a shared resouce, kernel kas some means to protect itself from ram starvation. These methods cause problems in userspace, but they keep the system alive. The one that can harm your applications the most it the OutOfMemoryKiller - OOMK. 
The Kernel keeps an eye on the memory usage at all times. If the memory usage grows too high and only a few percent of reusable (reusable > free) memory remain, the kernel feels threatened and invokes the OOMK. The OOMK reviews all the processes in the userspace, inspects their memory usage and applies some formulas to calculate the oomk ratio. It then picks the process with the highest ratio and sends a SIGKILL signal to its pid. The process has no way to talk back to or survive this event and has to die. Usually the the process with the highest oomk ratio is the one which has the highest memory (rss) usage. 

### Pushing another process under the bus
There are knobs and levers to tweak the oomk ratio calculation though: if your java application is spawning multiple worker processes, e.g. soffice processes to convert docx to pdf, you might prefer the OOMK to kill the soffice processes rather than your JVM. It is possible to achieve that by configuring the processes after they are started accordingly. 
`echo -17 /proc/<jvm_pid>/oom_adj` will make your JVM get the lowest possible oomk ratio
`echo 15 /proc/<soffice_pid>/oom_adj` will make your soffice processes get the highest possible oomk ratio.

As a result, when the time comes, the OOMK will be likely to kill the soffice process instead of your java application. 

## Prevention measures
The best way to keep the OOMK sleep tight is to have memory usage under control. However, mistakes happen and memory leaks come as buil-in features. Also, one might not consider some use cases or environment nuances while developing the feature, that might trigger abnormaly large memory usage.

### Java/JVM
#### Memory pools
The JVM is an exceptionally good mechanism to keep your memory usage under control. It has memory pools, most of which can be limited. JVM memory is divided into:
- heap
- non-heap
Heap is the place where all the `new` objects live. It is further divided into multiple sections, but we will not get there in this article. Heap can be hard-capped to never exceed some predefined limit. The standard way to set that limit is to start the JVM with the -Xmx option, e.g.
`java -Xmx4g -jar app.jar`
This will start the JVM and limit its heap usage to 4GB max. Easy, right?

Non-heap is "everything else". Non-heap memory contains things, like
- JVM's code itself - JVM is also an application and it needs needs memory to operate, to run the java bytecode
- loaded classes - ClassLoaders load classes into memory too. As classes is not something applications usually change, they are stored in static memory regions. The more classes you have, the more and the larger static fields they have, the more memory they will consume.
- thread stacks - just like in asm, C and other lower-level languages, process/thread calls are stacked together. This can be seen real nice in threaddumps/stack traces. There you can see what functions are calling what functions, and, in fact, the whole chain of calls. Stack entries are added upon function invocation and removed when either `return` is called or an exception is thrown. Along with the calls' sequence, function parameters are also stored in the stack. All the parameters' values are pused to the stack and then the function is called, and each time you refer to a parameter in that function - you are reading it from the stack space.
- native code
	- native libraries - using JNI/JNA developers can hook write java applications that execute C/C++ code. These sub-applications are compiled into .so/.dll libraries and loaded into the JVM. They are stored in off-heap regions of the memory. The more native libraries you have, the more off-heap memory your JVM will consume. Please mind that your java dependencies might also depend on some native code.
	- native data structures. Native libraries most likely will consume some memory. And unless specifically requested, that memory will be allocated outside of heap. It is only up to that native code to manage its memory. Even the GC cannot touch it. The most familiar example of such data structures is the java's NIO `DirectByteBuffer`. It allocates the buffer directly in the memory, bypassing the GC, which makes that buffer faster, but the larger and the more buffers you have, the more likely you are to consume memory that will not be seen in Heap.

Problem with off-heap memory is that it's difficult to monitor and limit. In a sense, this part of the JVM memory is unbounded and mostly invisible. 

#### Monitoring memory usage
##### Heap

```
bash-4.2# jcmd 16 GC.heap_info
16:
 garbage-first heap   total 5033984K, used 1969903K [0x00000006a6600000, 0x0000000800000000)
  region size 2048K, 273 young (559104K), 10 survivors (20480K)
 Metaspace       used 116770K, capacity 119844K, committed 120228K, reserved 1155072K
  class space    used 13753K, capacity 14850K, committed 14924K, reserved 1048576K
bash-4.2#
```
```
bash-4.2# #jmap -heap 16
bash-4.2# jhsdb jmap --heap --pid 16
Attaching to process ID 16, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 11.0.1+13
using thread-local object allocation.
Garbage-First (G1) GC with 4 thread(s)
Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 5798625280 (5530.0MB)
   NewSize                  = 1363144 (1.2999954223632812MB)
   MaxNewSize               = 3479175168 (3318.0MB)
   OldSize                  = 5452592 (5.1999969482421875MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 2097152 (2.0MB)
Heap Usage:
G1 Heap:
   regions  = 2765
   capacity = 5798625280 (5530.0MB)
   used     = 144095232 (137.419921875MB)
   free     = 5654530048 (5392.580078125MB)
   2.4849895456600364% used
G1 Young Generation:
Eden Space:
   regions  = 24
   capacity = 895483904 (854.0MB)
   used     = 50331648 (48.0MB)
   free     = 845152256 (806.0MB)
   5.620608899297424% used
Survivor Space:
   regions  = 10
   capacity = 20971520 (20.0MB)
   used     = 20971520 (20.0MB)
   free     = 0 (0.0MB)
   100.0% used
G1 Old Generation:
   regions  = 36
   capacity = 4238344192 (4042.0MB)
   used     = 70694912 (67.419921875MB)
   free     = 4167649280 (3974.580078125MB)
   1.6679842126422564% used
```

##### Non-heap
There is a way to monitor off-heap memory regions, although it's most likely not enabled by default. To enable the more in-depth monitoring use this JVM option `-XX:NativeMemoryTracking=summary`. When you start your JVM with this option, you can access its memory statistics by using the JDK utility `jcmd <jvm_pid> VM.native_memory`. Bear in mind, that even these stats do not account for JNI; however, that helps to determine how much memory is wasted by JNI, which is also a good input in the triage!
Enabling NMT will result in a 5-10 percent JVM performance drop.

```
16:
Native Memory Tracking:
Total: reserved=7176125KB, committed=5641761KB									<--- Total amount of memory NMT can account for
-                 Java Heap (reserved=5033984KB, committed=4935680KB)			<--- The heap where your objects live
                            (mmap: reserved=5033984KB, committed=4935680KB)
-                     Class (reserved=1158454KB, committed=124082KB)			<--- Class meta data
                            (classes #22594)									<--- Number of classes loaded
                            (  instance classes #21282, array classes #1312)
                            (malloc=3382KB #68940)
                            (mmap: reserved=1155072KB, committed=120700KB)
                            (  Metadata:   )
                            (    reserved=106496KB, committed=105904KB)
                            (    used=103839KB)
                            (    free=2065KB)
                            (    waste=0KB =0.00%)
                            (  Class space:)
                            (    reserved=1048576KB, committed=14796KB)
                            (    used=13685KB)
                            (    free=1111KB)
                            (    waste=0KB =0.00%)
-                    Thread (reserved=272638KB, committed=40430KB)				<--- Memory used by threads, including thread data structure, resource area, handle area, and so on.
                            (thread #264)
                            (stack: reserved=271344KB, committed=39136KB)		<--- Thread stack. It is marked as committed memory, but it might not be completely committed by the OS.
                            (malloc=947KB #1334)
                            (arena=347KB #527)
-                      Code (reserved=253433KB, committed=87601KB)				<--- Generated code cache (JIT)
                            (malloc=5745KB #20774)
                            (mmap: reserved=247688KB, committed=81856KB)
-                        GC (reserved=252667KB, committed=249019KB)				<--- Data use by the GC, such as card table
                            (malloc=32763KB #72438)
                            (mmap: reserved=219904KB, committed=216256KB)
-                  Compiler (reserved=3407KB, committed=3407KB)					<--- Memory tracking used by the compiler when generating code
                            (malloc=3313KB #1836)
                            (arena=94KB #5)
-                  Internal (reserved=5884KB, committed=5884KB)					<--- Memory that does not fit the previous categories, such as the memory used by the command line parser, JVMTI, properties, and so on.
                            (malloc=5844KB #12801)
                            (mmap: reserved=40KB, committed=40KB)
-                     Other (reserved=3810KB, committed=3810KB)					<--- Unspecified allocations
                            (malloc=3810KB #417)
-                    Symbol (reserved=24261KB, committed=24261KB)				<--- Symbols
                            (malloc=22623KB #314562)
                            (arena=1638KB #1)
-    Native Memory Tracking (reserved=7967KB, committed=7967KB)					<--- Memory used by NMT
                            (malloc=80KB #1113)
                            (tracking overhead=7887KB)
-        Shared class space (reserved=17172KB, committed=17172KB)				<--- Memory mapped to class data sharing archive
                            (mmap: reserved=17172KB, committed=17172KB)
-               Arena Chunk (reserved=141690KB, committed=141690KB)				<--- Allocations for the arena-managed chunk. Arena is a chunk of memory allocated using malloc
                            (malloc=141690KB)
-                   Logging (reserved=5KB, committed=5KB)						<--- Memory used by logging
                            (malloc=5KB #190)
-                 Arguments (reserved=18KB, committed=18KB)						<--- Amount of the native memory allocated for processing arguments
                            (malloc=18KB #494)
-                    Module (reserved=734KB, committed=734KB)					<--- Memory used by modules
                            (malloc=734KB #4871)
```

As you see, JVM's NMT (Native Memory Tracker) gives you quite some visibility on what's happening in the application. Some of those arenas can be capped, others - sort of capped, while others can, in theory, grow as long as there is any ram free in the server/container (or OS limits are reached).

You can limit:
- Heap: 
	- -Xmx=N/-Xms=N (if set, disables the below mechanisms)
	- -XX:MaxRAMPercentage=N (use with UseContainerSupport in containers)
	- -XX:+UseContainerSupport (to make the JVM believe ALL RAM is what cgroups say it is, rather than the OS)
	- -XX:MaxRAMFraction=N (before java10)
	- -XX:MaxRam=N
- Classes:
	- -XX:MaxPermSize=N
	- -XX:MaxMetaspaceSize=N
- JIT Code Cache
	- -XX:ReservedCodeCacheSize=N

You can also limit Threads by limiting their max stack sizes with
- -Xss10m
However, there is no way to limit number of threads the JVM can spawn. While you can limit stack area of a single thread, should the JVM explode in threads count, the memory consumption could still sky-rocket.

Unfortunately, there is no way to limit other areas, nor the JNI-alocated memory blocks.

## Summary
JVM can be killed by the OS without any warnings. As soon as the JVM steps over the memory consumption line, the OS will summon the OOMKiller to kill that JVM (orany other rogue process, for that matter). The OOMK can be "bribed with full-size snickers" [http://www.salenalettera.com/2015/10/how-to-escape-grim-reaper.html] to sacrifice some other process but your JVM. While cheating death is one option, it will only work until your JVM kills the whole system by starving the Kernel from memory.
A better approach is to analyse JVM's memory pools' usage and fix your code accordingly, e.g. if Metaspace is too large -- review your dependencies, perhaps you can get rid of some classes you don't require!; if Heap usage is out of control - consider using Flyweight pattern where applicable, and so on. Another good approach is to cap memory pools so that the JVM does not threaten the OS. Let that JVM die naturally rather than be killed by the OOMK.

## References
- https://www.oracle.com/technical-resources/articles/it-infrastructure/dev-oom-killer.html
- https://lwn.net/Articles/317814/
- https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html
- https://www.baeldung.com/native-memory-tracking-in-jvm
- https://docs.oracle.com/en/java/javase/12/troubleshoot/diagnostic-tools.html#GUID-5EF7BB07-C903-4EBD-A9C2-EC0E44048D37
- https://docs.azul.com/zing/ZingNMT.htm
- https://docs.oracle.com/en/java/javase/11/troubleshoot/diagnostic-tools.html#GUID-5EF7BB07-C903-4EBD-A9C2-EC0E44048D37
- https://www.atamanroman.dev/articles/jvm-memory-settings-container-environment/#fnref:11

> Written with [StackEdit](https://stackedit.io/).
