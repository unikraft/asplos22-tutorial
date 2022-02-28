---
title: High performance applications
date: 2019-02-11T19:27:37+10:00
weight: 6
summary: We explore how we can create high performance application over Unikraft. Expected time 60mn.
---
## Requirements

In this session we will be making a high performance UDP packet generation application. We will be using ```tshark``` and ```tcpdump``` to analyse and debug our implementation. The VM that we provide you access to already has these packages installed.
```
$ sudo apt-get -y install qemu-kvm qemu-system-x86 sgabios socat bridge-utils ifupdown tshark tcpdump
```

We also need to enable the ``br_netfilter`` kernel module.
```
sudo modprobe br_netfilter
```
## Overview

We now turn our attention to what unikernels excel at, performance. We will introduce to you how to develop highly specialized and performance-optimized unikernels with Unikraft. So far, we have focused on applications and POSIX compatibility; where it is important to provide the same set of APIs and system calls that your application uses on its original environment (i.e., as a Linux user space application). We achieve this by stacking multiple micro-libraries which then assemble together to form a combination of various necessary “higher-level” APIs.

In this session we will be looking into networking applications, but the idea presented today is general and can be used to speedup many applications, such as memory disaggregation, filesystem operations implementation and many more. In the context of network-based applications, we would typically develop network functionality based on the ```sockets API```. This requires the following library stack being available within Unikraft for the socket (and friends) API to interface with the virtual Network Interface Card (vNIC):

```
 .-------------------------.
(     Socket application    )
 '-------------------------'
              |
              V
+---------------------------+
|         libvfscore        |
+---------------------------+
+---------------------------+
|          liblwip          |
+---------------------------+
+---------------------------+
|        libuknetdev        |
+---------------------------+
+---------------------------+
|        libkvmplat         |
+---------------------------+
              |
              V
 .-------------------------.
( Virtual Network Interface )
 '-------------------------'
```

Especially the Virtual File System (VFS) layer (provided by libvfscore) and the TCP/IP network stack (provided by liblwip) are complex subsystems which can introduce additional overheard.

For high-performance Network Functions (NFs), it is often more efficient to bypass any OS component and interact with the driver or hardware as directly; cutting out any indirection. A well known framework used to bypass kernel netowrking is Intel DPDK which operates network card drivers in Linux user space using `vfio` or `uio`. It operates in user space in order to avoid interactions with the kernel which comes with performance penalties resulting from additional permission checks. Despite this advantage in performance, you still need to maintain and operate a complete Linux environment in production deployments. Further, DPDK requires at least one dedicated core for pooling. 

Back to our unikernel, we now try to increase the performance by reducing the number of indirections. In the case with Unikraft, we can configure the libraries to be minimal and can, similar to Intel DPDK, directly develop our NF on top of network drivers API.

In this scenario, our library stack does look like the following:

```
 .-------------------------.
(    High performance NF    )
 '-------------------------'
              |
              V
+---------------------------+
|        libuknetdev        |
+---------------------------+
+---------------------------+
|        libkvmplat         |
+---------------------------+
              |
              V
 .-------------------------.
( Virtual Network Interface )
 '-------------------------'
```


In the following tutorial, you will develop a simple, high performance network packet generator. This tutorial will guide you through various options and possibilities which can help you during the development of more complex NFs with Unikraft.

## Work Items

### 01. Getting Started

For this session, a template has been provided which contains some basic building blocks (like crafting a IPv4/UDP packet) for our high performance NF.
Start by making a copy of it:

```sh
$ cp -a sol/pktgen path/to/your/copy
```

Go into your copy and initialize it with `kraft`:

```sh
$ cd path/to/your/copy
$ kraft list update
$ kraft list pull
$ kraft configure
$ kraft build
```

Check if the image runs and prints the Unikraft banner:

```sh
$ kraft run
```

### 02. Bring Up a Network Interface

We can directly interact with network device drivers which are typically provided by each platform using Unikraft's internal [`uknetdev`](https://github.com/unikraft/unikraft/tree/staging/lib/uknetdev) API.
First, make sure that we state a dependency of our application to `libuknetdev`.
To do this, open `Config.uk` and place the following dependency accordingly in the file (if not already there): `depends on LIBUKNETDEV`.

