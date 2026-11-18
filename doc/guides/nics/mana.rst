..  SPDX-License-Identifier: BSD-3-Clause
    Copyright 2022 Microsoft Corporation

MANA poll mode driver library
=============================

The MANA poll mode driver library (**librte_net_mana**) implements support
for Microsoft Azure Network Adapter VF in SR-IOV context.

Prerequisites
-------------

This driver relies on external libraries and kernel drivers
for resources allocations and initialization.
The following dependencies are not part of DPDK
and must be installed separately:

- **libibverbs** (provided by rdma-core package)

  User space verbs framework used by librte_net_mana.
  This library provides a generic interface between the kernel
  and low-level user space drivers such as libmana.

  It allows slow and privileged operations
  (context initialization, hardware resources allocations)
  to be managed by the kernel and fast operations to never leave user space.
  The minimum required rdma-core version is v44.

  In most cases, rdma-core is shipped as a package with an OS distribution.
  User can also install the upstream version of the rdma-core from
  https://github.com/linux-rdma/rdma-core.

- **libmana** (provided by rdma-core package)

  Low-level user space driver library
  for Microsoft Azure Network Adapter devices,
  it is automatically loaded by libibverbs.
  The minimum required version of rdma-core with libmana is v44.

- **Kernel modules**

  They provide the kernel-side verbs API and low level device drivers
  that manage actual hardware initialization
  and resources sharing with user space processes.
  The minimum required Linux kernel version is 6.2.

  Unlike most other PMDs, these modules must remain loaded
  and bound to their devices:

  - mana: Ethernet device driver that provides kernel network interfaces.
  - mana_ib: InifiniBand device driver.
  - ib_uverbs: user space driver for verbs (entry point for libibverbs).

Driver compilation and testing
------------------------------

Refer to the document :doc:`build_and_test` for details.

Runtime Configuration
---------------------

The user can specify below argument in devargs.

#.  ``mac``:

    Specify the MAC address for this device.
    If it is set, the driver probes and loads the NIC
    with a matching MAC address.
    If it is not set, the driver probes on all the NICs on the PCI device.
    The default value is not set,
    meaning all the NICs will be probed and loaded.
    User can specify multiple mac=xx:xx:xx:xx:xx:xx arguments for up to 8 NICs.

Device Reset Support
--------------------

The MANA PMD supports automatic recovery from hardware service reset events.
When the MANA kernel driver receives a hardware service event,
it initiates a device reset and notifies userspace
via ``IBV_EVENT_DEVICE_FATAL``.

The driver handles this transparently through a two-phase reset flow:

Enter phase
   Each MANA vport receives a device fatal notification through its own
   verbs context and asynchronous event file descriptor.
   The interrupt handler initiates recovery only for the notified vport,
   blocks new data path bursts,
   and waits for all in-flight burst calls to drain
   using per-queue atomic flags,
   then spawns a control thread for the remaining work.

Teardown and exit phase
   The control thread tears down IB resources and queues,
   unmaps secondary process doorbell pages, and closes the device.
   Each control thread re-probes its vport by matching its PCI address
   and MAC address after a delay for hardware recovery.
   Recovery assumes that the MAC uniquely identifies the vport among
   eligible ports on the PCI device and remains unchanged across reset.
   This avoids replacing a sibling's protection domain while its TX queues
   still own the vport, which would make further queue-pair creation fail
   with ``EBUSY``.
   If the device is still unavailable, it waits 15 seconds and retries
   until the device returns, reinstalls the interrupt handler,
   reinitializes resources, and restarts queues.
   Queues are restarted only if the vport's current ethdev started state
   is set at the restart decision.
   Secondary-process doorbell mappings are removed and restored
   separately for each affected vport.
   Synchronous control requests are serialized within each process.
   If recovery fails, packet bursts remain blocked.

The driver emits the following ethdev recovery events
to notify upper layers (e.g. netvsc) of the reset lifecycle:

``RTE_ETH_EVENT_ERR_RECOVERING``
   Reset has started.

``RTE_ETH_EVENT_RECOVERY_SUCCESS``
   Device has recovered successfully.

``RTE_ETH_EVENT_RECOVERY_FAILED``
   Recovery failed.

Recovery diagnostics
~~~~~~~~~~~~~~~~~~~~

MANA log messages contain the common prefix ``MANA_DIAG``.
Enable informational driver messages and timestamps in every primary and
secondary process by adding these EAL arguments before the application's ``--``::

   --log-level=pmd.net.mana.driver:info --log-timestamp=iso

Recovery stage records identify the process ID, process role, device name,
ethdev port number, verbs port number, MAC address, device state and return value.
Device state values are 0 (active), 1 (reset enter), 2 (reset exit) and
3 (reset failed).
Probe records include the target MAC, enumerated device count,
candidate port identities, attempt number and retry delay.
Failed verbs opens and protection-domain allocations report the captured
``errno``; verbs queries report their returned error number.
IPC records include request type, reply counts and secondary remap results.
Return values retain the underlying API's convention, so a positive probe
result is a count of matched ports, not an error.

Collect both standard output and standard error from each process.
Filter the saved logs with ``grep MANA_DIAG``.
For each recovering port, follow ``reset-enter``, ``teardown-complete``,
``probe-result``, ``secondary-reset-exit`` and ``recovery-success``.
``queues-restarted`` indicates that queue restart returned successfully;
``queues-left-stopped`` indicates that the application left the port stopped.
``mp-skip`` means no secondary was registered at the request decision.
These records confirm control-path progress, not successful packet delivery.
Verify that packet counters resume on every intended port as well.

To distinguish a PCI hot-remove from a service reset,
the driver registers for PCI device removal events.
This requires the application to call ``rte_dev_event_monitor_start()``
for removal events to be delivered (e.g. testpmd ``--hot-plug-handling`` option).
