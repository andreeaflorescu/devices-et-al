# Devices and others in rust-vmm

The purpose of this document is to describe the responsibilities and concerns
of devices in respect to handling interrupts.

The following sections are my observations when looking at the Firecracker
and CrosVM code.

## What do all devices need?

All devices need a way for triggering interrupts. Since this is a functionality
needed by all devices, we can define an `Interrupt` trait in `vm-device`
because the expectation is that all device implementation in rust-vmm will
take a dependency on vm-device.

Since the only common thing in devices is triggering interrupts, we should
insist on simplicity and extend the functionality only when needed.

```rust
pub trait Interrupt {
    fn trigger(&self) -> io::Result<()>;
}
```

If triggering an interrupt failed, there is not much to do about it.
Nevertheless, we can still return an error that can end up in a log file for
example. The caller needs to decide what to do with the error.

## What do all Virtio Devices need?

1. Trigger an interrupt.
2. Suppress interrupts.
3. Handling the interrupt status.

Number 1 on the list is already provided by the `Interrupt` trait from
vm-device. We can just import it.

Suppressing interrupts is a simple optimization mechanism which I don't believe
should need a trait. You can read more about it here:
http://docs.oasis-open.org/virtio/virtio/v1.0/cs04/virtio-v1.0-cs04.html#x1-370007

Number 3 unfortunately is different based on the transport so even this one
should be implemented somewhere else.

As a recap it looks like all virtio devices:
 - **do**: trigger an interrupt
 - **have**: an interrupt status.
 
As the specification also states: _"The modus operandi of the notifications
is **transport** specific"_.

Let's talk about transport.

## What do all MMIO Virtio Devices need?

1. Identify what caused an interrupt by reading the Interrupt Status.

TODO.

## What do all PCI Virtio Devices need?

This is where things get complicated and very much branched depending on
MSI-X being available or not.

TODO.

## What is missing?

You might have noticed that I completed ignored KVM and Interrupt Controllers
from this doc. This is on purpose. Here is why:

1. No KVM: There is no need for virtio device emulation to depend on the
   hypervisor. We can make rust-vmm attractive for other projects that don't
   use KVM. Especially if doing so it's not hard.
2. No Interrupt Controller logic. The device doesn't and shouldn't care about
   the interrupt controller. The device should only care about having a way
   to deliver interrupts. If these interrupts are routed by KVM great. If you
   implement IOAPIC or other black magic, also great. But there is no need
   for controller logic to affect in anyway the devices. This also gives the
   extra benefit that the aarch64 and x86_64 code does not need to be different
   for common devices. The only thing that differs between the platforms is
   the implementation of the interrupt controller.