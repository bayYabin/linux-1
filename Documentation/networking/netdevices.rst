.. SPDX-License-Identifier: GPL-2.0

=====================================
Network Devices, the Kernel, and You!
=====================================


Introduction
============
The following is a random collection of documentation regarding
network devices.

struct net_device allocation rules
==================================
Network device structures need to persist even after module is unloaded and
must be allocated with alloc_netdev_mqs() and friends.
If device has registered successfully, it will be freed on last use
by free_netdev(). This is required to handle the pathologic case cleanly
(example: rmmod mydriver </sys/class/net/myeth/mtu )

alloc_netdev_mqs()/alloc_netdev() reserve extra space for driver
private data which gets freed when the network device is freed. If
separately allocated data is attached to the network device
(netdev_priv(dev)) then it is up to the module exit handler to free that.

MTU
===
Each network device has a Maximum Transfer Unit. The MTU does not
include any link layer protocol overhead. Upper layer protocols must
not pass a socket buffer (skb) to a device to transmit with more data
than the mtu. The MTU does not include link layer header overhead, so
for example on Ethernet if the standard MTU is 1500 bytes used, the
actual skb will contain up to 1514 bytes because of the Ethernet
header. Devices should allow for the 4 byte VLAN header as well.

Segmentation Offload (GSO, TSO) is an exception to this rule.  The
upper layer protocol may pass a large socket buffer to the device
transmit routine, and the device will break that up into separate
packets based on the current MTU.

MTU is symmetrical and applies both to receive and transmit. A device
must be able to receive at least the maximum size packet allowed by
the MTU. A network device may use the MTU as mechanism to size receive
buffers, but the device should allow packets with VLAN header. With
standard Ethernet mtu of 1500 bytes, the device should allow up to
1518 byte packets (1500 + 14 header + 4 tag).  The device may either:
drop, truncate, or pass up oversize packets, but dropping oversize
packets is preferred.


struct net_device synchronization rules
=======================================
ndo_open:
	Synchronization: rtnl_lock() semaphore.
	Context: process

ndo_stop:
	Synchronization: rtnl_lock() semaphore.
	Context: process
	Note: netif_running() is guaranteed false

ndo_do_ioctl:
	Synchronization: rtnl_lock() semaphore.
	Context: process

ndo_get_stats:
	Synchronization: rtnl_lock() semaphore, dev_base_lock rwlock, or RCU.
	Context: atomic (can't sleep under rwlock or RCU)

ndo_start_xmit:
	Synchronization: __netif_tx_lock spinlock.

	When the driver sets NETIF_F_LLTX in dev->features this will be
	called without holding netif_tx_lock. In this case the driver
	has to lock by itself when needed.
	The locking there should also properly protect against
	set_rx_mode. WARNING: use of NETIF_F_LLTX is deprecated.
	Don't use it for new drivers.

	Context: Process with BHs disabled or BH (timer),
		 will be called with interrupts disabled by netconsole.

	Return codes:

	* NETDEV_TX_OK everything ok.
	* NETDEV_TX_BUSY Cannot transmit packet, try later
	  Usually a bug, means queue start/stop flow control is broken in
	  the driver. Note: the driver must NOT put the skb in its DMA ring.

ndo_tx_timeout:
	Synchronization: netif_tx_lock spinlock; all TX queues frozen.
	Context: BHs disabled
	Notes: netif_queue_stopped() is guaranteed true

ndo_set_rx_mode:
	Synchronization: netif_addr_lock spinlock.
	Context: BHs disabled

struct napi_struct synchronization rules
========================================
napi->poll:
	Synchronization:
		NAPI_STATE_SCHED bit in napi->state.  Device
		driver's ndo_stop method will invoke napi_disable() on
		all NAPI instances which will do a sleeping poll on the
		NAPI_STATE_SCHED napi->state bit, waiting for all pending
		NAPI activity to cease.

	Context:
		 softirq
		 will be called with interrupts disabled by netconsole.
