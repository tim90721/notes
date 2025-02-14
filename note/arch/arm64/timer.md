# Introduction
Timer can be divided to 2 types. One is the peripheral hardware timer that is not related to CPU. Another one is the CPU counter which is the main topic of this documentation.

Either peripheral hardware timer or CPU counter, the general hardware operation is like
1. A counter keep going up or down
2. A hardware comparator compare the value of a register with the counter value and an interrupt is generated 
   1. either the counter value reachs 0
   2. or the counter value is larger or equal to the comparator value

# Arm specific timer
There are several timer in Arm architecture. Some of them can only be used in secure environment. Some of them are CPU specific and some of them or global timer.

For different type of the timers, check [Learn the architecture - Generic Timer](https://developer.arm.com/documentation/102379/0104).

No matter which Arm timer you used, they are only 3 types of register for each timer
- <timer>_CTL_ELx
  - Enable/disable timer
  - Mask/unmask timer interrupt
  - Interrupt status
- <timer>_CVAL_ELx
  - Comparator value
- <timer>_TVAL_ELx
  - Write value to this register will add the value to current counter value and propagate the result to <timer>_CVAL_ELx

# Usage
In this example, I will use CNTP as example.
1. Before enabling the timer, configure its comparator value. Otherwise, it will generate an interrupt as soon as we enable the timer
   ```C
   msr  CNTP_TVAL_EL0, x0
   ```
2. Enable the timer and unmask its interrupt
   ```C
   mov  x0, #1
   msr  CNTP_CTL_EL0, x0
   ```
3. Once the interrupt is received, reconfigure the timer comparator value to wait for next tick
   ```C
   msr CNTP_TVAL_EL0, x0
   ```

# Reference
https://developer.arm.com/documentation/102379/0104