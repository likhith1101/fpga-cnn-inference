# `top_pynqz1` architecture and data flow

This guide documents the Verilog top module in `C:\Users\HP\Documents\top_pynq.v`. The top-level module is `top_pynqz1`. The file is generated Verilog (from a Chisel/Scala design), so names such as `Queue_28` are implementation details; the meaningful architectural modules are `TCU`, `Decoder`, `SystolicArray`, `AccumulatorWithALUArray`, `DualPortMem`, and `AXIWrapperTCU`.

> Scope note: the original TCU architecture is described first. A prototype `WinogradF2x2_3x3` extension was appended to the supplied `top_pynq.v`; it is clearly marked as a prototype because its Q8 rounding test currently fails by one LSB in two outputs. It is not ready for a PYNQ bitstream.

## 1. What the top module is

`top_pynqz1` is an instruction-driven tensor compute accelerator for the PYNQ-Z1. Software sends 64-bit instructions to the FPGA. The TCU decodes those instructions, moves 16-bit vector data between external DDR and local memories, runs vector/systolic computation, and returns results to DDR.

It is **not** a fixed CNN-only datapath. CNN convolution, activation, data movement, and matrix-like workloads are intended to be expressed through its instruction/dataflow machinery.

```text
PYNQ software / AXI DMA
          │ 64-bit instruction stream
          ▼
      top_pynqz1
          │
          ▼
     AXIWrapperTCU
          │ converts memory streams to AXI4
          ▼
         TCU
   ┌──────┼────────────────────────────────────────────────────┐
   │      │                                                      │
   │   Decoder ──► dataflow / routing / lock control             │
   │      │                                                      │
   │      ├──► local dual-port vector memory                     │
   │      ├──► 8-lane systolic MAC array                         │
   │      └──► accumulator + 8-lane ALU array                   │
   └────────────────────────────────────────────────────────────┘
          │                               │
          ▼                               ▼
       DRAM0 AXI4                      DRAM1 AXI4
          │                               │
          └──────────── external DDR ─────┘
```

## 2. External top-level interfaces

### Clock and reset

| Signal | Meaning |
|---|---|
| `clock` | Single synchronous accelerator clock. |
| `reset` | **Active-low at the top level.** The top connects `tcu_reset = ~reset`; keep `reset=0` during reset and set it to `1` during normal operation. |

### Instruction stream

| Signal | Width | Direction | Meaning |
|---|---:|---|---|
| `instruction_tdata` | 64 | input | `{ opcode[63:60], flags[59:56], arguments[55:0] }` |
| `instruction_tvalid` | 1 | input | Source asserts when an instruction is available. |
| `instruction_tready` | 1 | output | Accelerator asserts when it accepts an instruction. |
| `instruction_tlast` | 1 | input | Present at the top interface but not used by the shown logic. |

An instruction transfer occurs only when both `instruction_tvalid` and `instruction_tready` are high on a clock edge.

### External-memory interfaces

The original design has two 64-bit AXI4 master ports:

| Interface | Purpose |
|---|---|
| `m_axi_dram0_*` | First external DDR path. |
| `m_axi_dram1_*` | Second external DDR path. |

Each has AXI address (`AW`/`AR`), write-data (`W`), write-response (`B`), and read-data (`R`) channels. Address/data width is 32/64 bits at the top boundary. The TCU internally sees 20-bit logical DRAM addresses and configured address offsets; `AXIWrapperTCU` produces AXI transactions.

The prototype Winograd patch adds a third 64-bit master, `m_axi_winograd_*`. It must be connected through AXI SmartConnect/Interconnect to DDR before it can be synthesized on PYNQ. It is not part of the original generated TCU interface.

## 3. Original TCU microarchitecture

### 3.1 Instruction decoder

`Decoder` receives:

```text
opcode[3:0], flags[3:0], arguments[55:0]
```

and produces ready/valid control streams for:

- local-memory port A and port B;
- external DRAM0 and DRAM1 transfers;
- systolic-array control;
- accumulator/ALU control;
- dataflow routers and lock pools;
- DRAM address-offset/cache configuration;
- timeout, tracepoint, and program-counter state.

The generated decoder visibly recognizes opcodes `0x1`, `0x2`, `0x3`, `0x4`, and `0xF`. Their exact user-facing ISA names are not recoverable safely from generated logic alone; do not assume a particular opcode is "convolution" without the original Tensil instruction documentation or Scala source.

### 3.2 Vector width and numeric representation

The compute and memory interfaces carry **eight 16-bit elements per vector transaction**:

```text
lane 7  lane 6  lane 5  lane 4  lane 3  lane 2  lane 1  lane 0
 16b     16b     16b     16b     16b     16b     16b     16b
────────────────────────────────────────────────────────────────
                         128 bits
```

