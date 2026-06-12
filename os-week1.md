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