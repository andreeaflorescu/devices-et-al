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

MMIO uses a single interrupt per device. Notifications are triggered by the
device to let the driver know that there is something available in the used
buffer or that the device configuration changed.

1. Trigger an interrupt to send a used buffer notification and/or a configuration
   change notification.
2. Set the interrupt status register to let the driver know about the reason
   of an interrupt (Write on InterruptStatus register).
3. Read the acknowledgment of an interrupt (Read on InterruptACK register).

Triggering an interrupt and writing the interrupt status register are always
done together and depend on the notification type. For this reasons the MMIO
Interrupt defines this functionality as `signal_used_queue` and
`signal_configuration_change`. This comes with the benefit of not having the
`Interrupt` trait depend on the notification type which is something related to
how virtio devices on MMIO work.

```rust
use vm_device::Interrupt;

pub enum InterruptError {
    SingalUsedQueueFailed(io::Error),
    SignalConfigurationChangedFailed(io::Error),
}

pub struct MMIOInterrupt {
    interrupt_status: Arc<AtomicUsize>,
    // EventFd is not very friendly because this is the mechanism used by
    // KVM and it is only available on Linux. We should probably abstract this
    // somehow to make it useful for other hypervisors and other OSs.
    interrupt_evt: EventFd,
}

impl Interrupt for MMIOInterrupt {
    fn trigger(&self) -> io::Error {
       self.interrupt_evt.write(1)
    }
}

impl MMIOInterrupt {
    const CONFIGURATION_CHANGED_NOTIFICATION: usize = 0x0;
    const USED_QUEUE_NOTIFICATION: usize = 0x1;

    fn signal_used_queue(&self) -> InterruptError {
        self.interrupt_status
            .fetch_or(USED_QUEUE_NOTIFICATION, Ordering::SeqCst);
        self.interrupt().map_err(SingalUsedQueueFailed)
    }

    fn signal_configuration_changed(&self) -> InterruptError {
        self.interrupt_status
            .fetch_or(CONFIGURATION_CHANGED_NOTIFICATION, Ordering::SeqCst);
        self.interrupt().map_err(SignalConfigurationChangedFailed)
    }

    fn ack(&self, bit_mask: usize) {
        self.interrupt_status
            .fetch_and(!(bit_mask as usize), Ordering::SeqCst);
    }
}
```

## What do all PCI Virtio Devices need?

This is where things get complicated and very much branched depending on
MSI-X being available or not.

TODO.

## Can the MMIO and PCI interrupt logic be merged?

I don't think so, but merging them at this point in time seems like an early
optimization. We should start with 2 traits/structs for interrupts on MMIO and
PCI and do this optimization if needed afterwards.

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