# SecureROM DFU emulation notes for Unicorn

This guide documents the parts of Inferno that matter if you only want to run Apple SecureROM in DFU mode, emulate the USB path that feeds it, and build a fuzzing harness around that path.

It is based on the current S8000 implementation in Inferno, which is the clearest reference path for SecureROM + DFU in this tree. The same high-level pattern also exists on newer Apple SoCs in this repository, but the file paths and symbols below focus on S8000 because that is where the ROM/DFU path is easiest to trace.

## Relevant files

- `hw/arm/apple-silicon/s8000.c`
- `include/hw/arm/apple-silicon/s8000.h`
- `hw/usb/apple_otg.c`
- `include/hw/usb/apple_otg.h`
- `hw/usb/hcd-dwc2.c`
- `include/hw/usb/hcd-dwc2.h`
- `hw/usb/hcd-tcp.c`
- `include/hw/usb/hcd-tcp.h`
- `hw/usb/tcp-usb.h`
- `hw/intc/apple_aic.c`
- `include/hw/usb.h`

## What Inferno does in SecureROM mode

### 1. It switches the machine from kernel boot to ROM boot

In `s8000_init()` the machine loads either the kernel path or the SecureROM path:

- if `securerom` is **not** provided, it loads the normal boot artifacts
- if `securerom` **is** provided, it reads the SecureROM file into memory and skips the normal boot path

The important bits are:

- `SROM_BASE = 0x100000000`
- `SROM_SIZE = 512 * KiB`
- `DRAM_BASE = 0x800000000`
- `SRAM_BASE = 0x180000000`

Even though this is conceptually ROM, Inferno allocates the SROM region as RAM and writes the SecureROM image into it during reset-time memory setup. That is useful for Unicorn too: you can map it as normal memory, write the ROM bytes, and then treat it as read-only in your own hooks if you want stricter behavior.

### 2. It starts the CPU at SecureROM

In `s8000_cpu_reset()`:

- normal boot uses `TZ1_BASE`
- SecureROM boot sets `rvbar = SROM_BASE`
- CPU0 is started at `SROM_BASE`

For a Unicorn model, that means your initial PC should be the ROM reset entry and your initial memory image must already contain the SecureROM at `0x100000000`.

### 3. It forces DFU by driving a GPIO input

`GPIO_FORCE_DFU` is defined as GPIO pin `123`.

In `s8000_reset()` Inferno does:

- `qemu_set_irq(qdev_get_gpio_in(gpio, GPIO_FORCE_DFU), s8000->force_dfu);`

So the ROM enters DFU because the platform model asserts the GPIO line that the ROM checks during early boot. For Unicorn, you do not need a full GPIO controller if your goal is only DFU mode. You only need whatever MMIO read path the ROM uses to observe that this line is asserted.

## How USB is wired in Inferno

### Topology

The USB path is:

`S8000 machine -> apple-otg wrapper -> DWC2 controller -> internal DWC2 USB device -> usb-tcp-host transport -> external client`

The key construction happens in two places:

### `s8000_create_usb()` in `hw/arm/apple-silicon/s8000.c`

This function:

- finds the DT nodes under `arm-io`
- creates the OTG device with `apple_otg_from_node()`
- copies the machine USB connection properties into it:
  - `usb-conn-type`
  - `usb-conn-addr`
  - `usb-conn-port`
- maps the OTG MMIO windows
- connects the OTG IRQ to the Apple AIC using the interrupt value from the `usb-device` DT node

### `apple_otg_from_node()` and `apple_otg_realize()` in `hw/usb/apple_otg.c`

This code creates four MMIO windows:

- MMIO 0: PHY registers
- MMIO 1: USB control registers
- MMIO 2: DWC2 controller registers
- MMIO 3: widget registers

It also:

- instantiates a `dwc2` controller
- instantiates a `usb-tcp-host`
- realizes the internal DWC2 USB device on the TCP host's bus

That last part is the important model detail: the ROM-side USB controller is represented as a USB device that a remote host can talk to over the TCP transport.

## The DWC2 pieces that matter

`include/hw/usb/hcd-dwc2.h` shows the register layout that matters for a ROM-only harness.

Important blocks:

- global registers at `0x000`
- host registers at `0x400`
- device registers at `0x800`
- IN endpoint registers at `0x900`
- OUT endpoint registers at `0xB00`
- power/clock block at `0xE00`
- FIFO window at `0x1000`

Important state in Inferno:

