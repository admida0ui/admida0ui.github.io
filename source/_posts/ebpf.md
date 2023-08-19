---
title: Ping Killer using XDP and eBPF
date: 2023-8-11 8:00:00
tags: Kernel
categories: Work
thumbnail: "https://i.imgur.com/wrGS8cR.png"
---

# PingKiller in eBPF

PingKiller is a simple eBPF program that drops ICMP packets. It is written in C and uses the eBPF library to load the program into the kernel.

## Terms

- eBPF: eBPF is a virtual machine inside the Linux kernel. It can be used to safely execute user-defined programs inside the kernel without the need to change kernel source code or load kernel modules. eBPF programs are written in a restricted C dialect and compiled into bytecode that can be loaded into the kernel with the help of the eBPF library.

- XDP: XDP is a Linux kernel technology that provides a high performance, programmable network data path. XDP is supported by most Linux distributions and is used extensively by cloud providers and other environments that require high performance networking.

- KSP: The Kernel Space Program uses XDP (eXpress Data Path) which is implemented using the eBPF (extended Berkeley Packet Filter) framework in C, with some help from libbpf. It allows us to intercept packets at an early stage in the Linux kernel’s networking stack and perform custom packet processing logic. That is to reach the packet header and decide what to do with it, ICMP packets are dropped in this case, other packets are passed to the next stage in the networking stack.

- USP: The User Space Program is a Golang application that interacts with the XDP eBPF program, providing a user-friendly interface to monitor the packet drop behavior and visualize performance statistics, it also allows us to load the KSP eBPF program into the kernel and attach it to the relevant tracepoint/probe (interface), all with the help of Cilium.

- Cilium: Cilium is an open-source project that provides networking and security capabilities powered by eBPF. Their documentation and codebase offer in-depth insights into eBPF and its applications.

- BCC: Out-of-our-scope, BCC is a collection of powerful command-line tools and libraries that leverage eBPF for various tracing and performance analysis tasks.

## Learning Section

We are currently gathering resources, watching videos, and reading documentations of the Linux Kernel, eBPF use-cases, and Cilium. We will keep updating this learn more section as we progress.

