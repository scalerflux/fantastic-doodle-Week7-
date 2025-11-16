

# Week 7: BabySoC Physical Design & Post-Route SPEF Generation

## Objective
To perform complete Physical Design for the BabySoC using OpenROAD — covering floorplanning, placement, clock tree synthesis (CTS), routing, and post-route SPEF generation.

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Step 1: Directory Setup](#step-1-directory-setup)
- [Step 2: Copy Source Files](#step-2-copy-source-files)
- [Step 3: Copy Configuration Files](#step-3-copy-configuration-files)
- [Step 4: Create config.mk](#step-4-create-configmk)
- [Step 5: Setup Environment](#step-5-setup-environment)
- [Step 6: Run Physical Design Flow](#step-6-run-physical-design-flow)
- [Results and Analysis](#results-and-analysis)

---

## Introduction

This project implements the complete RTL-to-GDS physical design flow for **VSDBabySoC**, a mixed-signal System-on-Chip that integrates:
- **RISC-V CPU (rvmyth)**: Digital core processor
- **PLL (Phase-Locked Loop)**: Analog IP for clock generation
- **DAC (Digital-to-Analog Converter)**: Analog IP for output conversion

### Why This Task Matters
This task bridges the gap between RTL design and physical chip layout, demonstrating how:
- Standard cells are placed and routed on silicon
- Analog and digital IPs are integrated
- Timing constraints are met through optimization

---

## Prerequisites

**Tools Required:**
- OpenROAD-flow-scripts (installed)
- SKY130 PDK
- VSDBabySoC design files

**Verify Installation:**
```
cd ~/OpenROAD-flow-scripts
ls tools/install/OpenROAD/bin/openroad
ls tools/install/yosys/bin/yosys
```

---

## Step 1: Directory Setup

### What We're Doing
Creating the proper directory structure for the BabySoC design within OpenROAD-flow-scripts framework.

### Why
OpenROAD-flow-scripts expects designs organized in specific locations:
- `flow/designs/sky130hd/vsdbabysoc/` → Design configuration files
- `flow/designs/src/vsdbabysoc/` → Source Verilog and IP files

```
# Navigate to OpenROAD-flow-scripts
cd ~/OpenROAD-flow-scripts/flow/designs/sky130hd

# Create design configuration directory
mkdir -p vsdbabysoc

# Create source files directory
cd ../src
mkdir -p vsdbabysoc

# Verify directories created
ls -la ~/OpenROAD-flow-scripts/flow/designs/sky130hd/ | grep vsdbabysoc
ls -la ~/OpenROAD-flow-scripts/flow/designs/src/ | grep vsdbabysoc
```


---

## Step 2: Copy Source Files

### What We're Doing
Copying all design files from the VSDBabySoC repository to OpenROAD-flow-scripts.

### Why Each File Matters
- **Verilog files (*.v)**: RTL design sources (vsdbabysoc.v, rvmyth.v, clk_gate.v)
- **LEF files**: Abstract layout views of analog IPs for placement
- **LIB files**: Timing characterization models for synthesis and STA
- **GDS files**: Physical layouts of analog macros (PLL, DAC)
- **Include files**: Header files for RISC-V core (sandpiper-generated)


```
# Clone VSDBabySoC repository (if not already done)
cd ~
git clone https://github.com/manili/VSDBabySoC.git
cd VSDBabySoC

# Copy Verilog source files
cp src/module/*.v ~/OpenROAD-flow-scripts/flow/designs/src/vsdbabysoc/

# Copy analog IP GDS files
cp -r src/gds ~/OpenROAD-flow-scripts/flow/designs/src/vsdbabysoc/

# Copy include directory (sandpiper headers)
cp -r src/include ~/OpenROAD-flow-scripts/flow/designs/src/vsdbabysoc/

# Copy LEF files (abstract layout views)
cp -r src/lef ~/OpenROAD-flow-scripts/flow/designs/src/vsdbabysoc/

# Copy LIB files (timing models)
cp -r src/lib ~/OpenROAD-flow-scripts/flow/designs/src/vsdbabysoc/

# Verify all files copied
ls ~/OpenROAD-flow-scripts/flow/designs/src/vsdbabysoc/
```



---

## Step 3: Copy Configuration Files

### What We're Doing
Copying design constraints and configuration files that control the physical design flow.

### Why Each File Matters
- **vsdbabysoc_synthesis.sdc**: Defines timing constraints (clock period, input/output delays)
- **macro.cfg**: Specifies placement locations for PLL and DAC macros
- **pin_order.cfg**: Defines I/O pin placement around chip periphery


```
cd ~/VSDBabySoC

# Copy SDC constraints file
cp src/sdc/vsdbabysoc_synthesis.sdc ~/OpenROAD-flow-scripts/flow/designs/sky130hd/vsdbabysoc/

# Copy macro placement configuration
cp src/layout_conf/vsdbabysoc/macro.cfg ~/OpenROAD-flow-scripts/flow/designs/sky130hd/vsdbabysoc/

# Copy pin order configuration
cp src/layout_conf/vsdbabysoc/pin_order.cfg ~/OpenROAD-flow-scripts/flow/designs/sky130hd/vsdbabysoc/

# Verify configuration files
ls -la ~/OpenROAD-flow-scripts/flow/designs/sky130hd/vsdbabysoc/
```


---

## Step 4: Create config.mk

### What We're Doing
Creating the master configuration file that defines all design parameters for the OpenROAD flow.

### Why This File is Critical
The `config.mk` file controls:
- Design name and platform selection
- File paths for all inputs (Verilog, LEF, LIB, GDS, SDC)
- Physical design parameters (die size, core area, placement density)
- Flow optimization settings


```
cd ~/OpenROAD-flow-scripts/flow/designs/sky130hd/vsdbabysoc
nano config.mk
```

### config.mk Content
```
export DESIGN_NICKNAME = vsdbabysoc
export DESIGN_NAME = vsdbabysoc
export PLATFORM    = sky130hd

# Explicitly list the Verilog files for synthesis
export VERILOG_FILES = $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/vsdbabysoc.v \
                       $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/rvmyth.v \
                       $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/clk_gate.v

export SDC_FILE      = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)/vsdbabysoc_synthesis.sdc

export vsdbabysoc_DIR = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)

export VERILOG_INCLUDE_DIRS = $(wildcard $(vsdbabysoc_DIR)/include/)
export ADDITIONAL_GDS  = $(wildcard $(vsdbabysoc_DIR)/gds/*.gds.gz)
export ADDITIONAL_LEFS  = $(wildcard $(vsdbabysoc_DIR)/lef/*.lef)
export ADDITIONAL_LIBS = $(wildcard $(vsdbabysoc_DIR)/lib/*.lib)

# Clock Configuration
export CLOCK_PORT = CLK
export CLOCK_NET = $(CLOCK_PORT)

# Floorplanning Configuration
export FP_PIN_ORDER_CFG = $(wildcard $(DESIGN_DIR)/pin_order.cfg)

export DIE_AREA   = 0 0 1600 1600
export CORE_AREA  = 20 20 1590 1590

# Placement Configuration
export MACRO_PLACEMENT_CFG = $(wildcard $(DESIGN_DIR)/macro.cfg)
export PLACE_PINS_ARGS = -exclude left:0-600 -exclude left:1000-1600: -exclude right:* -exclude top:* -exclude bottom:*

export TNS_END_PERCENT = 100
export REMOVE_ABC_BUFFERS = 1

# Magic Tool Configuration
export MAGIC_ZEROIZE_ORIGIN = 0
export MAGIC_EXT_USE_GDS = 1

# CTS tuning
export CTS_BUF_DISTANCE = 600
export SKIP_GATE_CLONING = 1

# Allow routing congestion (for completion)
export GRT_ALLOW_CONGESTION = 1
```

### Key Parameters Explained

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `DIE_AREA` | 0 0 1600 1600 | Total chip dimensions (1600μm × 1600μm) |
| `CORE_AREA` | 20 20 1590 1590 | Usable area for cells (leaving 20μm margins) |
| `CTS_BUF_DISTANCE` | 600 | Distance between clock buffers (600μm) |
| `TNS_END_PERCENT` | 100 | Optimize all timing paths (100%) |
| `GRT_ALLOW_CONGESTION` | 1 | Allow flow to continue despite minor routing congestion |


---

## Step 5: Setup Environment

### What We're Doing
Loading environment variables to make OpenROAD tools accessible in the terminal.

```
# Navigate to OpenROAD-flow-scripts root
cd ~/OpenROAD-flow-scripts

# Source environment variables
source ./env.sh

# Verify tools are accessible
yosys -version
openroad -version
```


---

## Step 6: Run Physical Design Flow

### Overview of Stages
The OpenROAD flow executes these stages automatically:
1. **Synthesis** - Convert RTL to gate-level netlist
2. **Floorplanning** - Define chip boundaries and place macros
3. **Placement** - Position standard cells
4. **Clock Tree Synthesis (CTS)** - Build clock distribution network
5. **Routing** - Create metal wire connections
6. **Finishing** - Parasitic extraction (SPEF), metal fill, GDS generation

---

### Stage 1: Synthesis

#### What Happens
Yosys converts Verilog RTL into a gate-level netlist using SKY130 standard cells.


```
cd ~/OpenROAD-flow-scripts/flow
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk synth
```

#### Key Results
- Clock period extracted: **11ns** (from SDC file)
- Design synthesized with SKY130 standard cells
- Output: `results/sky130hd/vsdbabysoc/base/1_synth.v`

---

<img width="1920" height="1243" alt="Screenshot 2025-11-16 at 3 12 16 PM" src="https://github.com/user-attachments/assets/0e3c1224-f1fe-49d3-893f-bbf03d0d0ef6" />


---

### Stage 2: Floorplanning

#### What Happens
- Defines die area (1600×1600 μm) and core area
- Places analog macros (PLL and DAC) based on macro.cfg
- Inserts tap cells (23,625 cells for substrate connections)
- Generates Power Distribution Network (PDN)

```
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk floorplan
```



#### Key Results
- **Design area**: 722,267 μm²
- **Utilization**: 29%
- **Total instances**: 6,605 (standard cells + macros)
- **Tap cells inserted**: 23,625

---

<img width="1920" height="1243" alt="Screenshot 2025-11-16 at 3 12 59 PM" src="https://github.com/user-attachments/assets/e8d5fde9-9871-429e-82fa-84ae650272da" />


---

### Stage 3: Placement

#### What Happens
- **Global placement**: Roughly positions all 29,885 instances
- **Resizing**: Optimizes cell sizes for timing
- **Detailed placement**: Finalizes legal cell positions with optimization

```
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk place
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_place
```

#### Optimization Algorithms Used
1. Set matching optimization
2. Global swaps (2.77% improvement)
3. Vertical swaps (0.39% improvement)
4. Reordering (0.06% improvement)
5. Random improvement (1.80% improvement)
6. Cell flipping (1.39% improvement)

#### Key Results
- **Initial HPWL**: 198,795 μm
- **Final HPWL**: 186,416 μm (6.2% improvement)
- **Zero violations**: No overlaps, no spacing issues ✓
- **Design utilization**: 29%

---
<img width="3840" height="2486" alt="image" src="https://github.com/user-attachments/assets/431b55fe-6cdf-4f0e-bf96-d7ace6dc5f6e" />

---
<img width="3840" height="2486" alt="image" src="https://github.com/user-attachments/assets/0e1b5c0b-e976-4d2d-909d-06af84d0312b" />

---

<img width="1920" height="1243" alt="Screenshot 2025-11-16 at 2 12 53 PM" src="https://github.com/user-attachments/assets/8420ce1a-0f42-4831-8d85-a7ec22e535f2" />

---
<img width="1920" height="1243" alt="Screenshot 2025-11-16 at 2 15 00 PM" src="https://github.com/user-attachments/assets/b97de819-11bd-4455-9103-dbe33ee69ca9" />

---
<img width="1920" height="1243" alt="Screenshot 2025-11-16 at 2 34 08 PM" src="https://github.com/user-attachments/assets/27c478c5-3b5f-47d2-8109-42da47dc1ecc" />

---

### Stage 4: Clock Tree Synthesis (CTS)

#### What Happens
Builds a balanced H-Tree clock distribution network using TritonCTS algorithm.

#### Key Results
- **Clock net**: CLK with 1,144 sink flip-flops
- **Buffers inserted**: 135 clock buffers (sky130_fd_sc_hd__clkbuf_16)
- **Tree depth**: 2-3 buffers (very balanced!)
- **Average wire length**: 1,371.72 μm per sink
- **Timing**: No setup violations, no hold violations ✓


```
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk cts
```

<img width="3840" height="2486" alt="image" src="https://github.com/user-attachments/assets/f6d3930e-b870-46d9-8223-cfcc643f51cf" />

---

### Stage 5: Routing

#### What Happens
- **Global routing**: FastRoute plans wire paths on routing grid
- **Detailed routing**: TritonRoute creates actual metal connections

#### Congestion Issue & Solution
Initial routing encountered congestion (22 overflow points). Fixed by adding to config.mk:
```
export GRT_ALLOW_CONGESTION = 1
```

#### Commands
```
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk route
```

#### Key Results
- **Total wirelength**: 388,918 μm
- **Routed nets**: 6,471
- **Congestion**: Minor overflows allowed for completion



## Results and Analysis

### Final Design Statistics

| Metric | Value |
|--------|-------|
| **Die Area** | 1,600 × 1,600 μm² |
| **Core Utilization** | 30% |
| **Total Instances** | 30,110+ |
| **Clock Period** | 11 ns |
| **Wirelength** | 388,918 μm |
| **Routed Nets** | 6,471 |





## Directory Structure (Final)

```
OpenROAD-flow-scripts/
└── flow/
    ├── designs/
    │   ├── sky130hd/
    │   │   └── vsdbabysoc/
    │   │       ├── config.mk
    │   │       ├── vsdbabysoc_synthesis.sdc
    │   │       ├── macro.cfg
    │   │       └── pin_order.cfg
    │   └── src/
    │       └── vsdbabysoc/
    │           ├── vsdbabysoc.v
    │           ├── rvmyth.v
    │           ├── clk_gate.v
    │           ├── gds/
    │           ├── lef/
    │           ├── lib/
    │           └── include/
    ├── results/sky130hd/vsdbabysoc/base/
    │   ├── 1_synth.v
    │   ├── 2_floorplan.odb
    │   ├── 3_place.odb
    │   ├── 4_cts.odb
    │   ├── 5_route.odb
    │   ├── 6_final.spef  ← TARGET FILE
    │   └── 6_final.gds
    ├── logs/sky130hd/vsdbabysoc/base/
    └── reports/sky130hd/vsdbabysoc/base/
```

---

## References
- [OpenROAD Documentation](https://openroad.readthedocs.io/)
- [OpenROAD Flow Scripts](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)
- [VSDBabySoC Repository](https://github.com/manili/VSDBabySoC)
- [SKY130 PDK Documentation](https://skywater-pdk.readthedocs.io/)

---

```

***

