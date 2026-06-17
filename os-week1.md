# Operating System Notes — Week 1

## Process vs Thread
- Process is an independent program in execution with its own private memory space
- Thread is the smallest unit of execution within a process
- Threads share heap memory, code segment, and open resources
- Each thread has its own private stack and program counter
- BookMyShow: One JVM process, Tomcat thread pool of 200 threads handles concurrent /bookSeat requests

## CPU Scheduling

### Round Robin
- Each process gets a fixed time quantum
- If process does not finish within quantum, it moves to back of circular queue
- Prevents starvation — every process gets CPU time
- Load balancer uses same concept — distributes requests across servers in circular order

### FCFS (First Come First Served)
- Processes executed in arrival order
- Simple but causes convoy effect — short processes wait behind long ones

### Priority Scheduling
- Each process assigned a priority number
- CPU executes highest priority process first
- Risk: starvation of low priority processes

## Thrashing
- Occurs when OS does not have enough RAM for active pages of all processes
- OS spends more time swapping pages to disk than executing processes
- CPU utilisation drops, throughput collapses
- BookMyShow risk: Under 500 concurrent requests on small EC2 instance,
  JVM heap exhaustion triggers OS swapping — all 200 threads stall waiting for pages

## Context Switch
- OS saving state of current process/thread and loading state of another
- Involves saving registers, program counter, stack pointer
- Overhead: pure CPU time spent switching rather than executing

# Virtual Memory, Paging, and Thrashing

## Virtual Memory
Every process gets a private virtual address space, mapped by the OS to physical RAM via a page table. This lets multiple processes (Spring Boot, MySQL, Redis) coexist on EC2 without overlapping memory.

## Page Table and Pages
RAM is divided into fixed-size frames (4KB). Virtual memory is divided into matching pages (4KB). The page table maps virtual pages to physical frames per process.

## Page Fault
When a process accesses a page not currently in RAM, the CPU triggers a page fault. The OS pauses the process, fetches the page from disk swap, loads it into a free RAM frame, updates the page table, and resumes the process. RAM access is nanoseconds; disk access is milliseconds — roughly 1,000,000x slower.

## Thrashing
When RAM is overloaded, the OS spends most of its time evicting and fetching pages instead of executing application code. CPU usage looks low because the CPU is idle waiting on disk I/O — this is the signature symptom.

**BookMyShow connection:** During IPL ticket sales, request spikes inflate Spring Boot heap and MySQL buffer pool, filling RAM. Confirm with `free -h` (RAM/swap usage), `vmstat 1` (watch `si`/`so` columns for active swapping), and `top`/`htop` (per-process memory).

## GC-Induced Thrashing
JVM GC pauses the app and scans the full heap (e.g. 512MB) to reclaim memory — normally ~200ms. If the OS has evicted heap pages to disk under memory pressure, GC must fetch every evicted page back from disk during the scan. A 200ms GC cycle can balloon to 10 seconds. The app is fully frozen during this window — no requests processed, no seats locked, no bookings completed.

**BookMyShow connection:** During peak IPL release, this manifests as user-facing timeout errors and seat-locking failures even with bug-free code, because memory — not CPU — is the bottleneck.
