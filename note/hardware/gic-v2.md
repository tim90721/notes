- [Introduction](#introduction)
- [Hardware components](#hardware-components)
  - [Distributer](#distributer)
    - [Registers](#registers)
  - [CPU interface](#cpu-interface)
  - [Registers](#registers-1)
  - [Interrupt lines](#interrupt-lines)
- [Handling interrupt with GIC](#handling-interrupt-with-gic)
  - [Initialize](#initialize)
  - [Handle the interrupt](#handle-the-interrupt)
- [Reference](#reference)

# Introduction
GIC is a hardware used to offload interrupt dispatching for CPU. If there isn't an interrupt controller, CPU will need several PINs to receive various interrupt and CPU will usually need to find out the PIN that raised the interrupt signal manually. The operation above will consume lots of CPU operations while CPU is in interrupt context. With an interrupt controller, it can help the CPU to handle the interrupt. However, it still depends on different hardware implementation.

The design of GIC (Generic Interrupt Controller) is a hardware design by Arm. There are several versions (v1, v2, v3, v4). Each version support different number of CPUs, interrupts and their hardware component may not be the same.

For GICv2, it has 2 major hardware componenet can be configured
- Distributer
- CPU interface

# Hardware components
## Distributer
The Distributor centralizes all interrupt sources, determines the priority of each interrupt, and for each CPU interface forwards the interrupt with the highest priority to the interface, for priority masking and preemption handling.

The Distributor provides a programming interface for:
- Globally enabling the forwarding of interrupts to the CPU interfaces.
- Enabling or disabling each interrupt.
- Setting the priority level of each interrupt.
- Setting the target processor list of each interrupt.
- Setting each peripheral interrupt to be level-sensitive or edge-triggered.
- Setting each interrupt as either Group 0 or Group 1.
- Forwarding an SGI to one or more target processors.

### Registers
- GICD_CTLR
  - Enable/disable GIC distributer
  - TODO: check with security
- GICD_TYPER
  - information about hardware capabilities
    - number of interrupts
    - security support
    - number of CPU interfaces
- GICD_IIDR
  - GIC distributer information about the implementer and version
- GICD_IGROUPRn
  - each bit corresponding to 1 interrupt
  - if set bit x to 1, interrupt x is belonged to grouper 1
- GICD_ISENABLERn
  - each bit corresponding to 1 interrupt
  - if set bit x to 1, interrupt x is enabled
- GICD_ICENABLERn
- GICD_SPENDRn
- GICD_CPENDRn
- GICD_ISACTIVERn
- GICD_ICACTIVERn
- GICD_IPRIORITYRn
  - each byte is corresponding to 1 interrupt
  - the higher the value, the lower priority it has
- GICD_ITARGETSRn
  - each byte is corresponding to 1 interrupt
  - for each byte, bit x is corresponding to CPU interface x
  - if bit x is set to 1, the interrupts is allowed to forward from distributer to CPU interface x
- GICD_ICFGRn
  - each 2 bits is corresponding to 1 interrupt
  - bit [1] set to 0 for level-sensitive, 1 for edge-triggered
- GICD_SGIR
  - used to generate software interrupt from CPUx to CPUy

## CPU interface
When enabled, a CPU interface takes the highest priority pending interrupt for its connected processor and determines whether the interrupt has sufficient priority for it to signal the interrupt request to the processor. To determine whether to signal the interrupt request to the processor, the CPU interface considers the interrupt priority mask and the preemption settings for the processor.

Each CPU interface block provides the interface for a processor that is connected to the GIC. Each CPU interface
provides a programming interface for:
- enabling the signaling of interrupt requests to the processor
- acknowledging an interrupt
- indicating completion of the processing of an interrupt
- setting an interrupt priority mask for the processor
- defining the preemption policy for the processor
- determining the highest priority pending interrupt for the processor.

## Registers
CPU interface registers are banked for each CPU. In other words, each CPU access the same address while it is actually configuring its own copy of CPU interface register
- GICC_CTLR
  - Enable/disable CPU interface
- GICC_PMR
  - CPU interface priority mask
  - If interrupt line is configured with lower priority than this mask, interrupt will not be visiable to the CPU connected to this CPU interface
  - The higher the value of this register, the lower the priority
  - To allow all interrupt, set this register to 0xFF (lowest priority)
- GICC_BPR
- GICC_IAR
  - Read the highest priority interrupt ID of the current interrupts
  - Read this register is acted as acknowledging an interrupt
- GICC_EOIR
  - Write to this register mark the interrupt ID is handled
- GICC_RPR
- GICC_HPPIR
- GICC_ABPR
- GICC_AIAR
- GICC_AEOIR
- GICC_AHPPIR
- GICC_APRn
- GICC_NSAPRn
- GICC_IIDR
  - GIC CPU interface information about the implementer and version
- GICC_DIR

## Interrupt lines
Interrupts from sources are identified using ID numbers. Each CPU interface can see up to **1020** interrupts.

The interrupts are divided to 3 parts
- SGI
  - Software generate interrupt
- PPI
  - Private peripheral interrupt
  - CPU timer interrupts are belonged to this category
- SPI
  - Shared peripheral interrupt
  - Usually all hardware device's, such as UART and GPIO, interrupt line are belonged to this category

The GIC assigns interrupt ID numbers ID0-ID1019 as follows:
- Interrupt numbers ID0-ID31 are used for interrupts that are private to a CPU interface. These interrupts are banked in the Distributor.  
  - A banked interrupt is one where the Distributor can have multiple interrupts with the same ID.
  - A banked interrupt is identified uniquely by its ID number and its associated CPU interface number. 
  - Of the banked interrupt IDs:
    - ID0-ID15 are used for SGIs
    - ID16-ID31 are used for PPIs
- Interrupt numbers ID32-ID1019 are used for SPIs.
- Interrupt numbers ID1020-ID1023 are reserved for special purposes

# Handling interrupt with GIC
## Initialize
1. Configure distributer
   1. Enable distributer
   2. Setup interrupt line
      1. setup interrupt line priority
      2. enable interrupt
2. Configure CPU interface
   1. Set priority to properly to allow interrupt to forward to this CPU interface
   2. Enable CPU interface

## Handle the interrupt
Once the interrupt is successfully received from the CPU and GIC driver is called to handle the interrupt
1. Read CPU inteface GICC_IAR to ackowledge the interrupt
   1. interrupt state transition from pending to active in GIC
2. Call the specific interrupt handler
3. Write interrupt line ID to GICC_EOIR to mark the end of interrupt and acknowledge GIC the interrupt is handled
   1. interrupt state transition from active to inactive in GIC

For more information about interrupt life cycle in GIC, check [Arm Generic Interrupt Controller Architecture Specification](https://developer.arm.com/documentation/ihi0048/b) chapter 3
# Reference
https://developer.arm.com/documentation/ddi0471/b?lang=en
https://developer.arm.com/documentation/ihi0048/b