- `gintsts`, `gintmsk`, `gahbcfg`
- `dcfg`, `dctl`, `dsts`
- `daint`, `daintmsk`
- `diepctl(n)`, `diepint(n)`, `dieptsiz(n)`, `diepdma(n)`
- `doepctl(n)`, `doepint(n)`, `doeptsiz(n)`, `doepdma(n)`

Important reset behavior in `dwc2_reset_enter()` / `dwc2_reset_exit()`:

- EP0 IN and EP0 OUT are marked active by default
- device address is reset to `0`
- the controller sets up default FIFO sizes
- timers are created for frame boundaries

That default EP0 activation is important: a ROM DFU harness can start with only endpoint 0 working and still reach the early DFU control path.

## How a USB packet reaches ROM code

### Transport layer

`hw/usb/tcp-usb.h` defines a small packed protocol:

- `TCP_USB_REQUEST`
- `TCP_USB_RESPONSE`
- `TCP_USB_RESET`
- `TCP_USB_CANCEL`

Request header:

- `addr`
- `pid`
- `ep`
- `id`
- `stream`
- `short_not_ok`
- `int_req`
- `length`

The token values come from `include/hw/usb.h`:

- `USB_TOKEN_SETUP = 0x2d`
- `USB_TOKEN_IN = 0x69`
- `USB_TOKEN_OUT = 0xe1`

### Receive path

`usb_tcp_host_msg_loop_co()` in `hw/usb/hcd-tcp.c`:

1. reads a transport header
2. for `TCP_USB_REQUEST`, reads `tcp_usb_request_header`
3. looks up the endpoint with `usb_ep_get()`
4. creates a `USBPacket`
5. copies payload bytes into the packet buffer for non-IN tokens
6. calls `usb_handle_packet()`

### DWC2 device-side path

`dwc2_usb_device_handle_packet()` calls `dwc2_device_process_packet()`.

Inside `dwc2_device_process_packet()`:

- `USB_TOKEN_SETUP` and `USB_TOKEN_OUT` use the OUT endpoint path
- `USB_TOKEN_IN` uses the IN endpoint path
- setup packets on EP0 update endpoint state and raise endpoint interrupts
- a `SET_ADDRESS` setup request triggers `GINTSTS_ENUMDONE`

Important EP0 behavior visible in the model:

- setup packets set `DXEPINT_SETUP`
- setup packets set `DXEPINT_SETUP_RCVD`
- completed transfers set `DXEPINT_XFERCOMPL`
- device IRQ aggregation updates `daint`
- global IRQ aggregation raises `GINTSTS_IEPINT` or `GINTSTS_OEPINT`
- `dwc2_update_irq()` asserts the single DWC2 IRQ line only when both:
  - `gintsts & gintmsk` is non-zero
  - `gahbcfg` has `GAHBCFG_GLBL_INTR_EN`

### Interrupt delivery to the CPU

`s8000_create_usb()` connects the OTG IRQ to Apple AIC.

In `hw/intc/apple_aic.c`, the main offsets you care about for a minimal model are:

- `REG_AIC_IACK = 0x2004`
- `REG_AIC_EIR_DEST(n) = 0x3000 + n * 4`
- `REG_AIC_EIR_MASK_SET(n) = 0x4100 + n * 4`
- `REG_AIC_EIR_MASK_CLR(n) = 0x4180 + n * 4`

For a Unicorn-only harness, you do not need the full AIC implementation. You only need enough behavior for:

- the ROM to unmask the USB interrupt
- the CPU to observe that an interrupt is pending
- the ROM to acknowledge the vector it expects

## Minimum model you should emulate in Unicorn

If your goal is ROM + DFU only, emulate these pieces first.

### Required memory

- SecureROM at `0x100000000` (`512 KiB`)
- DRAM at `0x800000000`
- SRAM at `0x180000000`

You can start with a much smaller DRAM/SRAM mapping than Inferno if the ROM code you are exercising does not touch the full range.

### Required platform state

- CPU reset PC at `SROM_BASE`
- a DTB or equivalent hardcoded constants for the USB and AIC MMIO bases
- a forced DFU condition
- one USB interrupt line

### Required MMIO behavior

#### GPIO

Only the DFU forcing path is required at first.

#### DWC2

At minimum, model:

- the global interrupt enable/mask path
- EP0 OUT setup reception
- EP0 IN response path
- transfer completion bits
- DMA or FIFO data movement into ROM-visible memory

You do **not** need a full host controller, periodic schedule, hub logic, or later-stage iBoot behavior.

#### AIC

At minimum, model:

- interrupt mask/unmask for the USB vector
- pending USB vector delivery
- interrupt acknowledge

