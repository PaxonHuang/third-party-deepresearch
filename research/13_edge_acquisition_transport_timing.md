# Edge acquisition: transport, scheduling and timestamp lessons

> Date: 2026-09-27. Status: research methodology and engineering inference; no device benchmark or deployment claim.
> Scope: generic portable multimodal acquisition. Private project inventory, wiring, procurement and capture evidence are intentionally outside this public note.

## 1. Identify the actual transport before budgeting

A USB serial device may terminate USB directly or bridge to a physical UART. The tty/COM API does not decide which. Read the board protocol and enumerate descriptors, driver and negotiated speed. CDC is a USB class, not a synonym for UART hardware.

For a physical 8N1 UART, ideal useful byte rate is baud/10. N independent converters provide N separate UART links, while their USB upstream and host resources may be shared. For a request/response protocol, include request bytes, response framing, payload, processing and scheduling. Aggregate streaming formats need a separate calculation; never reuse per-sensor request overhead without checking the format.

USB signalling rates are not application throughput. USB 2.0 Full Speed is a host-scheduled shared bus; a “full duplex” label must not become a budget of the headline bitrate in each direction. Multiple connectors do not establish separate host controllers. Inspect the real bus tree and test combined camera/sensor/storage load.

## 2. Linux and real-time acquisition

Hardware SPI generates transfer clocks independently of userspace scheduling. Linux can perform digital sensor reads without an RTOS, but request timing, response consumption and worst-case latency require measurement. Batch/composite transfers can reduce syscall gaps; GPIO bit-banging is not an equivalent timing guarantee.

Bare resistive/capacitive elements require excitation/analog front-end and ADC/capacitance conversion. GPIO is not an analog input. Use hardware timing, FIFO/DMA/buffers and explicit continuity evidence in an MCU, FPGA or autonomous digitizer as appropriate. An RTOS alone does not establish deterministic sampling, and an extra MCU is not mandatory if suitable acquisition hardware already owns the timing.

## 3. Separate event clocks

Keep sample/conversion time, device readout time, host receive/dequeue time and inference completion distinct. A request may read an earlier sample from a register. A response's arrival is not its conversion trigger. High reply rate does not prove equally high fresh-sample rate.

Map clock domains with an affine model and validity segments. Two-way exchanges or matched markers can estimate offset/drift with uncertainty; one-way delay and clock offset are confounded. A mapping cannot recover an event never timestamped. Camera driver flags distinguish clock domain and exposure/frame-end semantics; verify device support rather than treating all frame timestamps as exposure time.

CRC/LRC proves limited received-frame integrity. A host-created sequence counts host events; neither proves sensor-native sample completeness. Without a native counter/timestamp, some upstream loss is unobservable. Report that limit rather than a false lossless result.

## 4. Portable pipeline and validation

Use independent device readers, stable device identities, bounded queues and local raw recording. Make reconnects explicit. Vision and preview may consume latest frames; raw-recording losses must remain visible. Network/display loss should not silently terminate a local session. Test memory, storage stalls, supply margins, cable motion and queue age as well as average bandwidth.

Validate one source, then several, then combined load and worn operation. Software parsing, successful reads, sustained transport, synchronization, calibration and task accuracy are different evidence levels. Arrival dates and protocol calculations do not satisfy acceptance gates.

## 5. Sources

- [Linux SPI userspace API](https://docs.kernel.org/spi/spidev.html): hardware/composite transfers and userspace limitations.
- [Linux V4L2 buffer timestamps](https://www.kernel.org/doc/html/v4.9/media/uapi/v4l/buffer.html): clock and timestamp-source flags.
- [NVIDIA Nano 2GB user guide](https://developer.nvidia.com/embedded/learn/jetson-nano-2gb-devkit-user-guide): example of model-specific port roles, power and memory constraints. Do not transfer one model's specifications to another.
- [Espressif USB Host documentation](https://docs.espressif.com/projects/esp-idf/en/v6.0-beta1/esp32s3/api-reference/peripherals/usb_host.html): example of controller/PHY resource sharing; validate the actual platform before designing a USB relay.

The architecture/validation recommendations above are engineering synthesis. No private vendor package is redistributed, and no vendor-specific throughput is claimed from these general sources.