- The Linux Kernel Documentation: The Linux kernel documentation includes a comprehensive section on eBPF and XDP, covering various aspects, including API references, usage examples, and implementation details. [Link](http://www.kernel.org/doc/html/latest/bpf)

- The Cilium Project: Open-sourced platform Cilium was created by Isovalent Inc. to help address the rising security and networking challenges emerging as cloud-native environments, including Kubernetes, grow in scale and complexity. [Cilium](https://cilium.io/)

- eBPF Community: Community-driven website dedicated to providing resources, tutorials, and news about eBPF. It features articles, case studies, and a curated list of tools and libraries related to eBPF. [Link](https://ebpf.io/)

## Environment Set-up

We are using Ubuntu LTS and Linux Manjaro based on Linux kernel 6.1.31 as our development environment, basically eBPF is supported by all Linux Kernels > version 4.

### What we will need

- Clang: Clang is a C, C++, Objective-C, or Objective-C++ compiler that is compiled in C++ based on LLVM. It is designed to be compatible with the GNU Compiler Collection (GCC), supporting a wide variety of platforms. 
- LLVM: The LLVM Project is a collection of modular and reusable compiler and toolchain technologies. Despite its name, LLVM has little to do with traditional virtual machines. The name "LLVM" itself is not an acronym; it is the full name of the project. 
- bpftool: bpftool is a command-line utility in Linux that is used to manage and manipulate BPF (Berkeley Packet Filter) programs and maps. We are going to install this from the source.
- Golang: Go is an open-source programming language that makes it easy to build simple, reliable, and efficient software.

### Installation Commands

#### Ubuntu / Debian-based

1. Install Clang and LLVM

```bash
sudo apt update
sudo apt install clang llvm
```

2. Install bpftool

```bash
git clone --recurse-submodules https://github.com/libbpf/bpftool.git
cd src
make
sudo make install
```

3. Install Golang

```bash
sudo apt install golang
```

#### Manjaro / Arch-based

1. Install Clang and LLVM

```bash
sudo pacman -S clang llvm
```

2. Install bpftool

```bash
git clone --recurse-submodules https://github.com/libbpf/bpftool.git
cd src
make
sudo make install
```

3. Install Golang

```bash
sudo pacman -S go
```

## PingKiller: Our first eBPF program

### What is PingKiller?

PingKiller is a simple eBPF program that drops ICMP packets. It is written in C and uses the eBPF library to load the program into the kernel.

### Kernel Space Program

The kernel space program that uses XDP to drop all ICMP packets of a given network interface, and will collect statistics and measure the processing time for dropped and passed packets using eBPF perf event mechanism. The program will run in kernel space, and will expose the data to userspace using eBPF maps.

- Clone libbpf and change into the directory

```bash
git clone https://github.com/libbpf/libbpf.git
cd src
make
```

- Get back to project root folder and create a new directory where the kernel space program will live `ksp` and then create a subdirectory named `bpf` and change into it.

- Create a file `dicmp_kern.c`, this file will contain our XDP eBPF logic.

```c
#include <linux/bpf.h>
#include <bpf_helpers.h>
```

- The `bpf_helpers.h` header file contains a set of helper functions that are used to interact with the eBPF runtime. The `bpf.h` header file contains definitions for eBPF related data structures and constants. These files are part of the Linux kernel source code and are located in the libbpf source code directory.

- Next, we define the necessary data structures and maps for XDP, `perf_trace_event` that will represent the perf event data, `BPF_MAP_TYPE_PERF_EVENT_ARRAY` map to store the perf events, as follows:

```c
struct perf_trace_event {
    __u64 timestamp;
    __u32 processing_time_ns;
    __u8 type;
};

#define TYPE_ENTER 1
#define TYPE_DROP 2
#define TYPE_PASS 3

// these TYPE_* values are used in the userspace program later

struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(__u32));
    __uint(value_size, sizeof(__u32));
} output_map SEC(".maps");
```

- XDP Program Function

```c
SEC("xdp")
int xdp_dicmp(struct xdp_md *ctx)
{
    // logic goes here
}
```

- The `xdp_dicmp` function is the entry point of our XDP program, it will be called for each packet that arrives at the network interface. The `ctx` parameter is a pointer to the `xdp_md` data structure, which contains information about the packet and the network interface.

- Now, onto handling perf events, drop and pass packets, and collect statistics.

Inside the `xdp_dicmp` function, we will first create a `perf_trace_event` object and initialize it with the current timestamp and the packet processing time, then we will store the event in the `output_map` map, as follows:

```c
struct perf_trace_event e = {};
// Perf event for entering xdp program
e.timestamp = bpf_ktime_get_ns();
e.type = TYPE_ENTER;
e.processing_time_ns = 0;
bpf_perf_event_output(ctx, &output_map, BPF_F_CURRENT_CPU, &e, sizeof(e));
```

- Next, we will check if the packet is an ICMP packet, if it is, we will drop it and store a perf event with the type `TYPE_DROP`, otherwise, we will pass the packet and store a perf event with the type `TYPE_PASS`, as follows:

- We start by getting a pointer to the packet data using the `data` field of the `xdp_md` data structure, then we get the packet size using the `data_end` field, and finally, we get the packet header using the `data` field.

```c
void *data = (void *)(long)ctx->data; // start of packet data
void *data_end = (void *)(long)ctx->data_end; // end of packet data

struct ethhdr *eth = data; // ethernet header
if (data + sizeof(struct ethhdr) > data_end) // check if packet has ethernet header
{
    e.type = TYPE_DROP;
    __u64 ts = bpf_ktime_get_ns();
    e.processing_time_ns = ts - e.timestamp;
    e.timestamp = ts;
    bpf_perf_event_output(ctx, &output_map, BPF_F_CURRENT_CPU, &e, sizeof(e));
    return XDP_DROP;
}

if (bpf_ntohs(eth->h_proto) != ETH_P_IP) // check if ethernet header is ipv4
{
    e.type = TYPE_PASS;
    __u64 ts = bpf_ktime_get_ns();
    e.processing_time_ns = ts - e.timestamp;
    e.timestamp = ts;
    bpf_perf_event_output(ctx, &output_map, BPF_F_CURRENT_CPU, &e, sizeof(e));
    return XDP_PASS;
}

struct iphdr *iph = data + sizeof(struct ethhdr); // ip header
if (data + sizeof(struct ethhdr) + sizeof(struct iphdr) > data_end) // check if packet has ip header
{
    e.type = TYPE_DROP;
    __u64 ts = bpf_ktime_get_ns();
    e.processing_time_ns = ts - e.timestamp;
    e.timestamp = ts;
    bpf_perf_event_output(ctx, &output_map, BPF_F_CURRENT_CPU, &e, sizeof(e));
    return XDP_DROP;    
}
// This is a ping packet
if (iph->protocol == IPPROTO_ICMP) { // check if ip header is icmp
    bpf_printk("Got ICMP packet\n"); // print to kernel log
    e.type = TYPE_DROP;
    __u64 ts = bpf_ktime_get_ns();
    e.processing_time_ns = ts - e.timestamp;
    e.timestamp = ts;
    bpf_perf_event_output(ctx, &output_map, BPF_F_CURRENT_CPU, &e, sizeof(e));
    return XDP_DROP; 
}

if (iph->protocol == IPPROTO_TCP) // check if ip header is tcp
    bpf_printk("Got TCP packet\n"); // print to kernel log
```

- The drop and pass logic is the same, we just change the `e.type` value and the return value of the function, we also calculate the packet processing time and store it in the `e.processing_time_ns` field.

- If there is no ethernet header, we will drop the packet and store a perf event with the type `TYPE_DROP`, otherwise, we will check if the ethernet header is an IPv4 header, if it is not, we will pass the packet and store a perf event with the type `TYPE_PASS`, otherwise, we will check if the IP header is an ICMP header, if it is, we will drop the packet and store a perf event with the type `TYPE_DROP`, otherwise, we will check if the IP header is a TCP header, if it is, we will print a message to the kernel log.

- Finally, we will pass the other packets and store a perf event with the type `TYPE_PASS`, as follows:

```c
e.type = TYPE_PASS;
__u64 ts = bpf_ktime_get_ns();
e.processing_time_ns = ts - e.timestamp;
e.timestamp = ts;
bpf_perf_event_output(ctx, &output_map, BPF_F_CURRENT_CPU, &e, sizeof(e));
return XDP_PASS;
```

- The `bpf_printk` function is used to print messages to the kernel log, it is similar to the `printf` function in C, but it is only used for debugging purposes, it is not recommended to use it in production.

- Now, we are done with the function, do not forget to add the license:

```c
char _license[] SEC("license") = "GPL";
```

- And that is it, we are done with the XDP program, now we will compile it using the following command:

```bash
clang -S \
    -g \
    -target bpf \
    -I../../libbpf/src\
    -Wall \
    -Werror \
    -O2 -emit-llvm -c -o dicmp_kern.ll dicmp_kern.c
```

- Which will generate the LLVM IR file `dicmp_kern.ll`, then we will use the `llc` tool to compile the LLVM IR file to BPF bytecode, as follows:

```bash
    llc -march=bpf -filetype=obj -O2 -o dicmp_kern.o dicmp_kern.ll
```

Notice, that this compilation step can take part in the userspace program since we can compile c programs to bytecode in golang, we will check that next time.

### User Space Program

- Here, we will write the Golang logic that interacts with the XDP program in order to collect metrics

- First, the USP will load the XDP program, then it will create a perf event map, and finally, it will read the perf events from the map and print them to the terminal.

- We start by importing the required packages:

```go
package main
import (
        "encoding/binary"
        "fmt"
        "net"
        "os"
        "os/signal"
        "syscall"
        "github.com/cilium/ebpf"
        "github.com/cilium/ebpf/link"
        "github.com/cilium/ebpf/perf"
)
```

- Then, we will define the perf event data structure:

```go
const (
        TYPE_ENTER = 1
        TYPE_DROP  = 2
        TYPE_PASS  = 3
)
type event struct {
        TimeSinceBoot  uint64
        ProcessingTime uint32
        Type           uint8
}
const ringBufferSize = 128 // size of ring buffer used to calculate average processing times
type ringBuffer struct {
        data   [ringBufferSize]uint32
        start  int
        pointer int
        filled bool
}
```

- The `ringBuffer` data structure is used to calculate the average processing time of the last `ringBufferSize` packets, it is a ring buffer that stores the processing time of the last `ringBufferSize` packets, the `start` field is used to keep track of the oldest packet in the buffer, the `pointer` field is used to keep track of the next empty slot in the buffer, and the `filled` field is used to check if the buffer is full or not.

- Implemented below, are the functions to sum and average the processing times in the ring buffer:

```go
func (rb *ringBuffer) add(val uint32) {
        if rb.pointer < ringBufferSize {
                rb.pointer++
        } else {
                rb.filled = true
                rb.pointer= 1
        }
        rb.data[rb.pointer-1] = val
}
func (rb *ringBuffer) avg() float32 {
        if rb.pointer == 0 {
                return 0
        }
        sum := uint32(0)
        for _, val := range rb.data {
                sum += uint32(val)
        }
        if rb.filled {
                return float32(sum) / float32(ringBufferSize)
        }
        return float32(sum) / float32(rb.pointer)
}
```

- Now, we will define the `main` function:

```go
func main() {
	spec, err := ebpf.LoadCollectionSpec("../ksp/bpf/dicmp_kern.o")
	if err != nil {
		panic(err)
	}
	coll, err := ebpf.NewCollection(spec)
	if err != nil {
		panic(fmt.Sprintf("Failed to create new collection: %v\n", err))
	}
	defer coll.Close()
	prog := coll.Programs["xdp_dicmp"]
	if prog == nil {
		panic("No program named 'xdp_dicmp' found in collection")
	}
	iface := "wlo1"
	if iface == "" {
		panic("No interface specified. Please set the INTERFACE environment variable to the name of the interface to be use")
	}
	iface_idx, err := net.InterfaceByName(iface)
	if err != nil {
		panic(fmt.Sprintf("Failed to get interface %s: %v\n", iface, err))
	}
	opts := link.XDPOptions{
		Program:   prog,
		Interface: iface_idx.Index,
		// Flags is one of XDPAttachFlags (optional).
	}
	lnk, err := link.AttachXDP(opts)
	if err != nil {
		panic(err)
	}
	defer lnk.Close()
	fmt.Println("Successfully loaded and attached BPF program.")
	// handle perf events
	outputMap, ok := coll.Maps["output_map"]
	if !ok {
		panic("No map named 'output_map' found in collection")
	}
	perfEvent, err := perf.NewReader(outputMap, 4096)
	if err != nil {
		panic(fmt.Sprintf("Failed to create perf event reader: %v\n", err))
	}
	defer perfEvent.Close()
	buckets := map[uint8]uint32{
		TYPE_ENTER: 0, // bpf program entered
		TYPE_DROP:  0, // bpf program dropped
		TYPE_PASS:  0, // bpf program passed
	}
	processingTimePassed := &ringBuffer{}
	processingTimeDropped := &ringBuffer{}
	go func() {
		// var event event
		for {
			record, err := perfEvent.Read()
			if err != nil {
				fmt.Println(err)
				continue
			}
			var e event
			if len(record.RawSample) < 12 {
				fmt.Println("Invalid sample size")
				continue
			}
			// time since boot in the first 8 bytes
			e.TimeSinceBoot = binary.LittleEndian.Uint64(record.RawSample[:8])
			// processing time in the next 4 bytes
			e.ProcessingTime = binary.LittleEndian.Uint32(record.RawSample[8:12])
			// type in the last byte
			e.Type = uint8(record.RawSample[12])
			buckets[e.Type]++
			if e.Type == TYPE_ENTER {
				continue
			}
			if e.Type == TYPE_DROP {
				processingTimeDropped.add(e.ProcessingTime)
			} else if e.Type == TYPE_PASS {
				processingTimePassed.add(e.ProcessingTime)
			}
			fmt.Print("\033[H\033[2J")
			fmt.Printf("total: %d. passed: %d. dropped: %d. passed processing time avg (ns): %f. dropped processing time avg (ns): %f\n", buckets[TYPE_ENTER], buckets[TYPE_PASS], buckets[TYPE_DROP], processingTimePassed.avg(), processingTimeDropped.avg())
		}
	}()
	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt, syscall.SIGTERM)
	<-c
}
```

- We start by loading the eBPF collection from the compiled object file, then we get the `xdp_dicmp` program from the collection, and we attach it to the specified interface.

- Then, we get the `output_map` map from the collection, and we create a perf event reader for it.

- We create two ring buffers, one for the processing time of the dropped packets, and one for the processing time of the passed packets.

- Then, we start a goroutine to read the perf events from the `output_map` map, and we print the statistics every time we receive a perf event.

- Finally, we wait for an interrupt signal to exit the program.

- Now, we can run the program:

```bash
go mod init dicmp
go mod tidy
CGO_ENABLED=0 go build . 
sudo ./dicmp
```

**DEMO**

- We will run the program, and we will ping the interface to generate ICMP packets:

<video src="https://i.imgur.com/dxDPlJb.mp4" controls="controls" style="max-width: 730px;">
</video>

# Next Steps

- Learning how to deal with eBPF map entries lookups and updates
- Learning how to control the kernel space program from the user space program using eBPF maps
- Dealing with syscalls in eBPF programs

Shout out to Peter McConnell for his great post on Medium about XDP implementation.