The RTL treats ALU operands as signed 16-bit values in many places. Scaling (for example Q8 or Q15), rounding, saturation, zero points, and per-channel quantization are software/command-policy responsibilities unless a specific compute operation provides them.

### 3.3 Local vector memories

The design contains two 128-bit dual-port local memories:

| Block | Storage shape | Use |
|---|---|---|
| `DualPortMem` | 2,048 × 128 bits | One local vector-memory bank. |
| `DualPortMem_1` | 8,192 × 128 bits | Larger local vector-memory bank. |

Both ports can transfer one eight-lane 16-bit vector. The supplied RTL instantiates vendor-style modules `bram_dp_128x2048` and `bram_dp_128x8192`; Vivado must provide/infer equivalent synthesizable BRAMs. The behavioral models in `top_pynq_winograd_flow_tb.v` exist only for Icarus simulation.

### 3.4 Systolic array

`SystolicArray` has 8×16-bit input lanes, 8×16-bit weight lanes, and 8×16-bit output lanes. It is controlled through:

| Control | Meaning |
|---|---|
| `load` | Load/advance the array’s data path according to the internal schedule. |
| `zeroes` | Initialize/clear an accumulation condition. |

This is the high-throughput MAC resource. A convolution compiler schedules activation and weight vectors into it; it is not directly a 3×3-window engine.

### 3.5 Accumulator and ALU array

`AccumulatorWithALUArray` combines an accumulator storage/data path with an 8-lane `ALUArray`.

The scalar ALU operation encoding visibly includes operations for pass/clear/register behavior, arithmetic fixed-point MAC variants, absolute value, signed comparisons, minimum, and maximum. It supports the kinds of add/subtract and requantization operations needed around CNN layers, but the instruction encoding selects and sequences them.

### 3.6 Routers, queues, and locks

Queues provide elasticity between units. `LocalRouter` and `HostRouter` select which producers/consumers use a path. `LockPool` prevents conflicting dataflow phases from using the same resources at once. These units are why the TCU can overlap transfers and compute—but they also mean new datapaths should be integrated through the decoder and routing protocol, not connected informally to existing wires.

## 4. Normal original-TCU data flow

For a generic vector/MAC operation, the intended flow is:

```text
1. Software sends instructions over the 64-bit stream.
2. Decoder accepts each instruction and creates control records.
3. AXIWrapperTCU converts DRAM control/data records into AXI4 transactions.
4. Incoming DRAM data is unpacked into eight 16-bit lanes.
5. The router sends vectors to local memory, systolic input, systolic weight, or accumulator input.
6. The systolic array computes MAC data; the accumulator retains partial sums.
7. ALU array optionally applies post-processing such as clamp/ReLU-like max or scaling operations.
8. Results return through local memory/router to the selected DRAM write stream.
9. AXIWrapperTCU packs eight 16-bit lanes into 64-bit AXI beats and completes the write response.
```

Actual overlap depends on the issued instruction program and lock/dataflow state.

## 5. How a sample image reaches a convolution result

This is a conceptual direct-convolution mapping, not a claimed existing instruction listing.

Assume a grayscale 8×8 image, one 3×3 kernel, stride 1, valid padding. The output is 6×6.

```text
Input image in DDR                One output location Y[r,c]

I[r, c] I[r, c+1] I[r, c+2]       3×3 window from DDR/local SRAM
I[r+1,c] ...                       × 3×3 kernel coefficients
I[r+2,c] ...                       → accumulate nine products
                                      → store Y[r,c]
```

For a practical schedule, software packs 16-bit values into 8-lane vectors, fetches windows/weights into local memory, and uses zero/accumulate controls so partial sums are initialized once then accumulated. Output values are stored back to DDR. Eight output channels or positions can be batched where the data layout permits.

## 6. Winograd `F(2×2,3×3)` mapping

### 6.1 Applicability

`F(2×2,3×3)` replaces four direct-convolution output calculations with a transformed computation. It is suitable for **standard dense 3×3 convolution with stride 1 and dilation 1**. It is not a universal convolution algorithm.

It does not directly apply to 1×1 layers, stride-2 layers, dilated layers, depthwise/group layers without special scheduling, pooling, fully connected layers, or arbitrary kernels other than 3×3.

### 6.2 Tile mathematics

For one input channel, one output channel, and one 2×2 output tile:

```text
d : 4×4 activation tile            g : 3×3 kernel

V = Bᵀ d B                         input transform (4×4)
U = G g Gᵀ                         kernel transform (4×4)
M = U ⊙ V                          16 element-wise products
Y = Aᵀ M A                         output transform (2×2)
```

