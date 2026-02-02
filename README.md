## BOARD_DISC1 – DC Motor Velocity Control (STM32F407)

This firmware controls a DC motor **speed** using **encoder feedback + PID**.  
You can send commands from a PC terminal tool (e.g., **Hercules**) over **UART4** to set speed and tune PID.

## How it works (simple flow)

1. **Startup**
   - Initializes clocks and peripherals.
   - Starts:
     - **TIM2** in encoder mode (position counter)
     - **TIM3/TIM4** PWM outputs (two directions)
     - **UART4 RX DMA** (fixed 10-byte frames)
     - **TIM6** periodic interrupt (heartbeat LED)

2. **Main loop**
   - Reads the current encoder counter from TIM2.
   - Every **10 ms**:
     - Computes `encoder_diff = encoder - last_encoder` (wrap-safe for 32-bit counter)
     - Updates a filtered velocity estimate
     - Runs `PID_velo(desired_velo, current_velo)` to produce `u_signal`
   - Applies PWM:
     - If `u_signal < 0`: PWM on **TIM3**, TIM4 = 0
     - Else: PWM on **TIM4**, TIM3 = 0
     - PWM compare is clamped to avoid `CCR > ARR`
   - If a UART4 DMA frame is ready, parses the command and re-arms DMA.
   - Every **1 second**, sends a status line back to Hercules.

## UART commands (UART4 / Hercules)

UART4 receives **10-byte frames**. Use short ASCII commands and end with `\r`/`\n` if possible (example: `stop\r\n`).

- **`aaaa`**: increase PWM period (`default_PWM`) by 10 (max 5000)
- **`bbbb`**: decrease PWM period by 10 (min 1000)
- **`stop`**: set speed target to 0
- **`rev`**: reverse direction (negate target)
- **`pid:P:I:D`**: update PID gains (example: `pid:1.9:1.8:0.04`)
  - Note: because frames are only **10 bytes**, long numbers may be truncated.
- **`<number>`**: set desired speed (float), valid range **-300 to 300**

## What you will see (status)

Once per second, firmware sends:

```
Status: Speed=..., Target=..., PWM=..., Error=...