### Recommended simplification order

1. boot ROM with DFU forced
2. stub enough GPIO/AIC reads for the ROM not to hang
3. emulate only DWC2 registers the ROM actually touches
4. accept only EP0 setup/IN/OUT packets
5. add DMA writes into ROM buffers
6. add remaining endpoint behavior only if the ROM reaches it

## Suggested control-flow for your Unicorn emulator

1. map SROM, DRAM, SRAM
2. write SecureROM bytes into SROM
3. set CPU registers to reset state with PC at `SROM_BASE`
4. install MMIO hooks for:
   - GPIO block used by DFU detection
   - DWC2 block
   - AIC block
5. run until the ROM enables USB interrupts and waits for traffic
6. inject a USB reset event
7. inject EP0 setup packets
8. when your DWC2 model has a completed packet, mark the IRQ pending
9. let ROM service the interrupt and read/write its buffers
10. repeat with mutated control payloads for fuzzing

## Example Unicorn scaffold

This is intentionally small and incomplete. It shows the state you need to keep around your Unicorn instance, not a full SoC model.

The exact MMIO callback API depends on the Unicorn version and binding you use, so the example below keeps the MMIO dispatch layer abstract on purpose.

```python
from unicorn import Uc, UC_ARCH_ARM64, UC_MODE_ARM
from unicorn.arm64_const import UC_ARM64_REG_PC

SROM_BASE = 0x100000000
SROM_SIZE = 512 * 1024
DRAM_BASE = 0x800000000
DRAM_SIZE = 16 * 1024 * 1024
SRAM_BASE = 0x180000000
SRAM_SIZE = 2 * 1024 * 1024

GAHBCFG_OFFSET = 0x08
GINTSTS_OFFSET = 0x14
GINTMSK_OFFSET = 0x18

class MinimalDWC2:
    def __init__(self, usb_irq_vector):
        self.regs = {}
        self.usb_irq_vector = usb_irq_vector
        self.irq_pending = False
        self.ep0_out_dma = 0
        self.ep0_in_dma = 0
        self.gintsts = 0
        self.gintmsk = 0
        self.gahbcfg = 0

    def read32(self, off):
        if off == GINTSTS_OFFSET:
            return self.gintsts
        if off == GINTMSK_OFFSET:
            return self.gintmsk
        if off == GAHBCFG_OFFSET:
            return self.gahbcfg
        return self.regs.get(off, 0)

    def write32(self, off, value):
        self.regs[off] = value & 0xFFFFFFFF
        if off == GAHBCFG_OFFSET:
            self.gahbcfg = value & 0xFFFFFFFF
        elif off == GINTMSK_OFFSET:
            self.gintmsk = value & 0xFFFFFFFF
        elif off == GINTSTS_OFFSET:  # write-1-to-clear in real hw
            self.gintsts &= ~value

    def inject_ep0_setup(self, uc, dma_addr, setup_packet):
        uc.mem_write(dma_addr, setup_packet)
        self.ep0_out_dma = dma_addr
        # In a fuller model, also update doepint(0), daint and gintsts
        # the same way Inferno's dwc2_device_process_packet() does.
        self.irq_pending = True

class MinimalAIC:
    REG_IACK = 0x2004

    def __init__(self, usb_irq_vector):
        self.usb_irq_vector = usb_irq_vector
        self.unmasked = set()
        self.usb_pending = False

    def read32(self, off):
        if off == self.REG_IACK and self.usb_pending:
            self.usb_pending = False
            return self.usb_irq_vector
        return 0

    def write32(self, off, value):
        pass

def build_emulator(rom_bytes, dwc2_base, aic_base, usb_irq_vector):
    uc = Uc(UC_ARCH_ARM64, UC_MODE_ARM)
    uc.mem_map(SROM_BASE, SROM_SIZE)
    uc.mem_map(DRAM_BASE, DRAM_SIZE)
    uc.mem_map(SRAM_BASE, SRAM_SIZE)
    uc.mem_write(SROM_BASE, rom_bytes)
    uc.reg_write(UC_ARM64_REG_PC, SROM_BASE)

    dwc2 = MinimalDWC2(usb_irq_vector)
    aic = MinimalAIC(usb_irq_vector)
    # Wire dwc2.read32/write32 and aic.read32/write32 into whatever
    # MMIO callback mechanism your Unicorn binding or wrapper exposes at
    # dwc2_base and aic_base.
    return uc, dwc2, aic, {"dwc2": dwc2_base, "aic": aic_base}
```

### Why this scaffold is useful

It lets you start with:

- ROM execution
- fake MMIO blocks
- a single interrupt path
- direct injection of setup packets into the ROM-visible DMA buffer

That is enough to start finding where the ROM expects USB traffic before you build a full DWC2 emulation.

## Example TCP USB packet sender

If you want to keep Inferno as the reference model and drive it externally, the transport in `tcp-usb.h` is simple enough to script.

This example sends:

1. a transport reset
2. a USB `SET_ADDRESS` setup packet to endpoint 0

```python
import socket
import struct

TCP_USB_REQUEST = 1
TCP_USB_RESET = 3

USB_TOKEN_SETUP = 0x2D

def send_reset(sock):
    sock.sendall(struct.pack("<B", TCP_USB_RESET))

def send_setup(sock, request_id, dev_addr, ep, setup_packet):
    header = struct.pack("<B", TCP_USB_REQUEST)
    # pid is signed here because tcp_usb_request_header declares it as `int`.
    req = struct.pack(
        "<B i B Q I B B H",
        dev_addr,          # addr
        USB_TOKEN_SETUP,   # pid
        ep,                # ep
        request_id,        # id
        0,                 # stream
        0,                 # short_not_ok
        0,                 # int_req
        len(setup_packet), # length
    )
    sock.sendall(header + req + setup_packet)

sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
sock.connect("/tmp/InfernoUSBRemote")

send_reset(sock)

set_address = struct.pack(
    "<BBHHH",
    0x00,  # bmRequestType
    0x05,  # SET_ADDRESS
    0x0001,  # wValue
    0x0000,  # wIndex
    0x0000,  # wLength
)

send_setup(sock, request_id=1, dev_addr=0, ep=0, setup_packet=set_address)
```

### Notes about this example

- The default Unix socket path is `/tmp/InfernoUSBRemote`.
- That default path comes from Inferno itself; for real testing, prefer a private socket path in a directory only your user can access.
- For TCP mode, use the S8000 machine properties `usb-conn-type`, `usb-conn-addr`, and `usb-conn-port`.
- This is the transport-level injection example, not a complete USB host stack.
- A real control transfer usually also includes the status stage after the setup stage.
- Inferno's DWC2 model has explicit special handling for `SET_ADDRESS`, so this is a good first packet when you are testing enumeration.

## Minimal packet flow for a DFU control transfer

For a real control transfer on EP0, use this order:

1. host issues USB reset
2. host sends `SET_ADDRESS`
3. host sends descriptor requests until the ROM is enumerated
4. host sends a DFU class request on EP0

For a class request with data from host to device, the flow is:

1. `USB_TOKEN_SETUP` on EP0 with the 8-byte setup packet
2. `USB_TOKEN_OUT` on EP0 with the data stage payload
3. `USB_TOKEN_IN` on EP0 for the status stage

In your Unicorn model, the important part is not perfect host timing. The important part is making the ROM see:

- bytes land in its DMA/FIFO buffer
- endpoint status bits change
- the USB interrupt becomes pending

## What to fuzz first

If your goal is checkm8-style bug hunting, start with a harness that resets back to ROM DFU mode every iteration and mutates only USB-visible inputs.

Recommended first targets:

- EP0 setup packet fields
- EP0 OUT data stage length
- descriptor/class request mixtures
- short packets vs exact packet sizes
- repeated resets, cancels, and partially completed control transfers

Recommended first harness structure:

1. boot ROM to the DFU loop
2. snapshot CPU + RAM + MMIO state
3. restore snapshot
4. inject one mutated control transfer
5. run until:
   - exception
   - panic pattern
   - invalid MMIO
   - timeout
6. save the triggering packet sequence

If you fuzz the Unicorn model rather than Inferno directly, keep the model small. The closer your harness is to "ROM + EP0 + interrupt + DMA buffer", the easier it will be to reason about crashes.

## Practical implementation advice

- Parse the DTB once and hardcode the USB/AIC/GPIO bases into your first harness.
- Start with DWC2 device mode only.
- Prefer DMA-backed packet injection over trying to emulate every FIFO corner case immediately.
- Add traces for:
  - MMIO reads/writes
  - IRQ assertion/acknowledge
  - writes into the ROM's USB buffers
  - endpoint register changes
- Stop emulating anything that belongs only to later boot stages.

## Short checklist

You are ready to start checkm8-focused experiments once you can do all of these:

- boot from `SROM_BASE`
- force DFU
- deliver one EP0 setup packet
- raise the USB interrupt the ROM expects
- observe the ROM consume the packet
- repeat the sequence from a clean snapshot