For multi-channel convolution, `M` is accumulated across every input channel before the final inverse transform:

```text
M_total = Σ(input_channel) U[channel] ⊙ V[channel]
Y       = Aᵀ M_total A
```

### 6.3 Worked sample used by the testbench

The flow test uses one 4×4 signed-Q8 tile and one 3×3 signed-Q8 kernel. Raw stored integers are shown below. Q8 real value equals raw integer ÷ 256.

```text
Input tile d, at byte address 0x00000200

 16   32   48   64
 80   96  112  128
144  160  176  192
208  224  240  256

Kernel g, at byte address 0x00000400

 16   32   48
 64   80   96
112  128  144
```

Direct convolution is the reference:

```text
Expected raw Q8 2×2 result

348  393
528  573

Packed DDR word at byte address 0x00000600:
{Y11, Y10, Y01, Y00} = 0x023D_0210_0189_015C
```

### 6.4 Prototype command sequence

The appended prototype intercepts opcode `0x5` at `top_pynqz1`. It does **not** route `0x5` into the original generated `Decoder`.

| Command | Opcode | Flags | Arguments `[31:0]` | Action |
|---|---:|---:|---|---|
| Configure input | `0x5` | `0x0` | byte address of 4×4 tile | Stores input base address. |
| Configure kernel | `0x5` | `0x1` | byte address of 3×3 kernel | Stores kernel base address. |
| Configure output | `0x5` | `0x2` | byte address of 2×2 result | Stores output base address. |
| Start | `0x5` | `0x3` | ignored | Starts the controller. |

The prototype then performs:

```text
Read 4 × 64-bit words  → 16 input values
Read 3 × 64-bit words  →  9 kernel values
Compute transforms and 16 products
Write 1 × 64-bit word  →  4 output values
```

### 6.5 Cycle-level trace from the sample

The flow test prints a readable trace. Its expected structural order is:

```text
C3   configure input address (0x200)
C5   configure kernel address (0x400)
C7   configure output address (0x600)
C9   start operation
C10–C17  AXI reads four input words
C18–C23  AXI reads three kernel words
C24       AXI write-address request
C25       AXI write-data transfer
C26       AXI write-response handshake
C27       controller returns idle
```

The current prototype produces `Y00=347, Y01=393, Y10=527, Y11=573`; the reference is `348,393,528,573`. This is a known one-LSB fixed-point rounding error in the kernel transform. Treat the test as proof of command/AXI sequencing only, not proof of numerical Winograd correctness.

## 7. Verification artifacts and commands

| File | Purpose |
|---|---|
| `winograd_tb.v` | Unit test for only `WinogradF2x2_3x3` plus simple AXI RAM. |
| `top_pynq_winograd_flow_tb.v` | Top-level educational test: instruction stream → `top_pynqz1` → prototype AXI master → DRAM model → output comparison. |
| `top_pynq_winograd_flow_tb.vcd` | Waveform generated by the top-level educational test. |

Run the detailed top-level trace from PowerShell:

```powershell
wsl bash -lc "cd '/mnt/c/Users/HP/Documents/ChatGPT/hardware accelarator' && iverilog -g2012 -s top_pynq_winograd_flow_tb -o /tmp/top_pynq_winograd_flow_tb.vvp '/mnt/c/Users/HP/Documents/top_pynq.v' top_pynq_winograd_flow_tb.v && vvp /tmp/top_pynq_winograd_flow_tb.vvp"
```

## 8. What is required before FPGA deployment

1. Fix and prove the fixed-point Winograd math with randomized positive/negative tests and exact/defined rounding behavior.
2. Test AXI backpressure: vary `ARREADY`, `RVALID`, `AWREADY`, `WREADY`, and `BVALID` independently.
3. Add multi-input-channel accumulation and a deliberate output/bias/requantization policy.
4. Replace/instantiate the simulation-only BRAM models as synthesizable Vivado BRAM IP or inferred RAM.
5. Connect `m_axi_dram0_*`, `m_axi_dram1_*`, and any Winograd AXI port correctly in the Vivado block design. A SmartConnect must have as many `Sxx_AXI` interfaces as connected masters.
6. Verify reset polarity in the final block design: top-level `reset` is active low.
7. Generate a bitstream and compare PYNQ DDR outputs against a Python/NumPy or PyTorch reference.

## 9. Recommended long-term architecture

The prototype is useful for learning, but the clean production approach is to modify the original Scala/Chisel TCU source (if available): add a decoder instruction, local-memory/dataflow routing, a Winograd controller, and systolic/accumulator reuse. Avoid permanently maintaining a hand-edited patch at the end of generated `top_pynq.v`, because regeneration would overwrite or invalidate it.
