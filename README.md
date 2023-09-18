# pes_pd
RTL to GDS flow using openlane
<details>
<summary>openlane and sky130 PDK</summary>

### fundametals of chip structure


![267375537-8ba7b58c-0580-43ec-aaed-bc877bb9ff21](https://github.com/Shrachinag/pes_pd/assets/119600435/74cb5f86-83fc-4998-a6b9-56b82820e9c2)
)

![267374545-c2274d58-299e-413e-afb6-d7325d4fc7a7](https://github.com/Shrachinag/pes_pd/assets/119600435/41092398-2b52-4020-a93c-02a52cb5efe0)

![267375712-eb5c48b1-91c2-4f2f-9df3-e929a357cc5e](https://github.com/Shrachinag/pes_pd/assets/119600435/85c13be5-25ad-4620-a06d-f9176d99f8d1)



we call the above as a package and not a chip

1. PADS - the ways signal comes inside or goes outside
2. CORE - all the digital logic recides
3. DIE - size of the chip


The core of the chip will contain two types of blocks:
 - **Foundry IP Blocks** (e.g. ADC, DAC, PLL, and SRAM) = blocks which requires some amount of intelligent techniques to build which can only be designed by foundries.
 - **Macro blocks** (e.g. RISC-V SOC and SPI) = pure digital logic blocks compared to IP's which might require some analog parts. 
 
![182751377-2810d388-21b0-4df1-b1d4-c72176d80d28](https://github.com/Shrachinag/pes_pd/assets/119600435/3fe08263-78f3-4bc0-854e-9dffa5c3c76b)

Open Source Digital ASIC Design requires three open-source components:  
- **RTL Designs** = github.com, librecores.org, opencores.org
- **EDA Tools** = OpenROAD, OpenLANE,QFlow  
- **PDK** = Google + Skywater 130nm Production PDK

**PDK (Process Design Kit)** = A set of data files and documents which serves as the interface between the designer and the fab. This includes cell libraries, IO libraries, design rules (DRC, LVS, etc.)

### OPENLANE RTL to GDSII Flow:
## Synthesis
1. The RTL Level Design is then synthesized using a Logic Synthesizer we use yosys for the same here.
2. The RTL Netlist is then converted into a synthesised netlist where there are details about the standard cells and its implementations.
3.  Yosys takes the RTL design and timing .libs and verilog models of standard cells and converts into a RTL Netlist.
4.  abc does the tehnology mapping to the required skywater-pdk variants

#### Synthesis Strategies
Different strategies can be used to synthesize for the either the least area or the best timing. To analyse this, synthesis exploration utility generates a report showing the effect on delays/timing/area et.,

#### Deign Exploration Utility
This is used to suit the design configuration and generate reports with different metrics to select the best. This is also used for regression testing

#### Design For Test - DFT Insertion
1. in a real chip once fabricated we  need to check for manufacturing defects
2. we cannot test individual blocks in an soc thus we need to design the chip such that it can be tested out later
3. DFT is done using scan chains we do this by modifying the flip flop such that it has extra inputs and we can pass test signals to check the functionality of the flip flop
4. we do this using an open source tool called fault

## Floor Planning 
1. This is done by OpenROAD flow The macros and IPs are placed in the core before proceding further This is called as pre-placement. 
2. Floor planning is done separately for the macros and it is called macro floor planning where in the macros are placed in such a way that they are closer to the inputs/outputs/other macros where more connections are present.
3. To prevent the loading effects de-coupling capacitors are placed so that the logic states are well within the noise margin.
   
## power planning

1. When several blocks tap power from a single source, there is a problem of Voltage Drop at the Vdd and Ground source at the Vss which can again push the logic out of the required noise margin into the undefined state.
2. To mitigate this Vdd and Vss are placed as horizontal and vertical strips in the chip so that the blocks can tap power from the nearest source.
3. in power grid creation
   1. **rings**:vdd and vss rings are formed across core and macro
   2. **stripes**: they carry vdd and vss across the chip
   3. **rails** :connect vdd and vss to the standard cell

## Placement
There are two types of placement. The other required logic is placed optimally. Placement is of two steps

1. Global Placement- finds the optimal position for each cells. These positions are not necessarly correct, cells may overlap
2. Detialed Placement - After Global placement is done minimal alterations are done to correct the issues
4. Clock Tree Synthesis
To ensure minimum skew the Clock is routed optimally through the circuit using different algorithms. This is done in the OpenROAD flow by TritonCTS.

5. Fake Antenna and diode swapping
Long wires acts as antennas and cause accumulation of charges during the fabrication process damaging the transistor. To avoid this bridging is used to pass the wire through different layers or an antenna diode cell is added to leak away the charges

OpenLane approach - Insert Fake Diode to every cell input during placement. This matches the footprint of the library of the antenna diode. The Antenna Checker is run to check for violations, if there are violations then the fake diode is swapped with a real one.
OpenROAD approach - In the global route step, the antenna violation is addressed automatically by inserting an antenan diode OpenLane allows the user to chose either of the above approaches

## Routing
This step is used to implement the interconnect using the different metal layers specified in the PDK. There are two steps

1. Global Routing - This is done inside the OpenROAD flow (FastRoute)
2. Detailed Routing - This is performed using TritonRoute outside the OpenROAD flow after the global routing.
3. Before performing this step the Logic Equivalence Check is performed by Yosys, since OpenROAD does some optimisations the circuit.

## RC Extraction
From the .def file, the parasitic extraction is done to generate the .spef file (Standard Prasitic Exchange Format) which produces an accurate analog model of the circuit by including the parasitic effects due to wires, parasitic capacitances, etc.,

## STA
Static timing analysis (STA) is a method of validating the timing performance of a design by checking all possible paths for timing violations. STA breaks a design down into timing paths, calculates the signal propagation delay along each path, and checks for violations of timing constraints inside the design and at the input/output interface.

## Sign-off Steps
It involves a series of checks and simulations to confirm that the design is ready for fabrication 
and meets the desired functionality, performance, power, and reliability targets

Design Rule Check (DRC) is performed by Magic
Layout Versus Schematic (LVS) is performed by Netgen

## GDSII Extraction
1. Once the layout is verified and passes all checks, the final step is to generate the GDSII file format which represents the complete physical layout of the chip.
2. The GDSII file contains the geometric information necessary for fabrication, including the shapes, layers, masks, and other relevant details.
3. The routed .def file is used my Magic to generate the GDSII file

## intro to openlane
1. OpenLane is an automated RTL to GDSII flow based on several components including OpenROAD, Yosys, Magic, Netgen, CVC, SPEF-Extractor, KLayout and a number of custom scripts for design exploration and optimization It also provides a number of custom scripts for design exploration and optimization
2. Currently, it supports both A and B variants of the sky130 PDK, and the C variant of the gf180mcu PDK, and instructions to add support for other (including proprietary) PDKs are documented
3. OpenLane abstracts the underlying open source utilities, and allows users to configure all their behavior with just a single configuration file

OpenLane integrated several key open source tools over the execution stages:

1. RTL Synthesis, Technology Mapping, and Formal Verification : yosys + abc
2. Static Timing Analysis: OpenSTA
3. Floor Planning: init_fp, ioPlacer, pdn and tapcell
4. Placement: RePLace (Global), Resizer and OpenPhySyn (formerly), and OpenDP (Detailed)
5. Clock Tree Synthesis: TritonCTS
6. Fill Insertion: OpenDP/filler_placement
7. Routing: FastRoute or CU-GR (formerly) and TritonRoute (Detailed) or DR-CU
8. SPEF Extraction: OpenRCX or SPEF-Extractor (formerly)
9. GDSII Streaming out: Magic and KLayout
10. DRC Checks: Magic and KLayout
11. LVS check: Netgen
12. Antenna Checks: Magic
13. Circuit Validity Checker: CVC
    
### OpenLane Directory Hierarchy:

``` 
├── OOpenLane             -> directory where the tool can be invoked (run docker first)
│   ├── designs          -> All designs must be extracted from this folder
│   │   │   ├── picorv32a -> Design used as case study for this workshop
│   |   |   ├── ...
|   |   ├── ...
├── pdks                 -> contains pdk related files 
│   ├── skywater-pdk     -> all Skywater 130nm PDKs
│   ├── open-pdks        -> contains scripts that makes the commerical PDK (which is normally just compatible to commercial tools) to also be compatible with the open-source EDA tool
│   ├── sky130A          -> pdk variant made especially compatible for open-source tools
│   │   │  ├── libs.ref  -> files specific to node process (timing lib, cell lef, tech lef) for example is `sky130_fd_sc_hd` (Sky130nm Foundry Standard Cell High Density)  
│   │   │  ├── libs.tech -> files specific for the tool (klayout,netgen,magic...) 
```

Inside a specific design folder contains a `config.tcl` which overrides the default settings on OpenLANE. These configurations are specific to a design (e.g. clock period, clock port, verilog files...). The priority order for the OpenLANE settings:
1. sky130_xxxxx_config.tcl in `OpenLane/designs/[design]/`
2. config.tcl in `OpenLane/designs/[design]/`
3. Default values in `OpenLane/configuration/`
 
## labwork 

tool files

![267984553-84de87c9-bc7b-487d-9cb0-bc21f8e52575](https://github.com/Shrachinag/pes_pd/assets/119600435/8b50e02e-d2ff-44b8-900d-051c3842112b)


process files

![267987118-f2c8e77b-80a9-4e35-b8bf-0313fc2bd8d8](https://github.com/Shrachinag/pes_pd/assets/119600435/b1f2495d-fd9c-40bc-8cea-955281cfbcb0)

running openlane
**1. Run OpenLANE:**
 - `$ make mount` = Open the docker platform inside the `openlane/!![image](https://github.com/Shrachinag/pes_pd/assets/119600435/70fdc708-9f66-43bf-94db-6a433b5fd6e5)
[image](https://github.com/Shrachinag/pes_pd/assets/119600435/f3af2c8e-4258-420e-a765-ca29a737513a)
`
 - `% flow.tcl -interactive` = run script for automating the whole RTL to GDSII flow but in step by step `-interactive` mode
 - `% package require openlane 0.9` == retrives all dependencies for running v0.9 of OpenLANE  
 
![267988467-3a4e6512-b358-4dae-9a8c-d072a004d91c](https://github.com/Shrachinag/pes_pd/assets/119600435/047eeb29-aebf-42bb-8250-9e49c651ac41)

- `% prep -design picorv32a` = Setup the filesystem where the OpenLANE tools can dump the outputs. This also creates a `run/` folder inside the specific design directory which contains the command log files, results, and the reports dump by each tools. These folders will be empty for now except for lef files generated by this design setup stage. This merged the [cell LEF files](https://teamvlsi.com/2020/05/lef-lef-file-in-asic-design.html) `.lef` and [technology LEF files](https://teamvlsi.com/2020/05/lef-lef-file-in-asic-design.html) `.tlef` generating `merged.nom.lef` inside `run/tmp/`

![267989039-94693ab5-7019-4639-a652-2079dfd768b7](https://github.com/Shrachinag/pes_pd/assets/119600435/0f936846-fdce-4804-baae-dcd6b8c6bf58)


we see runs being executed

![267990212-d9ef08b5-1b59-4b7a-b787-6e42693d5012](https://github.com/Shrachinag/pes_pd/assets/119600435/c5315555-2d49-47ab-b608-f6cf99b54946)

synthesizing picorv32 on openlane
- `% run_synthesis` = Run yosys RTL synthesis, ABC scripts (for technology mapping), and OpenSTA.  


```
Synthesis Report
=================


   Number of wires:               9824
   Number of wire bits:          10206
   Number of public wires:        1512
   Number of public wire bits:    1894
   Number of memories:               0
   Number of memory bits:            0
   Number of processes:              0
   Number of cells:              10104
     sky130_fd_sc_hd__a2111o_2       2
     sky130_fd_sc_hd__a211o_2      101
     sky130_fd_sc_hd__a211oi_2       4
     sky130_fd_sc_hd__a21bo_2       19
     sky130_fd_sc_hd__a21boi_2       7
     sky130_fd_sc_hd__a21o_2       414
     sky130_fd_sc_hd__a21oi_2      127
     sky130_fd_sc_hd__a221o_2       65
     sky130_fd_sc_hd__a221oi_2       1
     sky130_fd_sc_hd__a22o_2       197
     sky130_fd_sc_hd__a22oi_2        2
     sky130_fd_sc_hd__a2bb2o_2      16
     sky130_fd_sc_hd__a311o_2       38
     sky130_fd_sc_hd__a31o_2        90
     sky130_fd_sc_hd__a31oi_2       10
     sky130_fd_sc_hd__a32o_2        89
     sky130_fd_sc_hd__a41o_2         2
     sky130_fd_sc_hd__and2_2       283
     sky130_fd_sc_hd__and2b_2       32
     sky130_fd_sc_hd__and3_2        77
     sky130_fd_sc_hd__and3b_2       76
     sky130_fd_sc_hd__and4_2        46
     sky130_fd_sc_hd__and4b_2        6
     sky130_fd_sc_hd__and4bb_2       3
     sky130_fd_sc_hd__buf_1       2735
     sky130_fd_sc_hd__buf_2         16
     sky130_fd_sc_hd__conb_1       106
     sky130_fd_sc_hd__dfxtp_2     1596
     sky130_fd_sc_hd__inv_2         83
     sky130_fd_sc_hd__mux2_2      1817
     sky130_fd_sc_hd__mux4_2       323
     sky130_fd_sc_hd__nand2_2      250
     sky130_fd_sc_hd__nand2b_2       2
     sky130_fd_sc_hd__nand3_2       18
     sky130_fd_sc_hd__nand3b_2       3
     sky130_fd_sc_hd__nand4_2        2
     sky130_fd_sc_hd__nor2_2       185
     sky130_fd_sc_hd__nor3_2        11
     sky130_fd_sc_hd__nor3b_2        3
     sky130_fd_sc_hd__nor4_2         4
     sky130_fd_sc_hd__nor4b_2        3
     sky130_fd_sc_hd__o2111a_2       1
     sky130_fd_sc_hd__o211a_2      224
     sky130_fd_sc_hd__o211ai_2       6
     sky130_fd_sc_hd__o21a_2       154
     sky130_fd_sc_hd__o21ai_2       94
     sky130_fd_sc_hd__o21ba_2       15
     sky130_fd_sc_hd__o21bai_2       3
     sky130_fd_sc_hd__o221a_2       19
     sky130_fd_sc_hd__o221ai_2       1
     sky130_fd_sc_hd__o22a_2        26
     sky130_fd_sc_hd__o22ai_2        1
     sky130_fd_sc_hd__o2bb2a_2       7
     sky130_fd_sc_hd__o311a_2       31
     sky130_fd_sc_hd__o311ai_2       2
     sky130_fd_sc_hd__o31a_2        21
     sky130_fd_sc_hd__o31ai_2        2
     sky130_fd_sc_hd__o32a_2        14
     sky130_fd_sc_hd__o41a_2         1
     sky130_fd_sc_hd__or2_2        337
     sky130_fd_sc_hd__or2b_2        20
     sky130_fd_sc_hd__or3_2        102
     sky130_fd_sc_hd__or3b_2        17
     sky130_fd_sc_hd__or4_2         29
     sky130_fd_sc_hd__or4b_2         6
     sky130_fd_sc_hd__xnor2_2       78
     sky130_fd_sc_hd__xor2_2        29

   Chip area for module '\picorv32': 102957.494400

```

```
Flop ratio = Number of D Flip flops = 1596  = 0.1579
             ______________________   _____
             Total Number of cells    10104
```


After running synthesis, inside the `runs/[date]/results/synthesis` is `picorv32a_synthesis.v` which is the mapping of the netlist to standard cell library using ABC. The `runs/[date]/reports/synthesis` will contain synthesis statistic reports and static timing analysis reports. The `runs/[date]/synthesis/logs` contains log files for the terminal output dumps for running yosys and OpenSTA.
</details>






<details>
 <summary>Good Floorplan vs Bad Floorplan and intro to library cells </summary>
 
## floor plan stage

### Find height and width of core and die.



1. Core is where the logic blocks are placed and this seats at the center of the die.
2. The width and height depends on dimensions of each standard cells on the netlist.
3. Utilization factor is (area occupied by netlist)/(total area of the core).
4. In practical scenario, utilization factor is 0.5 to 0.6. This is space occupied by netlist only,the remaining space is for routing and more additional cells.
5. Aspect ratio is (height)/(width) of core, so only aspect ratio of 1 will produce a square core shape.
   
### visual understanding 
1. What is the core and die of the chip?
A die which consists of core, is smallsemiconductor material specimen on which the fundamental circuit is fabricted.
We imprint the die multiple times in the chip.

![268518450-19971c29-85d5-4645-baa0-672a105c3284](https://github.com/Shrachinag/pes_pd/assets/119600435/306169c4-6748-4e93-80e6-fa9644cff855)


2. die and core

![268518318-cdeac4a7-b399-41aa-8739-fa2fe9cf7382](https://github.com/Shrachinag/pes_pd/assets/119600435/22b9ef65-617b-49da-9ab1-8f825bd0c308)

let us start with basic netlsit

![268518384-af137b6b-2087-4886-9602-538cd771ad21](https://github.com/Shrachinag/pes_pd/assets/119600435/c5cb29b3-b0dd-4dbb-b3c7-3cc9dd9b2740)


we are mostly interested in the dimensions of the standard cells. Lets start with rough dimensions of Standard cells and Flip flops as 1unit x 1unit, area = 1sq. unit

![268518418-08dd2fa7-4afb-4c8b-819e-7ce8fea2de30](https://github.com/Shrachinag/pes_pd/assets/119600435/d1c74d95-7a5e-4630-959e-969a911b2d81)

lets calculate the area occupied by the netlist on a silicon wafer.
arranging the cells and flip flops

![268518430-db826254-a47e-49a2-b827-239ead80f07c](https://github.com/Shrachinag/pes_pd/assets/119600435/6c041a8a-0a72-4921-bf3e-b1bd3d866870)


The netlist occuping 4sq. units in placed inside the core.
In this case we are at 100% utilization of the core area, because the logial cells occupies the complete are of the core.

![268518471-3a3e6708-ff78-42fe-9cc2-4274017e539e](https://github.com/Shrachinag/pes_pd/assets/119600435/6910013a-1e82-464b-95a5-43b6876a8aa7)

Utilization Factor = (Area Occupied by netlist)/(Total Area of the Core)
Utilization factor = 1

Usually we design with utilization factor of 0.5, 0.6

Aspect Ratio = Height/Width
Aspect Ratio = 1(square chip)

If the aspect ratio is other than 1, it means that the chip is in rectangular shape.

lets take another example,

![268518506-ab897fad-fd92-41cd-a5b9-bfc3d3d4540c](https://github.com/Shrachinag/pes_pd/assets/119600435/c37786d7-800c-4da3-86f3-e3fa57ecf414)

utilization factor = 0.5 (only 50% is being used, 50% is free)
aspect ratio = 0.5 (rectangle chip)

![268518540-cd2ab491-a02b-45f2-b13c-df330070c693](https://github.com/Shrachinag/pes_pd/assets/119600435/71cd3a65-c493-47ed-9d47-806e3a8e54c2)



### Define location of Preplaced Cell.

1. These are reusable complex logicblocks or modules or IPs or macros that is already implemented (memory, clock-gating cell, mux, comparator...) . The placement on the core is user-defined and must be done before placement and routing (thus preplaced cells). The automated place and route tools will not be able to touch and move these preplaced cells so this must be very well defined

2. Surround preplaced cells with decoupling capacitors.The complex preplaced logicblock requires a high amount of current from the powersource for current switching. But since there is a distance between the main powersource and the logicblock, there will be voltage drop due to the resistance and inductance of the wire. This might cause the voltage at the logicblock to be not within the noise margin range anymore (logic is unstable). The solution is to use decoupling capacitors near the logic block, this capacitor will send enough current needed by the logicblock to switch within the noise margin range.

   ![268524244-8c69d4f8-855d-4515-8821-3065816d0d81](https://github.com/Shrachinag/pes_pd/assets/119600435/634e7cfa-024d-4466-8cdd-a907b443e654)


#### decoupling capacitors
1. The purpose of the decoupling capacitor is to charge the circuit. When a switching activity occurs, the decoupling capacitor transfers some of its charge to the circuit.
2. Decoupling capacitors are large capacitors that store electrical charge. They have a voltage across them similar to that of the power supply
3. When a circuit switches, the decoupling capacitor acts as a power source for the circuit, effectively isolating it from the main power supply. During switching events, the decoupling capacitor supplies the necessary current to the circuit
4. to minimize voltage drops, these capacitors are positioned in close proximity to the circuit. They ensure that the circuit receives the required current during switching operations. 
5. During periods of no switching activity, the decoupling capacitor replenishes its charge from the power supply
6. decoupling caps also help in  mitigating noise and maintaining consistent voltage for delicate components.
7. As electronic apparatuses operate at elevated frequencies, abrupt shifts in current demands can incite voltage fluctuations and unwanted noise,
   thereby resulting in performance dilemmas and signal deterioration.
9. Decoupling capacitors, akin to a safeguard, establish a local storehouse of electrical charge that can swiftly respond to these fluctuations.
   Essentially, they act as reservoirs, storing and disbursing electrical energy as required, effectively sieving out undesirable noise and voltage oscillations.
   
### power planning
1. Power planning in integrated circuit (IC) design involves the careful consideration and distribution of power and ground connections to ensure proper functionality and performance of the chip.
2. One important aspect of power planning is the placement of multiple ground (GND) and supply voltage (VDD) points throughout the IC layout.
3. The need for multiple GND and VDD points arises due to several reasons by providing multiple GND and VDD points, the power can be distributed more evenly throughout the chip, reducing the chances of voltage drops and improving overall power delivery efficiency.
4. Ground bounce occurs when there are variations in the voltage levels of different GND points due to transient currents. This current flow to the ground creates an inductive effect,which causes the ground voltage to rise or fall momentarily which can lead to logic's in ckts to change for a moment
#### example of ground bounce
- We need to transfer the value long the red line.Lets assume the read line is a 16-bit bus and i connected to an inverter

![268521206-f56ffe49-f7e3-46d7-a0b8-cf12788665f2](https://github.com/Shrachinag/pes_pd/assets/119600435/1eec87ed-2d64-4e71-874b-a4da226cc825)


- At once all the ones are made zero, and zeros are made ones, due to this there is bump in the ground tap point as all 16-bit are connected to the same ground.

![268521220-50c1d623-389a-4f23-aa66-101d33c11d05](https://github.com/Shrachinag/pes_pd/assets/119600435/3e8a8ba1-3a5f-4685-a09e-27ad9abeb888)

- The phenomenon is called ground bounce and it will eventually settle down.
If the capacitors are charging from low to high, there is a Voltage drop in the Supply Voltage.

![268521237-b429b0d0-a38a-43e3-a44a-e789b0f43c38](https://github.com/Shrachinag/pes_pd/assets/119600435/71f7a178-cf6a-4b24-9f70-496b673eac12)

- This issue is coming because we are having only one power supply, it was having multiple power supplies we would have this issue. Now we are going to have multiple power supplies.

![183446309-a0714ec5-0619-4327-bdfe-890c19cc97e0-3](https://github.com/Shrachinag/pes_pd/assets/119600435/30a75fc3-fd94-41f5-b467-4c740a7055bb)


6. Similarly, power supply noise refers to fluctuations in the VDD levels caused by switching events.
7![image](https://github.com/Shrachinag/pes_pd/assets/119600435/2868f821-ed40-44c5-83e7-74d2276f2e7a)
. By strategically placing multiple GND and VDD points, the impact of ground bounce and power supply noise can be minimized, improving circuit performance and reducing the risk of functional failures.


### pin placement
1. Pin placement in physical design is all about how and where we put the input/output pins on a chip or circuit board. 
2. It's important because it affects how well signals move around, how little they get messed up, and how easy it is to build and test the device.
3. We have to think about things like keeping the signals strong, spreading out power evenly, managing heat, and making sure it fits with standard connectors and packaging.
4. When we do this pin placement right, it makes the electronic system more reliable, easier to build, and more user-friendly.

### Logical Cell Placement Blockage
This makes sure that the automated placement and routing tool does not place any cell on the pin locations of the die.

Below are all 6  steps for floor planning:  

![image](https://user-images.githubusercontent.com/87559347/183446309-a0714ec5-0619-4327-bdfe-890c19cc97e0.png)


### Placement Stage:
1. Bind the netlist to a physical cell with real dimensions. The physical cell will come from a library that can provide multiple options for shapes, dimensions, and delay for same cells. 
2. Next is placement of those physical cells to the floorplan. The flip flops must be placed as near as possible to the input and output pins to reduce timing delay. 
3. Optimize placement to maintain signal integrity. This is where we estimate wirelength and capacitance (C=EA/d) and based on that insert repeaters/buffers. The wirelength will form a resistanace which will cause unnecessary voltage drop and a capacitance which will cause a slew rate that might not be permissible for fast current switching of logic gates. The solution to reduce resistance and capacitance is to insert buffers for long routes that will act as intermediary and separate a single long wire to multilple ones. Sometime we also do abutment where logic cells are placed very close to each other (almost zero delay) if it has to run at high frequency (2GHz). Crisscrossing of routes is a normal condition for PnR since we can use separate metal layer (using vias) for crisscrossed path.
4. After placement optimization, We will setup timing analysis using idle clock (zero delay for wires and has no clock buffer related delays) considering we have not yet done CTS.   

The goal of placement is not yet on timing but on congestion. Also, standard cells are not placed on floorplan stage, it is done on Placement stage. Macros or preplaced cells are the ones placed on floorplan stage.Macros or preplaced cells are placed on floorplan stage.

![image](https://user-images.githubusercontent.com/87559347/183224947-67a29c54-9a18-45a4-bbd1-9132bcebc304.png)  

Placement is done on two stages:
 - Global Placement = placement with no legalizations and goal is to reduce wirelength. It uses Half Perimeter Wirelength (HPWL) reduction model. 
 - Detailed Placement = placement with legalization where the standard cells are placed on stadard rows, abutted, and must have no overlaps    
 


### Lab [Day 2] - Determine Die Area: 

**1. Set configuration variables.** Before running floorplan stage, the configuration variables or switches must be configured first. The configuration variables are on `openlane/configuration`:  

```
.
├── README.md      
├── checkers.tcl
├── cts.tcl
├── floorplan.tcl  
├── general.tcl
├── lvs.tcl
├── placement.tcl
├── routing.tcl
└── synthesis.tcl 

```  

The  `README.md` describes all configuration variables for every stage and the tcl files contain the default OpenLANE settings. All configurations accepted by the current run is on `openlane/designs/picorv32a/runs/config.tcl`. This may come either from (with priority order):
 - PDK specific configuration inside the design folder
 - `config.tcl` inside the design folder
 - System default settings inside `openlane/configurations`

**2. Run floorplan on OpenLane:** `% run floor_plan`

 
**3. Check the results.** The output of this stage is `runs/[date]/results/floorplan/picorv32a.floorplan.def` which is a [design exchange format](https://teamvlsi.com/2020/08/def-file-in-vlsi-design-exchange.html), containing the die area and positions. 
```
...........
DESIGN picorv32a ;
UNITS DISTANCE MICRONS 1000 ;
DIEAREA ( 0 0 ) ( 660685 671405 ) ;
............
```
The die area here is in database units and 1 micron is equivalent to 1000 database units. **Thus area of the die is (660685/1000)microns\*(671405/1000)microns = 443587 microns squared.** 

**4. View the layout on magic**. Open def file using `magic`:  

```
magic -T /home/kunalg123/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech lef read ../../tmp/merged.lef def read picorv32a.floorplan.def
```  
![268436674-d4f96b63-14f5-4e37-b31a-4df1eab6809c-2](https://github.com/Shrachinag/pes_pd/assets/119600435/ca08f802-ea27-4e68-89b5-6a678446f5e8)


To center the view, press "s" to select whole die then press "v" to center the view. Point the cursor to a cell then press "s" to select it, zoom into it by pressing 'z". Type "what" in `tkcon` to display information of selected object. These objects might be IO pin, decap cell, or well taps as shown below.  

![183100900-b3527702-5375-4a4e-ad87-194fce382128](https://github.com/Shrachinag/pes_pd/assets/119600435/ffcba142-6d94-41c8-8111-8dbca82a6e91)

 Useful Magic commands are listed on the [Magic Commands section](https://github.com/AngeloJacobo/OpenLANE-Sky130-Physical-Design-Workshop#magic-commands).

**5 Run placement:** `% run_placement`. This commmand is a wrapper which does global placement (performed by RePlace tool), Optimization (by Resier tool), and detailed placement (by OpenDP tool). It displays hundreds of iterations displaying HPWL and OVFL. The algorithm is said to be converging if the overflow is decreasing. It also checks the legality. 

**6. View the output of this stage**. The output of this stage is `runs/[date]/results/placement/picorv32a.placement.def.` To see actual layout after placement, open def file using `magic`:  

```
magic -T /home/kunalg123/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech lef read ../../tmp/merged.lef def read picorv32a.placement.def
```  

![268436727-b1c696b7-f87f-4515-af2c-d4da71a1569c](https://github.com/Shrachinag/pes_pd/assets/119600435/96cb9735-f116-4555-b689-690c041aaba2)


![268436730-7c1be3d4-6d90-4200-b79b-721b532d4dd3](https://github.com/Shrachinag/pes_pd/assets/119600435/c2e6c145-af82-473e-ae35-fcd789746e7d)


![268436734-e91690d9-ab46-40d7-bb90-919513648956](https://github.com/Shrachinag/pes_pd/assets/119600435/e00e3913-b108-43a5-84ab-f657f525011e)


## Libraary Binding and Placement

1. Netlist binding is the process of mapping the logical representation of a digital design (typically described in a HDL onto a library of standard cells.
2. Every component is mapped to a given shape and the working of these are defined in the library.
3. Then all these shapes from each stage of the netlist are placed onto the floorplan in a efficient way so that delay is minimal and the blocks are closer to their  i/p or o/p pins.

![268438082-ec7863b1-e009-4bb4-8434-52ea8bd4d2f2](https://github.com/Shrachinag/pes_pd/assets/119600435/cba8cced-6201-41ab-860f-25ef8c49a98a)

4. from din2 to ff1 we use buffers because in longer distances the resistance and capacitance of the interconnect causes distortion in signal aka signal integrity is lost thus buffers are used that take the signal and replicate it enhanching signal integrity
5. 


### CELL DESIGN AND CHARACETRIZATION FLOWS

Library is a place where we get information about every cell. It has differents cells with different size, functionality,threshold voltages. There is a typical cell design flow steps.

1. Inputs : PDKS(process design kit) : DRC & LVS, SPICE Models, library & user-defined specs.
2. Design Steps :Circuit design, Layout design (Art of layout Euler's path and stick diagram), Extraction of parasitics, Characterization (timing, noise, power).
3. Outputs: CDL (circuit description language), LEF, GDSII, extracted SPICE netlist (.cir), timing, noise and power .lib files

### Standard Cell Characterization Flow
1. characterization provides the essential data needed to accurately model and simulate the behavior of these components
   
A typical standard cell characterization flow that is followed in the industry includes the following steps:

1. Read in the models and tech files
2. Read extracted spice Netlist
3. Recognise behavior of the cells
4. Read the subcircuits
5. Attach power sources
6. Apply stimulus to characterization setup
7. Provide neccesary output capacitance loads
8. Provide neccesary simulation commands

Now all these 8 steps are fed in together as a configuration file to a characterization software called GUNA. This software generates timing, noise, power models.
These .libs are classified as Timing characterization, power characterization and noise characterization.


### TIMING CHARACTERIZATION

In standard cell characterisation, One of the classification of libs is timing characterisation.

#### Timing threshold definitions 
Timing defintion |	Value
-------------- | --------------
slew_low_rise_thr	| 20% value
slew_high_rise_thr | 80% value
slew_low_fall_thr |	20% value
slew_high_fall_thr |	80% value
in_rise_thr	| 50% value
in_fall_thr |	50% value
out_rise_thr |	50% value
out_fall_thr | 50% value

#### Propagation Delay and Transition Time 

**Propagation Delay** 
The time difference between when the transitional input reaches 50% of its final value and when the output reaches 50% of its final value. Poor choice of threshold values lead to negative delay values. Even thought you have taken good threshold values, sometimes depending upon how good or bad the slew, the dealy might be still +ve or -ve.

```
Propagation delay = time(out_thr) - time(in_thr)
```
**Transition Time**

1. The time it takes the signal to move between states is the transition time ,
2. where the time is measured between 10% and 90% or 20% to 80% of the signal levels.

```
Rise transition time = time(slew_high_rise_thr) - time (slew_low_rise_thr)

Low transition time = time(slew_high_fall_thr) - time (slew_low_fall_thr)
```

</details>

<details>
 <summary>Design library cell using Magic Layout and ngspice </summary>

### customizing openlane configuration

1. Configurations on OpenLANE can be changed on the flight.
2. openlane/config has a readme witch description on all switches go to the floorplan section
3. FP_IO_MODE is the mode to set i/p o/p pin placement strategy
4. For example, to change IO_mode to be not equidistant use `% set ::env(FP_IO_MODE) 2;` on OpenLANE.
5. The IO pins will not be equidistant on mode 2 (default of 1).
6. Run floorplan again via `% run_floorplan` and view the def layout on magic.
7. go to `designs/picorv32a/runs/date/result/floorplan` and use the following command
  `magic -T tech file info lef read picorv32a.floorplan.def & `
9. However, changing the configuration on the fly will not change the `runs/config.tcl`, the configuration will only be available on the current session. To echo current value of variable: `echo $::env(FP_IO_MODE)`


### Designing a Library Cell:
1. SPICE deck = component connectivity (basically a netlist) of the CMOS inverter.
2. SPICE deck values = value for W/L (0.375u/0.25u means width is 375nm and lengthis 250nm). PMOS should be wider in width(2x or 3x) than NMOS. The gate and supply voltages are normally a multiple of length (in the example, gate voltage can be 2.5V)  
3. Add nodes to surround each component and name it. This will be used in SPICE to identify a component.    


**Notes:**
 - Syntax for the PMOS and NMOS descriptiom:
     - `[component name] [drain] [gate] [source] [substrate] [transistor type] W=[width] L=[length]`
 - All components are described based on nodes and its values
 -  cload out 0 10pf means cload is connected b/w node out and 0 and has value 10pf
 - `.op` is the start of SPICE simulation operation where Vin will be sweep from 0 to 2.5 with 0.5 steps
 - `tsmc_025um_model.mod` is the model file containing the technological parameters for the 0.25um NMOS and PMOS
   
The steps to simulate in SPICE:
```
source [filename].cir
run
setplot 
dc1 
plot out vs in 
```  

## SPICE Analysis for Switching Threshold and Propagation Delay:
CMOS is a robust device for logic application as the vtc curve is high for low i/p 
and low for high i/p the robustness of cmos inverter depends on:  

## Switching threshold
1. Vin is equal to Vout. This the point where both PMOS and NMOS is in saturation or kind of turned on, and leakage current is high.
2. If PMOS is thicker than NMOS, the CMOS will have higher switching threshold (1.2V vs 1V) while threshold will be lower when NMOS becomes thicker.
3. It's the point at which this inverter switches between sending out a "0" or a "1" in a computer chip
4. there are two types of tests to analyse cmos inverter behaviour
##### static test
we see how the inverter behaves in a stable state that is
1. how much power it uses
2. how fast it can send a signal
3. how safe it is against errors
#### dynamic test
we see how inverter behaves when it is switching on and off that is
1. how fast can it switch
2. how strong the signals are
3. to catch issues like sudden changes and stuck states

## Propagation delay (rise or fall delay)

DC transfer analysis is used for finding switching threshold. SPICE DC analysis below uses DC input of 2.5V. Simulation operation is DC sweep from 0V to 2.5V by 0.05V steps:
```
Vin in 0 2.5
*** Simulation Command ***
.op
.dc Vin 0 2.5 0.05
```  
Below is the result of SPICE simulation for DC analysis, the line intersection is the switching threshold:  

![187056328-d6f6d5f5-4ce1-4454-9a5a-26be83a84734-2](https://github.com/Shrachinag/pes_pd/assets/119600435/633aa1fd-3e59-4308-887d-d22bd73a320f)



Meanwhile, transient analysis is used for finding propagation delay. SPICE transient analysis uses pulse input: 
1. starts at 0V
2. ends at 2.5V
3. starts at time 0
4. rise time of 10ps
5. fall time of 10ps
6. pulse-width of 1ns
7. period of 2ns  

![187056370-18949899-a158-4307-96d9-d5c06bbeed66](https://github.com/Shrachinag/pes_pd/assets/119600435/b99f34e9-e9e7-47cb-8c7f-faadf3e0ea19)
 

The simulation operation has 10ps step and ends at 4ns:  

```
Vin in 0 0 pulse 0 2.5 0 10p 10p 1n 2n 
*** Simulation Command ***
.op
.tran 10p 4n
```  
Below is the result of SPICE simulation for transient analysis:

![187062659-9e18e9a5-eff4-4d01-804d-cc1e10597486](https://github.com/Shrachinag/pes_pd/assets/119600435/846c6d2e-db74-465a-a788-24db555a6dbd)

 
 ### CMOS Fabrication Process (16-Mask CMOS Process):  
 **1. Selecting a substrate** = Layer where the IC is fabricated. Most commonly used is P-type substrate  
 **2. Creating active region for transistor** = Separate the transistor regions using SiO2 as isolation
  - Mask 1 = Covers the photoresist layer that must not be etched away (protects the two transistor active regions)
  - Photoresist layer = Can be etched away via UV light  
  - Si3N4 layer = Protection layer to prevent SiO2 layer to grow during oxidation (oxidation furnace)  
  - SiO2 layer = Grows during oxidation (LOCOS = Local Oxidation of Silicon) and will act as isolation regions between transistors or active regions  
  
![image](https://user-images.githubusercontent.com/87559347/187062659-9e18e9a5-eff4-4d01-804d-cc1e10597486.png)  

 **3. N-Well and P-Well Fabrication** = Fabricate the substrate needed by PMOS (N-Well) and NMOS (P-Well)  
  - Phosporus (5 valence electron) is used to form N-well  
  - Boron (3 valence electron) is used to form P-Well.  
  - Mask 2 protects the N-Well (PMOS side) while P-Well (NMOS side) is being fabricated then Mask 3 while N-Well (PMOS side) is being fabricated
   
![image](https://user-images.githubusercontent.com/87559347/187099587-4a837f08-b6d3-4cb9-afe6-75ee8d88cfff.png) 

 **4. Formation of Gate** = Gate fabrication affects threshold voltage. Factors affecting threshold voltage includes:    
 
![image](https://user-images.githubusercontent.com/87559347/187111068-874f408a-d41b-4b16-a5f0-49edfced8926.png)

Main parameters are:
  - Doping Concentration = Controlled by ion implantation (Mask 4 for Boron implantation in NMOS P-Well and Mask 5 for Arsenic implantation in PMOS N-Well)
  - Oxide capacitance = Controlled by oxide thickness  (SiO2 layer is removed then rebuilt to the desire thickness)  
  
 Mask 6 is for gate formation using polysilicon layer.
 
![image](https://user-images.githubusercontent.com/87559347/187116601-0ac34212-3622-4719-9309-fca887ad995a.png)
**5. Lightly Doped Drain formation** = Before forming the source and drain layer, lightly doped impurity is added: 
 - Mask 7 for N- implantation (lightly doped N-type) for NMOS 
 - Mask 8 for P- implantation (lightly doped P-type) for PMOS.  
Heavily doped impurity (N+ for NMOS and P+ for PMOS) is for the actual source and drain but the lightly doped impurity will help maintain spacing between the source and drain and prevent hot electron effect and short channel effect. 

![image](https://user-images.githubusercontent.com/87559347/187121868-94dfade0-2c63-4c9c-afef-942ef9662d5a.png)
**6. Source and Drain Formation** = Mask 9 is for N+ implantation and Mask 10 for P+ implantation  
 - Channeling is when implantations dig too deep into substrate so add screen oxide before implantation
 - The side-wall spacers maintains the N-/P- while implanting the N+/P+    
 
![image](https://user-images.githubusercontent.com/87559347/187128442-76d48790-53a0-4ad2-9856-924f3efd33eb.png)

**7. Form Contacts and Interconnects** =  TiN is for local interconnections and also for bringing contacts to the top. TiS2 is for the contact to the actual Drain-Gate-Source. Mask 11 is for etching off the TiN interconnect for the first layer contact. 

![image](https://user-images.githubusercontent.com/87559347/187141267-b043152d-0a76-4101-90ec-82c9adcc64e2.png)

**8. Higher Level Metal Formation** = We need to planarize first the layer via CMP before adding a metal interconnect. Aluminum contact is used to connect the lower contact to higher metal layer. Process is repeated until the contact reached the outermost layer.
 - Mask 12 is for first contact hole
 - Mask 13 is for first Aluminum contact layer
 - Mask 14 is for second contact hole
 - Mask 15 is for second Aluminum contact layer. Mask 16 is for making contact to topmost layer. 
 
![image](https://user-images.githubusercontent.com/87559347/187158161-4d230654-5102-4225-8e58-d6d8ed950990.png)

### Layout and Metal Layers:

When polysilicon crosses N-diffusion/P-diffusion (diffusion is also called implantation), then an NMOS/PMOS is created. [Explained here](https://electronics.stackexchange.com/questions/223973/why-diffusions-in-cmos-cad-tool-magic-is-continuous) is the reason why the diffusion layer of source and drain "seems" to be connected under the polysilicon (diffusion layer for source and drain supposedly be separated).


The first layer is local-interconnect layer or local-i then metal 1 to 5. [Here is the process stack diagram](https://skywater-pdk.readthedocs.io/en/main/rules/assumptions.html) of sky130nm PDK. Metal 1 is for Power and Ground lines. `Nsubstratecontact` connects the N-well to locali. `licon` connects the locali to metal1.Locali is for local connections of cells. 

The layer hierarchy for NMOS is: Psubstrate -> Psubstrate Diffusion (psd) -> Psubstrate Contact (psc) -> Local-interconnect (li) -> Mcon -> Metal1. For poly: Poly -> Polycontact -> Locali. P-substrate diffusion an N-substrate diffusion is also referred to as P-tap and N-tap. 

The output of the layout is the LEF file. [LEF (Library Exchange Format)](https://teamvlsi.com/2020/05/lef-lef-file-in-asic-design.html) is used by the router tool in PnR design to get the location of standard cells pins to route them properly. So it is basically the abstract form of layout of a standard cell. `picorv32a/runs/[DATE]/tmp` contains the merged lef files (cell LEF and tech LEF). Notice how metal layer directon (horizontal or vertical) is alternating. Also, metal layer width and thickness is increasing. 

### Magic Commands:  
[Here is a great video guide](https://www.youtube.com/watch?v=RPppaGdjbj0) on layout using Magic. And [here is the Magic website](http://opencircuitdesign.com/magic/) with tutorials.
- Left click = lower-left corner of box  
- Right click = upper-right corner of box  
- "z" = zoom in, "Z" = zoom out, "ctrl + z" = zoom into the box 
- Middle click on empty area will turn the box into empty (similar to erasing it)
- "s" three times will select all geometries electrically connected to each other  
- `:box` = display parameters of selected box  
- `:grid` 0.5um 0.5um = turn on/off and set grid   
- `:snap user` = snap based on current grid  
- `:help snap` = display help for command  
- `:drc style drc(full)` = use all DRC when doing DRC checking
- `:paint poly` = paint "poly" to current box
- `:drc why` = show drc violation inside selected area (white dots are DRC violations )
- `:erase poly` = delete poly inside the box
- `:select area` = select all geometries inside the box
- `:copy n 30` = copy selected geometries to North by 30 grid steps
- `:move n 1` = move selected geometries to North by 1 step ("." to move more, "u" to undo)  
- `: select cell _08555_` = select a particular cell instance (e.g. cell \_08555_ which can be searched in the DEF file)
- `:cellname allcells` = list all cells in the layout
- `:cellname exists sky130_fd_sc_hd__xor3_4` = check if a cell exists 
- `:drc why` = show DRC violation and also the DRC name which can be referenced from [Sky130 PDK Periphery Rules](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html#rules-periphery--page-root).

![image](https://user-images.githubusercontent.com/87559347/187588800-f083e5a5-2f22-4670-8a69-93d222794d27.png)


### Lab Part 1 [Day 3] - Slew Rate and Propagation Delay Characterization:

The task is to characterize a sample inverter cell by its slew rate and propagation delay.  

1. Clone [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign). Copy the techfile `sky130A.tech` from `pdks/sky130A/libs.tech/magic/` to directory of the cloned repo. Below are the contents of `vsdstdcelldesign/libs/`:


2. View the mag file using magic `magic -T sky130A.tech sky130_inv.mag &`:  the below image is the layout of the inverter
![268459722-f10fb291-02eb-4896-814e-9638d512d26e](https://github.com/Shrachinag/pes_pd/assets/119600435/de597408-10eb-4589-a226-0d2a85d405b9)



4. We can get to know the details of the inverter by hovering the mouse cursor over it and pressing 's' on the keyboard.
Then we can type **what** in the tkcon terminal

![268459894-ca9955a6-5aae-48a2-8e60-03cf13517b2a](https://github.com/Shrachinag/pes_pd/assets/119600435/32592e95-36d4-47e9-b070-81206967b14c)


6. Pressing 's' three times will show what parts are connected to the selected part
7. LEF represents abstract component data in a machine-readable format for IC libraries, while layout is the physical geometric arrangement of these components on a semiconductor chip.
8. DRC errors in magic will be highlighted with white dotted lines:
  ![268460294-ddf4ee1c-9a2f-43f9-b750-887eb6fedfe0](https://github.com/Shrachinag/pes_pd/assets/119600435/962b51aa-da72-4f24-bbaf-eba79790739f)


 ### extracting spice netlist
 
1. Make an extract file `.ext` by typing `extract all` in the tkon terminal. 
2. Extract the `.spice` file from this ext file by typing `ext2spice cthresh 0 rthresh 0` 
3. cthresh 0 rthresh 0 -> this is done to copy the parasitic capacitances
4. then `ext2spice` in the tkcon terminal.
![268460358-c893924b-7085-4d70-adce-4079b0f5a4fa](https://github.com/Shrachinag/pes_pd/assets/119600435/5331f101-10f0-45ef-8d12-8f0f7b4822f1)


5. we can now see that sky130_inv.spice file has been created

## sky130 tech file labs

1. let us first find the dimensions of a box in the layout window
2. We can use 'g' on the keyboard to activate the grid and after selecting a grid by right clicking on the mouse, we type box in tkcon window to check the minimum value of the layout window.
   
3. We then modify the spice file to be able to plot a transient response:

```
* SPICE3 file created from sky130_inv.ext - technology: sky130A

.option scale=0.01u
.include ./libs/pshort.lib
.include ./libs/nshort.lib

* .subckt sky130_inv A Y VPWR VGND
M0 Y A VGND VGND nshort_model.0 ad=1435 pd=152 as=1365 ps=148 w=35 l=23
M1 Y A VPWR VPWR pshort_model.0 ad=1443 pd=152 as=1517 ps=156 w=37 l=23
C0 A VPWR 0.08fF
C1 Y VPWR 0.08fF
C2 A Y 0.02fF
C3 Y VGND 0.18fF
C4 VPWR VGND 0.74fF
* .ends

* Power supply 
VDD VPWR 0 3.3V 
VSS VGND 0 0V 

* Input Signal
Va A VGND PULSE(0V 3.3V 0 0.1ns 0.1ns 2ns 4ns)

* Simulation Control
.tran 1n 20n
.control
run
.endc
.end
```  

4. Open the spice file by typing `ngspice sky130A_inv.spice`.
5. Generate a graph using `plot y vs time a` :  

![268460358-c893924b-7085-4d70-adce-4079b0f5a4fa](https://github.com/Shrachinag/pes_pd/assets/119600435/9f033b1b-7c93-4d7a-8539-011337d546a8)


Using this transient response, we will now characterize the cell's slew rate and propagation delay:  
- Rise Transition [output transition time from 20%(0.66V) to 80%(2.64V)]:
**Tr_r = 2.19981ns - 2.15739ns = 0.04242 ns**  
![268467214-502051a8-3f59-4d7d-bc12-6decc8dde6fe](https://github.com/Shrachinag/pes_pd/assets/119600435/b28cbb3d-a054-40af-80b8-157f93105400)



- Fall Transition [output transition time from 80%(2.64V) to 20%(0.66V)]:
- **Tr_f = 4.0672ns - 4.04007ns = 0.02713ns**   
![image](https://user-images.githubusercontent.com/87559347/188260236-4cc5d4c7-654a-4600-a277-9f6c1df63b11.png)


- Rise Delay [delay between 50%(1.65V) of input to 50%(1.65V) of output]:
**D_r = 2.18197ns - 2.15003ns = 0.03194ns**   
![image](https://user-images.githubusercontent.com/87559347/188261194-395c7cfd-caea-4efa-a670-310cb30ff6a2.png)


- Fall Delay [delay between 50%(1.65V) of input to 50%(1.65V) of output]:
**D_f = 4.05364ns - 4.05001ns =0.00363ns**  
![image](https://user-images.githubusercontent.com/87559347/188261518-792d3e99-6a5a-423d-9309-62287c608ec0.png)


### Lab Part 2 [Day 3] - Fix Tech File DRC via Magic:
 
Read through [this site about tech file](http://opencircuitdesign.com/magic/techref/maint2.html). All technology-specific information comes from a technology file. This file includes such information as layer types used, electrical connectivity between types, design rules, rules for mask generation, and rules for extracting netlists for circuit simulation. 
Read through also [this site on the DRC rules for SKY130nm PDK](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html#rules-periphery--page-root)

1. Download the [lab contents from this site](opencircuitdesign.com/open_pdks/archive/drc_tests.tgz). Extract the tarball. Inside the `drc_tests/` are the `.mag` layout files and the `sky130A.tech`.
2. Commands to open magic 
```
magic -d XR
```
Then we open the met3.mag file
![267716815-e746a4b8-cd4b-442e-8916-2bc8fdce4c9e](https://github.com/Shrachinag/pes_pd/assets/119600435/21da06ec-bc9f-4d72-8afd-d0e0e9e8e8c1)


To check which DRC rule is being violated select area and type drc why in tkcon 

![267718678-be24dcd6-9893-4c5a-b2d1-5b2061b9370c](https://github.com/Shrachinag/pes_pd/assets/119600435/99f99ac7-370a-4a01-a9cc-7d4d0b0805ed)

to add contact cuts 
add met3 contact by selecting area and clicking on m3contact using middle mouse button.
then type ``` cif see VIA2``` in tkcon prompt

![268468449-a27f74cd-0cbe-4b9c-8651-ecbd7941fb6a](https://github.com/Shrachinag/pes_pd/assets/119600435/7369d06b-303b-432f-8c14-fc8de108abf5)



## fixing drc errors

1. Open magic with `poly.mag` as input: `magic poly.mag`. Focus on `Incorrect poly.9` layout.
2. As described on the poly.9 [design rule of SKY130 PDK](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html#poly), the spacing between polyresistor with poly or diff/tap must at least be 0.480um.
3. Using `:box`, we can see that the distance is 0.250um YET there is no DRC violations shown. Our goal is to fix the tech file to include that DRC.  
![188370620-7e802ce0-cd15-4385-9b73-d8f5ee5fe8ae](https://github.com/Shrachinag/pes_pd/assets/119600435/3d3551c0-608a-4f7c-bcec-c25a8ea58428)

4. Open `sky130A.tech`. The included rules for poly.9 are only for the spacing between the n-poly resistor with n-diffusion and the spacing between the p-poly resistor with diffusion.
5. We will now add new rules for the spacing between the **poly resistor with poly non-resistor**,
6. highlighted green below are the two added rules.
7. On the left is the rule for spacing between n-poly resistor with poly non-resistor and on the right is the rule for the spacing between the p-poly resistor with poly non-resistor. The `allpolynonres` is a macro under `alias` section of techfile. 
![187588800-f083e5a5-2f22-4670-8a69-93d222794d27](https://github.com/Shrachinag/pes_pd/assets/119600435/fdf70583-2f4c-4776-bcdb-7d47bd0383ef)


8. Run `tech load sky130A.tech` then `drc check` in tkcon to reload the tech file.
9.  The new DRC rules will now take effect notice the white dots on the poly indicating the design rule violations.
10.   Command `drc find` to iterate in each violations.  
![188370620-7e802ce0-cd15-4385-9b73-d8f5ee5fe8ae](https://github.com/Shrachinag/pes_pd/assets/119600435/85b35c64-c517-49f6-8439-facbc16c277e)


11. Next, notice below that there are violations between N-substrate diffusion with the polyresistors (from left: npolyres, ppolyres, xpolyres) which is good. But between npolyres with P-substrate diffusion, there is no violation shown. 
![188421029-0f94f6c8-8fc7-4aab-b895-05de5de40f7c](https://github.com/Shrachinag/pes_pd/assets/119600435/91ce4b91-df1c-4e08-95de-125417c931e2)

12. To fix that, just modify the tech file to include not only the spacing between npolyres with N-substrate diffusion in poly.9 but between **npolyres and all types of diffusion**. `alldif` is also a macro under `alias` section. Load the tech file again, the new DRC will now take effect.  
![188384339-225f2a84-8aca-44c6-b742-272448051fc9](https://github.com/Shrachinag/pes_pd/assets/119600435/442f2418-94db-4655-85aa-1ab1a8209ade)

![188421488-3d84c048-06b3-46ac-9816-513dd7c721f2-2](https://github.com/Shrachinag/pes_pd/assets/119600435/cf7cffdb-7b99-47e2-bfb3-6a7d86948da0)



**DRC Error as Geometrical Construct**
- We open the nwell.mag file.


- We type the below commands in the tkcon terminal
~~~
cif ostyle drc # op of this is " CIF output style is now "drc ""
cif see dnwell_shrink
cif see nwell_missing
~~~

- The following is displayed

![268321992-40097614-0dfe-4479-b85c-4418e7b43787](https://github.com/Shrachinag/pes_pd/assets/119600435/c724f8a3-042d-4cb8-a6f8-487052e0f1ff)

**Find Missing or Incorrect Rules and Fix Them**
1.the below layout violates the rule
~~~
all n-wells will contain metal-contacted tap
(rule checks only for licon on tap) . Rule exempted
from high voltage cells inside UHV1
~~~
![image](https://github.com/AniruddhaN2203/pes_pd/assets/142299140/338b998a-9a6b-4485-8d2b-095fbbcf3d83)

- As we can see this is an incorrect implementation and the above rule is violated.

![image](https://github.com/AniruddhaN2203/pes_pd/assets/142299140/d689590d-b9a7-43b2-b1da-72e95a9f4dcb)

![image](https://github.com/AniruddhaN2203/pes_pd/assets/142299140/71d62451-245b-4010-8246-234727062c8b)
- We make the following changes

![268331118-08900ffa-8f91-4550-be8c-375b4cb42866](https://github.com/Shrachinag/pes_pd/assets/119600435/4a87ed85-92fa-4203-8ebf-4e2e2bf6d229)

- Now we select the nwell.4 and type the following commands
```
tech load sky130A.tech
drc check
drc style drc(full)
drc check
```
![268331428-dbb58cd3-4845-4d28-a22b-0228b8260cf6](https://github.com/Shrachinag/pes_pd/assets/119600435/99d2e8ee-7d39-4aaf-98e1-c74a755adda8)


- As we can see the error still persists
- We can fix it by the following method.

![268331631-4bc70446-9a20-47cd-95b9-d850e170887a](https://github.com/Shrachinag/pes_pd/assets/119600435/8bf9442f-b7ba-4089-9f21-efd4535d2d1b)

- Select the existing nwell.4 and make a copy of it by selecting it and clicking 'c'.
- Now select a small area on the nwell.4 and add an 'nsubstratecontact' by hovering over it and clicking middle mouse button.


</details>



<details>
<summary>Pre-layout Timing Analysis and Importance of Good Clock Tree</summary>

#### reusing previous runs
To run previous flow, add tag to prep design:
```
prep -design picorv32a -tag [date]
```

### Lab Part 1 [Day 4] - Extracting the LEF File:   

PnR tool does not need all informations from the `.mag` file like the logic part but only PnR boundaries, power/ground ports, and input/output ports. This is what a [LEF file](https://teamvlsi.com/2020/05/lef-lef-file-in-asic-design.html) actually contains. So the next step is to extract the LEF file from Magic. But first, we need to follow guidelines of the PnR tool for the standard cells:
 - The input and output ports lies on the intersection of the horizontal and vertical tracks (ensure the routes can reach that ports). 
 - The width of the standard cell must be odd multiple of the tracks horizontal pitch and height must be odd multiples of tracks vertical pitch   
 
 To check these guidelines, we need to change the grid of Magic to match the actual metal tracks. The `pdks/sky130A/libs.tech/openlane/sky130_fd_sc_hd/tracks.info` contains those metal informations.   

1. Use `grid` command inside the tkon terminal to match the tracks informations:  

![188419121-ce050fc7-6984-4266-9b24-47002934fc83](https://github.com/Shrachinag/pes_pd/assets/119600435/83d1cd77-c7ba-4a1e-b30b-927dfb100908)


The grids show where the routing for the local-interconnet layer can only happen, the distance of the grid lines are the required pitch of the wire. Below, we can see that the guidelines are satisfied:  

![183273195-485b64e0-fbb4-4c2b-85bf-6e578f7cc5df](https://github.com/Shrachinag/pes_pd/assets/119600435/7a9ce1af-c38f-4d60-8ead-3528b698acf0)


2. Next, we will extract the LEF file. The LEF file contains the cell size, port definitions, and properties which aid the placer and router tool. With that, the ports definition, port class, and port use must be set first. The instructions to set these definitions via Magic are on the [vsdstdcelldesign repo](https://github.com/nickson-jose/vsdstdcelldesign#create-port-definition). 

3. Next, save the mag file with a new filename `save sky130_myinverter.mag`. Then type `lef write` on the tcon terminal. It will generate a LEF file with same name as the magfile `sky130_myinverter.lef`. Inside that LEF file is:  
![268523880-b2c2f90a-c86e-4b16-86b5-960bd8f2dc84](https://github.com/Shrachinag/pes_pd/assets/119600435/cd41975e-845e-492a-a748-83b59d66335c)



### Lab Part 2 [Day 4] - Plug-in the Customized Inverter Cell to OpenLane:

Inside `pdks/sky130A/libs.ref/sky130_fd_sc_hd/lib/` are the [liberty timing files](https://teamvlsi.com/2020/05/lib-and-lef-file-in-asic-design.html) for SKY130 PDK which contains the timing and power parameters for each cell needed in STA. It can either be slow, typical, fast with different different supply voltages (1v80, 1v65, 1v95, etc.). These are the so called [PVT corners](https://chipedge.com/what-are-pvt-corners-in-vlsi/). The library name `sky130_fd_sc_hd__ss_025C_1v80` describes the PVT corner as slow-slow (delay is maximum), 25° Celsius temperature, at 1.8V power supply. Timing and power parameter of a cell is obtained by simulating the cell in a variety of operating conditions (different corners) and these data are represented in the liberty file. 

The liberty file characterizes all cells and is used by the ABC script during synthesis stage which maps the generic cells to the actual standard cells available in the liberty file.  Same cell functionality but different sizes are available inside the library. As shown below, both are inverter cells but one with size-1 and another with size-4. Notice how size-4 inverter is simply four instances of size-1 inverter:

![188778191-1ca86454-98eb-4f95-8e1d-e5276a4edc40](https://github.com/Shrachinag/pes_pd/assets/119600435/abd20544-7560-4bcd-aeb2-744e168ae249)


Provided inside the cloned `vsdstdcelldesign` are the liberty files containing the customized inverter cell.

1. Copy the extracted lef file `sky130_myinverter.lef` and the liberty files `sky130*.lib` from `/openlane/vsdstdcelldesign/libs` to the src directory of picorv32a. Open each liberty files then change the cell name `sky130_vsdinv` to `sky130_myinverter` to match the new LEF file cell name.


2. Add the folowing to `config.tcl` inside the picorv32a:    
```  
set ::env(LIB_SYNTH) "$::env(OPENLANE_ROOT)/designs/picorv32a/scr/sly130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/scr/sly130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/scr/sly130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL) "$::env(OPENLANE_ROOT)/designs/picorv32a/scr/sly130_fd_sc_hd__typical.lib"

set ::env(EXTRA_LEFS) [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

This sets the liberty file that will be used for ABC mapping of synthesis (`LIB_SYNTH`) and for STA (`_FASTEST`,`_SLOWEST`,`_TYPICAL`) and also the extra LEF files (`EXTRA_LEFS`) for the customized inverter cell. The whole `config.tcl` then is:  

![188798219-0b11e661-97f9-4960-b3a7-39aa43c19281](https://github.com/Shrachinag/pes_pd/assets/119600435/719a3d88-38c1-4a3d-803d-a7ca1a30a8cf)



3. Run docker and prepare the design picorv32a. Plug the new lef file to the OpenLANE flow via:  

```
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
```  

4. Next `run_synthesis`. Below is the synthesis statistics report `runs/[date]/reports/synthesis/1-synthesis.AREA_0.stat.rpt` after the run, and as we can see `sky130_myinverter` cell is successfully included in the design!  

![image](https://user-images.githubusercontent.com/87559347/188657588-5686cf61-4978-4842-bbf6-b0c01b111c12.png)

HOWEVER, looking  at the STA log `runs/[date]/logs/synthesis/sta.log`, we are not meeting timing. Our neext goal is to solve this negative slack:

![image](https://user-images.githubusercontent.com/87559347/188801207-1972f34d-80cd-4178-ae3c-e8e49bc0d841.png)



### Delay Table:  

In order to avoid large skew between endpoints of a clock tree (signal arrives at different point in time):
 - Buffers on the same level must have same capacitive load to ensure same timing delay or latency on the same level. 
 - Buffers on the same level must also be the same size (different buffer sizes -> different W/L ratio -> different resistance -> different RC constant -> different delay).    
 
 ![image](https://user-images.githubusercontent.com/87559347/188773408-e503023f-0288-4993-a68a-5f20bccb886c.png)


Buffers on different level will have different capacitive load and buffer size but as long as they are the same load and size on the same level, the total delay for each clock tree path will be the same thus skew will remain zero. **This means different levels will have varying input transition and output capacitive load and thus varying delay.** 

Delay tables are used to capture the timing model of each cell and is included inside the liberty file. The main factor in delay is the output slew. The output slew in turn depends on **capacitive load** and **input slew**. The input slew is a function of previous buffer's output cap load and input slew and it also has its own transition delay table.

![image](https://user-images.githubusercontent.com/87559347/188783693-423bd170-dd0b-4f2f-9652-8fae9418df31.png)

Notice how skew is zero since delay for both clock path is x9'+y15.

### Lab Part 3 [Day 4] - Fix Negative Slack:

1. Let us change some variables to minimize the negative slack. We will now change the variables "on the flight". Use `echo $::env(SYNTH_STRATEGY)` to view the current value of the variables before changing it:  
```
% echo $::env(SYNTH_STRATEGY)
AREA 0
% set ::env(SYNTH_STRATEGY) "DELAY 0"
% echo $::env(SYNTH_BUFFERING)
1
% echo $::env(SYNTH_SIZING)
0
% set ::env(SYNTH_SIZING) 1
% echo $::env(SYNTH_DRIVING_CELL)
sky130_fd_sc_hd__inv_2
```  
With `SYNTH_STRATEGY` of `Delay 0`, the tool will focus more on optimizing/minimizing the delay, index can be 0 to 3 where 3 is the most optimized for timing (sacrificing more area). `SYNTH_BUFFERING` of 1 ensures cell buffer will be used on high fanout cells to reduce delay due to high capacitance load. `SYNTH_SIZING` of 1 will enable cell sizing where cell will be upsize or downsized as needed to meet timing. `SYNTH_DRIVING_CELL` is the cell used to drive the input ports and is vital for cells with a lot of fan-outs since it needs higher drive strength (larger driving cell needed).

2. Below is the log report for slack and area. The area becomes bigger (from 98492 to 103364) but no negative slack anymore (from -1.2ns to +0.35ns)!  
![189464181-d8649d12-e4ef-4cb6-afab-8a305787dd72](https://github.com/Shrachinag/pes_pd/assets/119600435/00b7ae8e-e85b-4dab-97b2-da7ae64797bd)

3. Next, we do `run_floorplan` HOWEVER:  
![image](https://github.com/Shrachinag/pes_pd/assets/119600435/06534a19-4c0b-4083-aeba-2353a289c9d2)
   
The solution for this error is found on [this issue thread](https://github.com/The-OpenROAD-Project/OpenLane/issues/1307). `basic_macro_placement` command is failing since `EXTRA_LEFS` variable inside `config.tcl` is assumed as a macro which is not. The temporary solution is to comment call on `basic_macro_placement` inside the `OpenLane/scripts/tcl_commands/floorplan.tcl` (this is okay since we are not adding any macro to the design). 

4. After that `run_placement`, another error will occur relating to `remove_buffers`, the solution is to comment the call to `remove_buffers_from_nets` in `OpenLane/scripts/tcl_commands/placement.tcl`. After successfully running placement, `runs/[date]/results/placement/picorv32.def` will be created.  

### Lab Part 4 [Day 4] - Locating the Custom Inverter Cell in Layout:  
1. Search for instance of cell `sky130_myinverter` inside the DEF file after placement stage: `cat picorv32.def | grep sky130_myinverter`:  
![189475352-f74731e2-6ef8-4620-a3a9-16c31d326c82](https://github.com/Shrachinag/pes_pd/assets/119600435/d57a8119-db16-421c-8f0d-0028c88ccd67)



2. Open the def file via magic: 
```
magic -T /home/angelo/Desktop/OpenLane/pdks/sky130A/libs.tech/magic/sky130A.tech lef read ../../tmp/merged.nom.lef def read picorv32.def &
```
Select a single `sky130_myinverter` cell instance from the list dumped by grep (e.g. \_07237\_). On tkcon, command `% select cell _07237_` then ctrl+z to zoom into that cell. As shown below, our customized inverter cell `sky130_myinverter` is sucessfully placed on the core. Use `expand` on tkon to show the footprint of the cell and notice how the power and ground of sky130_myinverter overlaps the power and ground pins of its adjacent cells.  

![189507538-6d0c1b46-dc45-4768-b00c-01e8336ab518](https://github.com/Shrachinag/pes_pd/assets/119600435/36040310-54d0-4a05-a09b-82de02b625a2)




### Timing Analysis (Pre-Layout STA using Ideal Clocks):
Pre-layout STA will not yet include effects of clock buffers and net-delay due to RC parasitics (wire delay will be derived from PDK library wire model).    
![image](https://user-images.githubusercontent.com/87559347/189510818-050c6b22-a319-4969-a23e-c82c57ebd4ff.png)  

Setup timing analysis equation is:  
```
Θ < T - S - SU
```  

- Θ =  Combinational delay which includes clk to Q delay of launch flop and internal propagation delay of all gates between launch and capture flop  
- T = Time period, also called the required time
- S = Setup time. As demonstrated below, signal must settle on the middle (input of Mux 2) before clock tansists to 1 so the delay due to Mux 1 must be considered, this delay is the setup time. 
![image](https://user-images.githubusercontent.com/87559347/189511212-8e1ea86f-b2d6-4a68-9948-7d9999087886.png)
- SU = Setup uncertainty due to jitter which is temporary variation of clock period. This is due to non-idealities of PLL/clock source.

### Lab Part 5 [Day 4] - Pre-Layout STA with OpenSTA:
STA can either be **single corner** which only uses the `LIB_TYPICAL` library which is the one used in pre-layout(pos-synthesis) STA or **multicorner** which uses `LIB_SLOWEST`(setup analysis, high temp low voltage),`LIB_FASTEST`(hold analysis, low temp high voltage), and `LIB_TYPICAL` libraries. 

1. Run STA engine using OpenROAD (which in turn calls OpenSTA): run OpenROAD first then source `/openlane/scripts/openroad/sta.tcl` which contains the OpenROAD commands for single corner STA. This file also contains the path to the [SDC file](https://teamvlsi.com/2020/05/sdc-synopsys-design-constraint-file-in.html) which specifies the actual timing constraints of the design. 
![189568030-f442a238-21e8-4fc1-b5d0-22de00b11af9](https://github.com/Shrachinag/pes_pd/assets/119600435/dca4044e-51c4-4b60-9b69-5cfc03c665f9)

The result of running STA in OpenROAD will be exactly the same as the log result of STA after running `run_synthesis` inside OpenLane. Observe the delay:
![image](https://user-images.githubusercontent.com/87559347/189686801-46a9fb96-9be6-40c7-b62a-da3160489cb0.png)

2. To reduce negative slack, focus on large delays. Notice how net `_02682_` has big fanout of 5. Use `report_net -connections _02682_` to display connections. First thing we can do is to go back to OpenLane and reduce fanouts by `set ::env(SYNTH_MAX_FANOUT) 4` then `run_synthesis` again. As shown below, wns is reduced from -1.35ns to -0.82ns.  
![image](https://user-images.githubusercontent.com/87559347/189788023-9f6d85a9-a769-4b54-b156-2fa7b8980178.png)

3. To further reduce the negative slack, we can also try upsizing the cell with high fanout so bigger driver will be used. High fanout results in high load cap which then results in high delay. But since we cannot change the load cap, we can just change the cell size to better drive that large cap load for less delay. As shown below, cell `_41882_` has a high cap load of 0.04nF and this causes a large delay due to `buf_1` not having enough drive strength to drive that high cap load. We can try upsizing the `buf_1` to `buf_4` (listed on the used liberty files are all cells which you can choose) inside OpenSTA: `replace_cell _41882_ sky130_fd_sc_hd__buf_4` 
![image](https://user-images.githubusercontent.com/87559347/189793281-6acff965-b4d1-48a8-a6c3-17d312f901a2.png)

This can be done iteratively until desired slack is reached, this is called timing ECO (Engineering Change Order). To extract the modified verilog netlist: `write_verilog designs/picorv32a/runs/RUN_2022.09.14_05.18.35/results/synthesis/picorv32.v`. Beware that upsizing the cell will naturally increase core size. 

### Summary of OpenSTA Commands:  
```
report_net -connections _02682_
replace_cell _41882_ sky130_fd_sc_hd__buf_4`
report_checks -fields {cap slew nets} -digits 4
report_checks -from _18671_ -to _18739_ -fields {cap slew nets} -digits 4
report_wns
report_tns
report_worst_slack -max
write_verilog designs/picorv32a/runs/RUN_2022.09.14_05.18.35/results/synthesis/picorv32.v
```

### SDC File Parameters:

- [create_clock](http://ebook.pldworld.com/_Semiconductors/Actel/Libero_v70_fusion_webhelp/create_clock_sdc_constraint.htm)
```
create_clock clk  -name sys_clk  -period 10
```
This creates a clock named `sys_clk` to the port `clk` with period of 10ns. However, it is recommended to add `get_ports` when referencing a port to not confuse the object type (is it a clk, net, or port?):
```
create_clock [get_ports clk]  -name sys_clk  -period 10
```

 - [set_input_delay](http://ebook.pldworld.com/_Semiconductors/Actel/Libero_v70_fusion_webhelp/set_input_delay_%28sdc_input_delay_constraint%29.htm)/[set_output_delay](https://www.intel.com/content/www/us/en/docs/programmable/683432/21-4/tcl_pkg_sdc_ver_1-5_cmd_set_output_delay.html) = Defines the arrival/exit time of an input/output signal relative to the input clock. This is the delay of the signal coming from an external block and internal delay of the signal to be propagated to external ports.
 
 ```
 set_input_delay 1 -clock [get_clock clk] [all_input]
 set_output_delay 0.5 -clock [get_clock clk] [all_output]
 ```
 This adds a delay of 1ns relative to `clk` to all signals going to input ports, and delay of 0.5ns relative to `clk` to all signals going to output ports.


 - [set_max_fanout](https://hdvacademy.blogspot.com/2014/07/design-constraints.html) = below constraint specifies a max fanout of 10 for all output ports in the design
 ```
 set_max_fanout 10 [current_design]
 ```
- [set_driving_cell](https://www.micro-ip.com/tw/STA/dictionary_516_17/set_driving_cell.html) = Models an external driver at the input port of the current design 

```
set_driving_cell -lib_cell sky130_fd_sc_hd__inv_2 -pin Y $all_inputs_wo_clk_rst
set_driving_cell -lib_cell sky130_fd_sc_hd__inv_8 -pin Y clk
```
The first constraint sets `sky130_fd_sc_hd__inv_2` (specifically pin `Y` of the cell) to drive all input ports except `clk`. The second consraint sets `sky130_fd_sc_hd__inv_8` (specifically pin 'Y' of this cell) for the driving the clk input port.


- set_load = Below constraint sets a 10nF capacitive load to all output ports. set_load can also be used on internal net.
```
set_load 10 [all_outputs]
```

- set_clock_uncertainty = Below constraint incorporates 0.25ns skew to `clk`
```
set_clock_uncertainty 0.25 [get_clock clk]
```

- [set_clock_transition](https://www.micro-ip.com/tw/Synopsys(DC)/dictionary_37_8/set_clock_transition.html) = Sets both rise and fall transition times to 0.15ns on clock pins of all sequential elements clocked by `clk`:
```
set_clock_transition 0.15 [get_clocks clk]

```
[Here](https://hdvacademy.blogspot.com/2014/07/design-constraints.html) and [here](https://www.micro-ip.com/tw/drchip.php?mode=2&cid=8) is a great reference for some common SDC constraints. As a side note, [as said here](https://electronics.stackexchange.com/questions/339401/get-ports-vs-get-pins-vs-get-nets-vs-get-registers) I/Os of the top-level block are called port while I/Os of the subblocks are called pin.

### Clock Tree Synthesis Stage:
There are three parameters that we need to consider when building a clock tree:
- Clock Skew = In order to have minimum skew between clock endpoints, clock tree is used. This results in equal wirelength (thus equal latency/delay) for every path of the clock. 
- Clock Slew = Due to wire resistance and capacitance of the clock nets, there will be slew in signal at the clock endpoint where signal is not the same with the original input clock signal anymore. This can be solved by clock buffers. Clock buffer differs in regular cell buffers since clock buffers has equal rise and fall time. 
- Crosstalk = Clock shielding prevents crosstalk to nearby nets by breaking the coupling capacitance between the victim (clock net) and aggresor (nets near the clock net), the shield might be connected to VDD or ground since those will not switch. Shileding can also be done on critical data nets.

![image](https://user-images.githubusercontent.com/87559347/190031283-3bc25c79-f622-4b58-a448-95982d32612d.png)

### CTS Command Script:
After extracting the modified verilog netlist after doing timing ECO, `run_floorplan` and `run_placement` and then `run_cts`. In CTS, the verilog netlist is modified to add the clock buffers and this new verilog netlist is saved under `/runs/[date]/results/cts/`.

 `run_cts` and the other OpenLane commands are actually just calling the tcl proc (procedure) inside `/OpenLane/scripts/tcl_commands/`. This tcl procedure will then call OpenROAD to run the actual tool. For example, `run_cts` can be found inside `/OpenLane/scripts/tcl_commands/cts.tcl`, this tcl procedure will call OpenROAD and will call `/OpenLane/scripts/openroad/cts.tcl` which contains the OpenROAD commands to run TritonCTS.

Inside the `/OpenLane/scripts/openroad/cts.tcl` contains the configuration variables for CTS. Notables ones are:
- `CTS_CLK_BUFFER_LIST` = list of clock branch buffers (`sky130_fd_sc_hd__clkbuf_8` `sky130_fd_sc_hd__clkbuf_4` `sky130_fd_sc_hd__clkbuf_2`)
- `CTS_ROOT_BUFFER` = clock buffer used for the root of the clock tree and is the biggest clock buffer to drive the clock tree of the whole chip (`sky130_fd_sc_hd__clkbuf_16`)
- `CTS_MAX_CAP` = maximum capacitance of the output port of the root clock buffer.


### Timing Analysis with Real Clocks:
Setup and hold analysis with real clock will now include clock buffer delays:
- In setup analysis, the point is that the data must arrive first before the clock rising edge to properly latch that data. Setup violation happens when path is slow. This is affected by parameters such as combinational delay, clock buffer delay, time period, setup time, and setup uncertainty (jitter).

- Hold analysis is the delay that the MUX2 model inside the flip flop needs to move the data to outside. This is the time that the launch flop must hold the data before it reaches the capture flop. Hold analysis is done on the same rising clock edge for launch and capture flop unlike in setup analysis where it spans between two rising clock edges. Hold violation happens when path is too fast. This is affected by parameters such as combinational delay, clock buffer delays, and hold time. (time period and setup uncertainty does not matter since launch and capture flops will receive the same rising clock edges fo hold analysis)

The goal is to have a positive slack on both setup and hold analysis.
![image](https://user-images.githubusercontent.com/87559347/190183335-fc20002a-b80b-4b86-ad0a-3db65a0b49c7.png)  

STA report for hold analysis (min path):
![image](https://user-images.githubusercontent.com/87559347/190203192-566f344e-b275-45de-af80-4058e1b34d31.png)

STA report for setup analysis (max path):
![image](https://user-images.githubusercontent.com/87559347/190202789-c79cd727-ebe3-4bc5-8fdc-a0f4dce77dba.png)

### Lab Part 6 [Day 4] - Multi-corner STA for Post-CTS:
We will now do STA for post clock tree synthesis to include effect of clock buffers. Similar to pre-layout STA, this will done on OpenROAD (which will then call OpenSTA):  
![image](https://user-images.githubusercontent.com/87559347/190295139-9ba76ec8-e116-467a-8960-77e941bf92ad.png)

- `write_db` and `read_db`is done before running STA tool, this creates a database file using LEF file and resulting DEF file of the last stage.
- Multi-corner STA must read both min library (for hold analysis) and max library (for setup analysis) unlike in single corner STA where only the typical library is read. 
- SDC file used is the same for single and multi-corner. 
- Since this is post-CTS STA, `set_propagated_clock` is used. `set_propagated_clock` propagates clock latency throughout a clock network, resulting in more accurate skew and timing results throughout the clock network. This is done  postlayout, after final clock tree generation, unlike in prelayout where ideal clock is used thus no clock latency.

Also instead of manually running these commands, we can just simply do `source /openlane/scripts/openroad/sta_multi_corner.tcl` inside OpenROAD which runs the readily-made tcl script of OpenROAD commmands for running multi-corner STA. The result might be slightly different from the result above since the settings for `sta_multi_corner.tcl` is much more comprehensive.



![image](https://user-images.githubusercontent.com/87559347/190305051-d703b77c-f634-4b2f-98ce-bb516d975faf.png)

We are now failing in both hold and setup analysis. Setup analysis can be solved by reducing clock frequency but hold analysis is independent of clock period so it is harder to solve. But this hold negative slack can be reduced when we run routing. In hold violation, the data path is too fast so the increase in combinational delay due to the actual resistance and capacitance of the routed wires can reduce the negative slack for hold analysis.

However, this large negative slack is due to TritonCTS only doing clock tree synthesis for typical corner and does not included max and min corners. Thus doing multi-corner STA is wrong on this case. What we can do is to go back to single corner STA simply by skipping reading min and max libraries and only the typical library.

### Lab Part 7 [Day 4] - Replacing the Clock Buffer:
When TritonCTS is building the branch clock tree, it tries each buffers listed in `$::env(CTS_CLK_BUFFER_LIST)` (`sky130_fd_sc_hd__clkbuf_8` `sky130_fd_sc_hd__clkbuf_4` `sky130_fd_sc_hd__clkbuf_2`) from smallest to largest until the target skew is met. Target skew is stored in `$::env(CTS_TARGET_SKEW)` as 200ps. The STA result shows that `sky130_fd_sc_hd__clkbuf_8` is the mostly used buffer, we will now change the `$::env(CTS_CLK_BUFFER_LIST)` to use smaller buffers and observe the effect on STA and area:

1. Use tcl `lreplace` command to modify `$::env(CTS_CLK_BUFFER_LIST)` so that only `sky130_fd_sc_hd__clkbuf_2` will remain:
![190331513-b0de28f8-6134-4805-8544-bf99f824226f](https://github.com/Shrachinag/pes_pd/assets/119600435/7f4f44e4-a716-4ba7-83ad-cc4b0d2a6e2a)


2. If you do `run_cts` now, the result will be wrong (the removed clock buffers will still appear) since we already run CTS before. The reason being is that the `$::env(CURRENT_DEF)` used by CTS is the DEF file result of the previously run CTS too. What DEF file we want for CTS is the placement's DEF file. So just change the `$::env(CURRENT_DEF)` to point to placement DEF file then `run_cts`:

![image](https://user-images.githubusercontent.com/87559347/190335410-0f95a7bd-b6f9-4c4e-a90f-1698a05d1596.png)

3. Observe the resulting post-CTS STA compared to before we modify the clock buffer. Only `buf_2` clock buffer is used now compared to `buf_8` used in previous run. The WNS is worse now since we used smaller clock buffers thus larger clock path delay, however the area is now smaller since we used smaller clock buffer.

![image](https://user-images.githubusercontent.com/87559347/190339718-0e759d3e-b81e-4cb1-94f1-b075404b4460.png)


 
</details>