This dependency gives us access to the `<uk/netdev.h>` and `<uk/netbuf.h>` headers which are available within the `libuknetdev` library:

```sh
$ ls [PATH-TO-UNIKRAFT]/lib/uknetdev/include/uk/
```

As described in `<uk/netdev.h>`, bringing up a network interface means transition it through configuration states before we can use the interface for sending packets. We'll be
editing the `main` function from `main.c`. Read the description of any function you encounter from [`netdev.h`](https://github.com/unikraft/unikraft/blob/staging/lib/uknetdev/include/uk/netdev.h).

1. Check that the platform detected network interfaces.
   [`uk_netdev_count()`](https://github.com/unikraft/unikraft/blob/staging/lib/uknetdev/include/uk/netdev.h#L80) should tell us how many interfaces are available.
   Please note that you should also check that the network driver is enabled in the platform configuration.
   For this session we are interested in `virtio-net` within `KVM guest`.
   ```C
   if (uk_netdev_count() <= 0) {
	uk_pr_debug("No available devices, maybe you didn't run \
			with kraft run -b asplosbr0\n");
   	return -1;
   }
   ```

2. Retrieve `struct uk_netdev *` for further API interaction from a netdev number (they are just incrementally going upwards).
   We take the first interface, so our device number should be `0`. We'll be using [`uk_netdev_get`](https://github.com/unikraft/unikraft/blob/staging/lib/uknetdev/include/uk/netdev.h#L93)
   ```C
   struct uk_netdev *dev = uk_netdev_get(0);
   ```

3. Configure the device, which essentially indicate how many receive and transmit queues the device should provide.
   In SMP scenarios, you typically configure as many queues as CPU-cores or handler threads you have been allocated.

   **Note:**  Not every driver or network card can support multiple queues.

   There is a query interface where you can check for queues are supported by your device.
   For simplicity, we are going to configure just one queue for each direction.
   This is supported by all drivers. We'll be using [`uk_netdev_configure`](https://github.com/unikraft/unikraft/blob/staging/lib/uknetdev/include/uk/netdev.h#L177).
   Although we are going to send packets only, we still have to also configure one receive queue (zero transmit or receive queues is not possible with our virtio driver):

   ```C
   /* Device configuration */
   struct uk_netdev_conf dev_conf = {
     	.nb_rx_queues = 1,
     	.nb_tx_queues = 1
   };

   ret = uk_netdev_configure(dev, &dev_conf);
   if (ret < 0) {
     	uk_pr_debug("Couldn't configure\n");
     	return -1;
   }
   ```

4. Configure the transmit queue `0` and the receive queue `0`. We use [`uk_netdev_rxq_configure`](https://github.com/unikraft/unikraft/blob/staging/lib/uknetdev/include/uk/netdev.h#L224)
   and [`uk_netdev_txq_configure`](https://github.com/unikraft/unikraft/blob/staging/lib/uknetdev/include/uk/netdev.h#L268).
   This step allows us to specify the size for each queue and which allocators should be used for internal queue descriptors and receive buffers.
   We will take the default allocator for those items.
   You can define a dummy allocation function for the receive buffers, because we are not interested in receiving for now.
   We will also let the driver to choose an optimal queue size for us.    You can hand-over `0`.

   ```C
   /* Dummy receive buffer allocation callback that is called by the driver */
   static uint16_t dummy_alloc_rxpkts(void *argp __unused,
   				struct uk_netbuf *pkts[] __unused,
   				uint16_t count __unused)
   {
   	return 0;
   }

   /* In at the end of main */
   /* Receive queue configuration */
   struct uk_netdev_rxqueue_conf rxq_conf = {
      	.a = uk_alloc_get_default(),
      	.alloc_rxpkts = dummy_alloc_rxpkts
   };

   /* Transmit queue configuration */
   struct uk_netdev_txqueue_conf txq_conf = {
      	.a = uk_alloc_get_default()
   };
   
   /* Configure the RX queue */
   ret = uk_netdev_rxq_configure(dev, 0, 0, &rxq_conf);
   if (ret < 0) {
     	uk_pr_debug("Failed to configure rx queue of netdev\n");
     	return -1;
   }
   
   /* Configure the TX queue */
   ret = uk_netdev_txq_configure(dev, 0, 0, &txq_conf);
   if (ret < 0) {
     	uk_pr_debug("Failed to configure tx queue of netdev\n");
     	return -1;
   }
   ```

5. Start the network interface using [`uk_netdev_start`](https://github.com/unikraft/unikraft/blob/staging/lib/uknetdev/include/uk/netdev.h#L286)
   If successful, the device is now ready to process network traffic.
   You will now have the ability to also enable interrupt mode for each queue individually and change the promiscuous setting for the interface.
   Because we will operate in _polling mode_ to achieve the highest possible performance, we should not change any interrupt settings.
   We also do not need promiscuous mode because we will put the device's hardware address as sender address into our generated traffic.
   It is probably a good moment to print on the console this mac address and store it for later.
   We will need it to craft our first network packet.
   ```C
   ret = uk_netdev_start(dev);
   if (ret < 0) {
   	uk_pr_debug("Failed to start netdev\n");
	return -1;
   }
   ```

For easier development of this state transition, we recommend to enable all kernel message types and optionally debug message (go to `Library Configuration` -> `ukbedug`).
Many of these steps should produce some kernel output so that you can quicker see if something got misconfigured.

In order to test your code you should run the guest with one interface attached.
For this purpose we need to create a network bridge on your Linux host first (we just need to do this once):

```sh
# Ensure you have permissions to change stp
sudo sysctl -w net.bridge.bridge-nf-call-arptables=0

# Create bridge 'asplosbr0'
brctl addbr asplosbr0
brctl setfd asplosbr0 0
brctl sethello asplosbr0 0
brctl stp asplosbr0 off
ifconfig asplosbr0 0.0.0.0 up

# Disable packet filtering on bridge interfaces
sudo su -
echo 0 > /proc/sys/net/bridge/bridge-nf-call-arptables
echo 0 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 0 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
exit
```

As soon as your unikernel image builds, the guest can then be started with:

```sh
$ kraft run -b asplosbr0
```

### 03. Say Hello on the Wire

In this chapter we are going to send out our first packet.
We provide you a function through the header `"genpkt.h"` which generates an Ethernet-IPv4-UDP frame with a dummy payload for a given size: `genpkt_udp4()`.
In the same header we also provide you the short-hand version `genpkt()` which has some parameters, like the IP addresses, pre-filled.

The only items that the function still wants to know from you are the following:

* `a`: Allocator where the packet should be allocated from.
  Use [`uk_alloc_get_default()`](https://github.com/unikraft/unikraft/blob/64870e20031aad230973b205ba80ff70a454c924/lib/ukalloc/include/uk/alloc.h#L135-L138) for now.

* `bufalign`: An alignment requirement for the packet buffer containing the packet.
   Some network drivers require specific alignments.
   You find this value after querying the device with [`uk_netdev_info_get()`](https://github.com/unikraft/unikraft/blob/104fed122c41cbdedb03b701c19c38d4974cca34/lib/uknetdev/netdev.c#L217-L236) on [`struct uk_netdev_info`](https://github.com/unikraft/unikraft/blob/104fed122c41cbdedb03b701c19c38d4974cca34/lib/uknetdev/include/uk/netdev_core.h#L141-L150) as `ioalign`.

* `headroom`: Reserved bytes at the beginning of the packet buffer and before the packet data starts.
  Some drivers require this in order to do another encapsulation on transmit (like virtio).
  You find this value on the `struct uk_netdev_info` as `nb_encap_tx`.

* `pktlen`: The size of the Ethernet frame (excluding CRC, FCS, SFD, and preamble) that should be generated.
  According to the [Ethernet specification](https://www.ietf.org/rfc/rfc1042.txt) the smallest packet size can be created with `60` and the biggest with `1518`.
  The most interesting are minimum sized packets because those stress software and hardware components the most.
  For each packet, the header needs to be parsed and the packet needs to get forwarded to the next processing layer of the stack.
  As smaller the packets are, the more load with parsing and handling packet buffers occurs. So, please take `60` ;-)

* `mac_src`: The hardware address of our interface where we are going to send the packet out.

The function returns you a `netbuf` that can be send out with [`uk_netdev_tx_one()`](https://github.com/unikraft/unikraft/blob/4e54f09a3930f0482a90903a5750c036346c7c06/lib/uknetdev/include/uk/netdev.h#L471-L508).
Please check the resulting status code for success and free the packet with [`uk_netbuf_free()`](https://github.com/unikraft/unikraft/blob/4e54f09a3930f0482a90903a5750c036346c7c06/lib/uknetdev/netbuf.c#L220-L254) in case of failures.
The driver will do the free operation itself only if a packet got correctly enqueued to the device and sent.
In such a case, you aren't allowed to touch this packet anymore after sending;
so your transmit code should look like this:

```c
/* <...> */
int status;
struct uk_netdev_info dev_info;

/* Get device info */	
uk_netdev_info_get(dev, &dev_info);

/* Get the mac */
struct uk_hwaddr *hw = uk_netdev_hwaddr_get(dev);

/* Generate UDP packet */
struct uk_netbuf *pkt = genpkt(uk_alloc_get_default(), dev_info.ioalign, dev_info.nb_encap_tx, 60, hw);

/* Send the packet */
status = uk_netdev_tx_one(dev, 0, pkt);
if (!uk_netdev_status_successful(status)) {
	uk_pr_err("netdev%u: Failed to send packet %p\n",
		  uk_netdev_id_get(dev), pkt);
	uk_netbuf_free(pkt);
}

/* Do not touch pkt here anymore, on success uk_netdev_tx_one clears the memory  */
/* <...> */
```

In order to see if everything works, attach `tshark` or `tcpdump` on your Linux host to `asplosbr0` on a second terminal:

```sh
$ tshark -i asplosbr0
```

Whenever you launch your unikernel, you should be able to see the UDP packet:

```
    3 1.050213439 192.168.128.1 → 192.168.128.254 UDP 60 5001 → 5001 Len=18
```

You may not see the exact source IP or destination, but the UDP packet should be there along some IPV6 packets.

### 04. Don't Stop

Now let us send as much as we can with the current implementation.
You can simply loop forever over packet generation and sending.
You may notice that we get too many messages on the console that slow us down.
Try disabling debug messages and all kernel messages except the critical ones from ``ukdebug`` config options. (``kraft menuconfig``)

**Note**: You should be able to terminate your unikernel with `CTRL`+`C` when you launched it with `kraft` or `qemu-guest`.

### 05. Measuring the TX throughput

It is now interesting to understand at which speed we are generating.
For this purpose we prepared a little function in `"netspeed.h"` that computes the packet rate (packets/sec) and current bandwidth (MBit/s): `print_netspeed()`.

Declare before your loop the following two variables:

```c
uint64_t total_nb_pkts = 0; /* total number of pkts successfully sent */
uint64_t total_nb_bytes = 0; /* total number of bytes successfully sent */
```

Whenever a packet was successfully sent, we will simply increment `total_nb_pkts` and add the sent bytes `total_nb_bytes` counters.
In order to see a bandwidth computation that is comparable with physical Ethernet speeds, we have to additionally add the number of bytes (=`24`) for CRC, FCS, SFD, and preamble to each accounted packet size:

```c
	status = uk_netdev_tx_one(netif, 0, pkt);
	if (uk_netdev_status_successful(status)) {
		/* success */
		total_nb_pkts += 1;
		total_nb_bytes += 60 /* pktlen */ + 24;
	} else {
		/* failed */
		uk_netbuf_free(pkt);
	}
```

By having this instrumentation, we could now just print the packet rate and bandwidth at every loop iteration with:

```c
	print_netspeed(total_nb_pkts, total_nb_bytes);
```

The problem is that printing is extremely expensive.
This is because it happens synchronously in Unikraft, so the CPU can not do anything else while waiting for the console to finish its operation.
Additionally, for computation, the clock is accessed to measure a time delta, which is also an expensive operation.
In general, this means that we do not want this function to be called very often.
The cheapest option is to call this print function every `n`th sent packet.
We could do a cheap modulo operation by using a bitmask, for example:

```c
		if ((total_nb_pkts & 0x3fffff) == 0x0) {
			print_netspeed(total_nb_pkts, total_nb_bytes);
		}
```

You are able to adopt the mask `0x3fffff` in order to make printing more often or less often.

* Faster: `0x1fffff`, `0x0fffff`, `0x07ffff`, `0x03ffff`, `0x01ffff`, ...
* Slower: `0x7fffff`, `0xffffff`, `0x1ffffff`, `0x3ffffff`, `0x7ffffff`, ...

Another option is to use another counter variable that is reset as soon as we print:

```c
		if (count == 1000) {
			print_netspeed(total_nb_pkts, total_nb_bytes);
			count = 0;
		}
```

Instrument your code with the two statistics variables and implement one of the mentioned printing mechanisms.
We should roughly print not faster than every `2` seconds, ideal are roughly `5`-`10` second intervals.
Remember, if your rate goes up or down with one of the following experiments, you may need to revisit your chosen value and adopt this `n`th packet parameter again.

### 06. Optimizing for even more performance

Now, we have can go through some options to play around with.

Our overall goal is to get the packet rate of our packer generator as high as possible.
Note your rate and bandwidth before and, after each of the optimization steps because, make sure that you do the steps non-destructive and keep the code of each step.

1. **Try compiler options**: Enable *Link Time Optimizations* (LTO) and *Dead Code Elimitation* (DCE) within `Build Options` of the `menuconfig`.
   The compiler reconsiders a second time optimizations like function inlining while linking the final binary;
   actually over the whole code base at once again.
   These optimizations can have some visible effect on your packet rate.
   Try it out!

2. **Don't waste packets**: An obvious idea might be to keep packets which have failed to send.
   We could save on packet generation time if we wouldn't free them.
   We retry sending a packet until it finally leaves.
   Our assumption is that the reason why it fails is that the transmit queue is full.
   This approach can have positive but also likely negative effects.
   The reason might be that some drivers may query their device more often to confirm that there is really no space left.
   This causes the device to be busy answering instead of doing some actual work.
   Try it out!

3. **Copy instead of create**: Depending on how expensive the packet generation function is (e.g., because of an extra step computing a checksum), it could be cheaper to do a `memcpy` operation from a primordial packet buffer instead.
   This means that we would run `genpkt_udp4()` just once and use as source for all cloned packets that are going to get transmitted.
   We provide you such an extra routine with `"netbuf.h"`: `uk_netbuf_dup_single()` duplicates a given `netbuf` packet with `memcpy`.
   Like `genpkt_udp4()`, it also needs the same extra information like `bufalign`, `headroom` for doing the allocation of the duplicate.
   Try it out!

4. **Use a memory pool allocator**: This is usually a very promising optimization.
   Instead of using a general purpose allocator you can ensure that all `malloc` and `free` operations are satisfied within O(1).
   If we deal with rates at maximum speed you want to have every job done as fast as possible.
   A pool is basically a list of pre-allocated objects that have all the same size and an alignment property (if given).
   On `malloc`, an object is returned out of this list;
   on `free`, the object gets back to the free list.
   For trying it out, continue with ``06.1`` and come back to point 5 of this list afterwards.

5. **Zero-copy with refcounting**: Instead of all the optimization ahead, we could also simply increase the `netbuf` reference counter before sending.
   This avoids that the packet being `free`'d after sending and we would not need to allocate, copy, or generate a packet over and over again.
   Every `free` operation will decrease the refcount until the reference counter becomes zero.
   At this point the netbuf is really `free`'d.
   Unfortunately, we do not support this mode with network drivers which modify the packet for the transmission, like virtio-net does.
   Unfortunately, it is not an option for virtio-net at the moment.
   The transmit function will return an error.

Besides these options, another common technique is using **batching**.
Instead of sending one packet at a time, you send multiple ones at once.
The advantage is that the device backend is notified just once per batch instead of for each packet.(less context switches)
This reduces communication overhead.
This feature is currently submitted as [PR#243](https://github.com/unikraft/unikraft/pull/243) and will be added in the near future. We'll not be using
this optimization today, but is good to know about it.

Which option increased the performance the most?

#### 06.1. Use a memory pool

We provide a pool allocator library with Unikraft: `libukallocpool`.
First of all, add a dependency to this library in your `Config.uk`:

```
depends on LIBUKALLOCPOOL
```

This dependency makes the header [`"<uk/allocpool.h>"`](https://github.com/unikraft/unikraft/blob/staging/lib/ukallocpool/include/uk/allocpool.h) available.

In order to allocate one pool, you call `uk_allocpool_alloc()` at your application startup.
The function will allocate the pool memory from a parent allocator.
This happens just during creation time for pre-allocating all the pool objects.

In our case this parent is the default allocator.
As `obj_len` you should choose `2048` because this is a big enough buffer to keep packet data and needed meta data.
`obj_align` should be again set to the alignment requirement of the device (`struct uk_netdev_info`->`ioalign`).
The `obj_count` argument should be big enough so that we do not run out of pool objects while sending.
You can try different values, start with `1024`.

```C
struct uk_allocpool *pool;

pool = uk_allocpool_alloc(uk_alloc_get_default(), 1024, 2048,
                          netdev_info.ioalign);
```

In order to use the pool as allocator for `pktgen_udp4()`  and `uk_netbuf_dup_single()`, you need to get the compatibility interface from libukallocpool:

```C
struct uk_alloc *p;

p = uk_allocpool2ukalloc(pool);
```

`p` can then be handed over as normal allocator, simply replace all the calls to `uk_alloc_get_default()` with `p`.
**Note**.`p` will always return 2048B objects as long as the malloc request is smaller or equal to the initialized `obj_len`.
Any bigger allocation request cannot be satisfied and libukallocpool is returning `NULL`.

### 07. Receiver unikernel

With the last task you will implement a receive-only unikernel that measures the received traffic.
We keep busy polling for receive as well but you should implement a switching logic to switch between transmit and receive mode.
You can do this either with a configuration option (`Config.uk`) or with a kernel argument (see: `int argc, char *argv[]`).

Opening the network device is the same for receive except that we implement and hand-over a proper receive buffer allocation function.
This will replace `dummy_alloc_rxpkts()` when the receive mode is activated.
We can use the same pool allocator that we allocated for transmit during task 06.1.

Whenever the driver calls our callback, it tries to setup new receive buffers to receive new packet data.
When filled, these buffers are later returned back to us.
The function should look like this:

```C
/* global variables, fill-out before configuring the receive queue */
struct uk_netdev_info netdev_info;
struct uk_alloc *p;

uint16_t alloc_rxpkts(void *argp __unused,
                      struct uk_netbuf *pkts[],
                      uint16_t count)
{
	uint16_t i;

	/* fill out given array with allocated receive buffers */
	for (i=0; i<count; ++i) {
	    pkts[i] = uk_netbuf_alloc_buf(p,
                                      2048,
                                      netdev_info.ioalign,
                                      netdev_info.nb_encap_rx, /* headroom for rx */
                                      0, NULL);
	    if (!pkts[i])
			break; /* We ran out of memory */
	}
	return i;
}
```

Please note that this function expects that we initialized the global variables `netdev_info` and `p` before we configure the receive queue.

Now you should be able to build the polling receive loop based on the following snippet:

```c
status = uk_netdev_rx_one(dev, 0, &pkt);
if (uk_netdev_status_successful(status)) {
	/* count packet and bytes and free received packet */
	nb_total_pkts += 1;
	nb_total_bytes += pkt->len + 24;
	uk_netbuf_free(pkt);
}
```

In order to test your configuration, you can run 2 unikernels that are both connected to `asplosbr0`.
One is transmitting traffic and the other one receives it.
