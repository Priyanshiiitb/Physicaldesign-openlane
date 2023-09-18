# Physicaldesign-openlane
## Day1 - Introduction
## SoC Design & OpenLANE
### Components of opensource digital ASIC design
-The design of digital Application Specific Integrated Circuit (ASIC) requires three enablers or elements - Resistor Transistor Logic Intellectual Property (RTL IPs), Electronic Design Automation (EDA) Tools and Process Design Kit (PDK) data.

* Opensource RTL Designs: github, librecores, opencores 
* Opensource EDA tools: QFlow, OpenROAD, OpenLANE 
* Opensource PDK data: Google Skywater130 PDK

-the ASIC flow objective is to convert RTL design to GDSII format used for final layout. The flow is essentially a software also known as automated PnR (Place & route).

### Simplified RTL2GDS Flow
![Screenshot from 2023-09-10 15-48-20](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/673d1f2c-e3a1-413b-b995-8069d2c89071)

## Openlane flow
### 1.  Synthesis 
The RTL Level Design is then synthesized using a Logic Synthesizer. We use Yosys which is an Open Source Logic Synthesizer. The RTL Netlist is then  converted into a synthesised netlist where there are details about the standard cells and its implementations. Yosys takes the RTL design and timing .libs and verilog models of standard cells and converts  into  a  RTL Netlist. abc does the tehnology mapping to the required skywater-pdk variants 

### 1.1 Synthesis Strategies
Different strategies can be used to synthesize for the either the least area or the best timing. To analyse this, synthesis exploration utility generates a report showing the effect on delays/timing/area et.,

### 1.2 Deign Exploration Utility 
This is used to suit the design configuration and generate reports with different metrics to select the best. This is also used for regression testing

### 1.3 Design For Test - DFT Insertion
This is an optional step carried out by Fault. It is used to test the design 

###  2. Floor Planning and Power Planning
This is done by OpenROAD flow. The macros and IPs are placed in the core before proceding further. This is called as pre-placement. Floor planning is done separately for the macros and it is called macro floor planning. They are placed in such a way that they are closer to the inputs/outputs/other macros where more connections are present. Then to prevent the loading effects de-coupling capacitors are placed so that the logic states are well within the noise margin. 

When several blocks tap power from a single source, there is a problem of Voltage Droop at the Vdd and Ground Bounce at the Vss which can again push the logic out of the required noise margin into the undefined state. To mitigate this Vdd and Vss are placed as horizontal and vertical strips in the chip so that the blocks can tap power from the nearest source. 

### 3. Placement
There are two types of placement.  The other required logic is placed optimally.
Placement is of two steps
- Global Placement- finds the optimal position for each cells. These positions are not necessarly correct, cells may overlap
- Detialed Placement - After Global placement is done minimal alterations are done to correct the issues

### 4. Clock Tree Synthesis 
To ensure minimum skew the Clock is routed optimally through the circuit using different algorithms. This is done in the OpenROAD flow. This is done by TritonCTS.

### 5. Fake Antenna and diode swapping
Long wires acts as antennas and cause accumulation of charges during the fabrication process damaging the transistor. To avoid this bridging is used to pass the wire through different layers or an antenna diode cell is added to leak away the charges
- OpenLane approach - Insert Fake Diode to every cell input during placement. This matches the footprint of the library of the antenna diode. The Antenna Checker is run to check for violations, if there are violations then the fake diode is swapped with a real one.
- OpenROAD approach - In the global route step, the antenna violation is addressed automatically by inserting an antenan diode
OpenLane allows the user to chose either of the above approaches

###  5. Routing
This step is used to implement the interconnect using the different metal layers specified in the PDK. There are two steps

 - Global Routing - This is done inside the OpenROAD flow (FastRoute)
 - Detailed Routing - This is performed using TritonRoute outside the OpenROAD flow after the global routing. Before performing this step the **Logic Equivalence Check** is performed by Yosys, since OpenROAD does some optimisations the circuit.  

### 6. RC Extraction
From the .def file, the parasitic extraction is done to generate the .spef file (Standard Prasitic Exchange Format) which produces an accurate analog model of the circuit by including the parasitic effects due to wires, parasitic capacitances, etc.,

### 7. STA
At this stage again OpenSTA is used to perform the Static Timing Analysis.  

### 8. Sign-off Steps
- Design Rule Check (DRC) is performed by Magic
- Layout Versus Schematic (LVS) is performed by Netgen

### 9. GDSII Extraction
The routed .def file is used my Magic to generate the GDSII file 

## Invoking OpenLANE and Design Preparation
Openlane can be invoked using docker command followed by opening an interactive session. flow.tcl is a script that specifies details for openLANE flow.

```
docker
./flow.tcl -interactive
package require openlane 0.9
prep -design picorv32a
```
![Screenshot from 2023-09-10 15-58-09](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/dec49ad1-9524-4315-9acb-7f5eef3726af)

## Synthesis

Run the synthesis

```run_synthesis```

OpenLane invokes the following

- `Yosys` - RTL Synthesis and maps to yosys generic cells
- `abc` - Technology mapping with the Skywater130 PDK. Here `sky130_fd_sc_hd` Skywater Foundry produced High density standard cells are used.
- `OpenSTA` - This does the Static Timing Analysis on the netlist generated after synthesis and generated the timing reports 

View the synthesis statistics
![Screenshot from 2023-09-10 10-07-14](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/2222c930-4ffb-4ca4-ab6f-2000c3dea088)




## Day 2 Good Floorplan Vs Bad Floorplan and Introduction to Library Cells
### Floorplanning considerations
### Utilization Factor & Aspect Ratio
Two parameters are of importance when it comes to floorplanning namely, Utilisation Factor and Aspect Ratio. They are defined as follows:
```
Utilisation Factor =  Area occupied by netlist
                     __________________________
                        Total area of core
```
```
Aspect Ratio =  Height
               ________
                Width
```
A Utilisation Factor of 1 signifies 100% utilisation leaving no space for extra cells such as buffer. However, practically, the Utilisation Factor is 0.5-0.6. Likewise, an Aspect ratio of 1 implies that the chip is square shaped. Any value other than 1 implies rectanglular chip.

#### Pre-placed cells

Once the Utilisation Factor and Aspect Ratio has been decided, the locations of pre-placed cells need to be defined. Pre-placed cells are IPs comprising large combinational logic which once placed maintain a fixed position. Since they are placed before placement and routing, the are known as pre-placed cells.

#### Decoupling capacitors

Pre-placed cells must then be surrounded with decoupling capacitors (decaps). The resistances and capacitances associated with long wire lengths can cause the power supply voltage to drop significantly before reaching the logic circuits. This can lead to the signal value entering into the undefined region, outside the noise margin range. Decaps are huge capacitors charged to power supply voltage and placed close the logic circuit. Their role is to decouple the circuit from power supply by supplying the necessary amount of current to the circuit. They pervent crosstalk and enable local communication.

#### Power Planning

Each block on the chip, however, cannot have its own decap unlike the pre-placed macros. Therefore, a good power planning ensures that each block has its own VDD and VSS pads connected to the horizontal and vertical power and GND lines which form a power mesh.

#### Pin Placement

The netlist defines connectivity between logic gates. The place between the core and die is utilised for placing pins. The connectivity information coded in either VHDL or Verilog is used to determine the position of I/O pads of various pins. Then, logical placement blocking of pre-placed macros is performed so as to differentiate that area from that of the pin area.
#### Floorplan run on OpenLANE & view in Magic

* Importance files in increasing priority order:

1. ```floorplan.tcl``` - System default envrionment variables
2. ```conifg.tcl```
3. ```sky130A_sky130_fd_sc_hd_config.tcl```

* Floorplan envrionment variables or switches:

1. ```FP_CORE_UTIL``` - floorplan core utilisation
2. ```FP_ASPECT_RATIO``` - floorplan aspect ratio
3. ```FP_CORE_MARGIN``` - Core to die margin area
4. ```FP_IO_MODE``` - defines pin configurations (1 = equidistant/0 = not equidistant)
5. ```FP_CORE_VMETAL``` - vertical metal layer
6. ```FP_CORE_HMETAL``` - horizontal metal layer

***Note: Usually, vertical metal layer and horizontal metal layer values will be 1 more than that specified in the files***
 
 To run the picorv32a floorplan in openLANE:
 ```
 run_floorplan
 
 ```
![Screenshot from 2023-09-11 11-19-41](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/d852e711-a445-40ab-8603-fbc266b8e3f0)

Post the floorplan run, a .def file will have been created within the ```results/floorplan``` directory. 
We may review floorplan files by checking the ```floorplan.tcl```. 
The system defaults will have been overriden by switches set in ```conifg.tcl``` and further overriden by switches set in ```sky130A_sky130_fd_sc_hd_config.tcl```.
 
To view the floorplan, Magic is invoked after moving to the ```results/floorplan``` directory:

```
magic -T /Home/OpenLane/pdks/sky130A/libs.tech/magic/sky130A.tech lef read ../../tmp/merged.min.lef def read picorv32.def &
```
![Screenshot from 2023-09-11 11-23-46](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/1eb7a3a3-bbf6-4308-b649-1c2ac1d356f4)

One can zoom into Magic layout by selecting an area with left and right mouse click followed by pressing "z" key.

Various components can be identified by using the what command in tkcon window after making a selection on the component.

Zooming in also provides a view of decaps present in picorv32a chip.

The standard cell can be found at the bottom left corner.

You can clearly see I/O pins, Decap cells and Tap cells. Tap cells are placed in a zig zag manner or you can say diagonally

### Placement
To perform placement
```
run_placement
```
![Screenshot from 2023-09-11 10-56-51](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/e6837458-50f9-4d49-a720-abbbfa8aa1a5)
### Standard Cell Design Flow

Standard cell design flow involves the following:

1. Inputs: PDKs, DRC & LVS rules, SPICE models, libraries, user-defined specifications. 
2. Design steps: Circuit design, Layout design (Art of layout Euler's path and stick diagram), Extraction of parasitics, Characterization (timing, noise, power).
3. Outputs: CDL (circuit description language), LEF, GDSII, extracted SPICE netlist (.cir), timing, noise and power .lib files

### Standard Cell Characterization Flow

A typical standard cell characterization flow includes the following steps:

1. Read in the models and tech files
2. Read extracted spice netlist
3. Recognise behaviour of the cell
4. Read the subcircuits
5. Attach power sources
6. Apply stimulus to characterization setup
7. Provide necessary output capacitance loads
8. Provide necessary simulation commands

The opensource software called GUNA can be used for characterization. Steps 1-8 are fed into the GUNA software which generates timing, noise and power models.

### Timing Parameter Definitions

Timing defintion | Value
------------ | -------------
slew_low_rise_thr  | 20% value
slew_high_rise_thr |  80% value
slew_low_fall_thr | 20% value
slew_high_fall_thr | 80% value
in_rise_thr | 50% value
in_fall_thr | 50% value
out_rise_thr | 50% value
out_fall_thr | 50% value

```
rise delay =  time(out_fall_thr) - time(in_rise_thr)

Fall transition time: time(slew_high_fall_thr) - time(slew_low_fall_thr)

Rise transition time: time(slew_high_rise_thr) - time(slew_low_rise_thr)
```

A poor choice of threshold points leads to neative delay value. Therefore a correct choice of thresholds is very important.

## Day 3 Design Library Cell using ngspice simulations and Magic layout
### SPICE Deck creation & Simulation
A SPICE deck includes information about the following:

1 Model description
2 Netlist description
3 Component connectivity
4 Component values
5 Capacitance load
6 Nodes
7 Simulation type and parameters
8 Libraries included
### CMOS inverter Switching Threshold Vm
The sitching threshold of a CMOS inverter is the point on the transfer characteristic where Vin equals Vout (=Vm). At this point both PMOS and NMOS are in ON state which gives rise to a leakage current

### 16 Mask CMOS Fabrication
The 16-mask CMOS process consists of the following steps:

1. Selection of subtrate: Secting the body/substrate material.
2. Creating active region for transistors: Isolation between active region pockets by SiO2 and Si3N4 deposition followed by photolithography and etching.
3. N-well and P-well formation: Ion implanation by Boron for P-well and by Phosphorous for N-well formation.
4. Formation of gate terminal: NMOS and PMOS gates formed by photolithography techniques.
5. LDD (lightly doped drain) formation: LDD formed to prevent hot electron effect.
6. Source & drain formation: Screen oxide added to avoid channelling during implants followed by Aresenic implantation and annealing.
7. Local interconnect formation: Removal of screen oxide by HF etching. Deposition of Ti for low resistant contacts.
8. Higher level metal formation: CMP for planarization followed by TiN and Tungsten deposition. Top SiN layer for chip protection.

## SPICE Deck Creation and Simulation for CMOS inverter

- Before performing a SPICE simulation we need to create SPICE Deck
SPICE Deck provides information about the following:
- Component connectivity - Connectivity of the Vdd, Vss,Vin, substrate. Substrate tunes the threshold voltage of the MOS.
- component values - values of PMOS and NMOS, Output load, Input Gate Voltage, supply voltage.
- Node Identification and naming - Nodes are required to define the SPICE Netlist
     For example ```M1 out in vdd vdd pmos w = 0.375u L = 0.25u``` , ```cload out 0 10f```
- Simulation commands
- Model file - information of parameters related to transistors
Simulation of CMOS using different width and lengths. From the waveform, irrespective of switching the shape of it are almost same.

![Screenshot from 2023-09-12 22-18-49](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/4b9a5f2d-c07b-494d-842f-24706fa9c56f)

## Lab steps to git clone vsdstdcelldesign

- First, clone the required mag files and spicemodels of inverter,pmos and nmos sky130. The command to clone files from github link is:
```
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
```
once I run this command, it will create ``vsdstdcelldesign`` folder in openlane directory.

Inorder to open the mag file and run magic go to the directory

For layout we run magic command

``` magic -T sky130A.tech sky130_inv.mag & ```

Ampersand at the end makes the next prompt line free, otherwise magic keeps the prompt line busy. Once we run the magic command we get the layout of the inverter in the magic window
![Screenshot from 2023-09-12 21-10-41](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/6c7aef09-21a1-413b-b1d6-87ec4ad9f02e)

## SKY130 basic layer layout and LEF using inverter

- From Layout, we see the layers which are required for CMOS inverter. Inverter is, PMOS and NMOS connected together.
- Gates of both PMOS and NMOS are connected together and fed to input(here ,A), NMOS source connected to ground(here, VGND), PMOS source is connected to VDD(here, VPWR), Drains of PMOS and NMOS are connected together and fed to output(here, Y). 
The First layer in skywater130 is ``localinterconnect layer(locali)`` , above that metal 1 is purple color and metal 2 is pink color.
If you want to see connections between two different parts, place the cursor over that area and press S one times. The tkson window gives the component name.
![Screenshot from 2023-09-12 21-13-41](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/a6f9bf65-2a2b-4246-833d-d42c2e57c82f)
### Designing standard cell and SPICE extraction in MAGIC 

-  First we need to provide bounding box width and height in tkson window. lets say that width of BBOX is 1.38u and height is 2.72u. The command to give these values to magic is
   ``` property Fixed BBOX (0 0 1.32 2.72)  ```
- After this, Vdd, GND segments which are in metal 1 layer, their respective contacts and atlast logic gates layout is defined
Inorder to know the logical functioning of the inverter, we extract the spice and then we do simulation on the spice. To extract it on spice we open TKCON window, the steps are
- Know the present directory - ``pwd ``
- create an extration file -  the command is  `` extract all `` and  ``sky130_inv.ext`` files has been created
          
- create spice file using .ext file to be used with our ngspice tool  - the commands are  
      ``` ext2spice cthresh 0 rthresh 0 ``` - extracts parasatic capcitances also since these are actual layers - nothing is created in the folder
      ``` ext2spice ``` - a file ```sky130_inv.spice``` has been created.
  ![Screenshot from 2023-09-12 21-21-01](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/cdd4f8b0-c339-4dd5-8ca6-46b92dc969d5)
  

  #### the final sky130_inv.spice file is modified to:
  
  ![Screenshot from 2023-09-12 22-34-42](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/c1c919fe-3da2-4ad8-95a1-8afcd447dd01)

  For simulation, ngspice is invoked in the terminal:

```
ngspice sky130_inv.spice
```

The output "y" is to be plotted with "time" and swept over the input "a":

```
plot y vs time a
```
![Screenshot from 2023-09-12 22-40-38](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/00351bdb-80e6-49e2-958f-7a0e95d38ae0)

## LAB exercise and DRC Challenges

## Intrdocution of Magic and Skywater DRC's

  - In-depth overview of Magic's DRC engine
  - Introduction to Google/Skywater DRC rules
  - Lab : Warm-up exercise : Fixing a simple rule error
  - Lab : Main exercie : Fixing or create a complex error

 # Sky130s pdk intro and Steps to download labs
  
  - setup to view the layouts
  - For extracting and generating views, Google/skywater repo files were built with Magic
  - Technology file dependency is more for any layout. hence, this file is created first.
  - Since, Pdk is still under development, there are some unfinished tech files and these are packaged for magic along with lab exercise layout and bunch of stuff into the tar ball
 
We can download the packaged files from web using ``wget `` command. wget stands for web get, a non-interactive file downloader command.
  
  ``` wget http://opencircuitdesign.com/open_pdks/archive/drc_tests.tgz```
  
The archive file drc_tests.tgz is downloaded into our user directory 
  
Now run MAGIC

For better graphics use command ``magic -d XR ``

Now, lets see an example of simple failing set of rules of metal 1 layer.  you can either run this by magic command line `` magic -d XR met1.mag `` or from the magic console window, `` menu - file - open -load file9here, met1.mag) ``

![Screenshot from 2023-09-16 10-13-24](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/4502c266-62e5-408e-9dae-0cbb74709af6)

## Load Sky130 tech rules for drc challenges 

First load the poly file by ``load poly.mag`` on tkcon window.

Finding the error by mouse cursor and find the box area, Poly.9 is violated due to spacing between polyres and poly.

![Screenshot from 2023-09-16 10-36-13](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/6b5dfe99-d4a5-41ef-9321-47fd47ed94a7)


   
In line

```
spacing npres *nsd 480 touching_illegal \
	"poly.resistor spacing to N-tap < %d (poly.9)"
```
change to

```
spacing npres allpolynonres 480 touching_illegal \
	"poly.resistor spacing to N-tap < %d (poly.9)"
```
Also,
```
spacing xhrpoly,uhrpoly,xpc alldiff 480 touching_illegal \

	"xhrpoly/uhrpoly resistor spacing to diffusion < %d (poly.9)"
```

change to 

```
spacing xhrpoly,uhrpoly,xpc allpolynonres 480 touching_illegal \

	"xhrpoly/uhrpoly resistor spacing to diffusion < %d (poly.9)"

```
![Screenshot from 2023-09-16 10-36-33](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/1fb876f5-f987-4dc2-abaa-170cd0589359)


## Day 4 Timing Analysis and Clock Tree Synthesis (CTS)
A requirement for ports as specified in ```tracks.info``` is that they should be at intersection of horizontal and vertical tracks. The CMOS Inverter ports A and Y are on li1 layer. It needs to be ensured that they're on the intersection of horizontal and vertical tracks. 
To ensure that ports lie on the intersection point, the grid spacing in Magic (tkcon) must be changed to the li1 X and li1 Y values. Convergence of grid and tracks can be achieved using the following command:

```
grid 0.46um 0.34um 0.23um 0.17um
```
![Screenshot from 2023-09-16 16-12-29](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/d36aba61-8cc1-4f06-83c4-9f99bdd19899)
## Create Port Definition: 

However, certain properties and definitions need to be set to the pins of the cell. For LEF files, a cell that contains ports is written as a macro cell, and the ports are the declared as PINs of the macro.

The way to define a port is through Magic console and following are the steps:
- In Magic Layout window, first source the .mag file for the design (here inverter). Then Edit >> Text which opens up a dialogue box.
- When you double press S at the I/O lables, the text automatically takes the string name and size. Ensure the Port enable checkbox is checked and default checkbox is unchecked as shown in the figure:
![Screenshot from 2023-09-16 16-20-06](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/4a2f70a3-5ed2-48c8-b140-58bca6d00026)
![Screenshot from 2023-09-16 16-56-46](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/97040496-92f5-4e61-b8f1-c132b85d0a49)
- In the above figure, The number in the textarea near enable checkbox defines the order in which the ports will be written in LEF file (0 being the first).

-  For power and ground layers, the definition could be same or different than the signal layer. Here, ground and power connectivity are taken from metal1

## Set port class and port use attributes for layout 

After defining ports, the next step is setting port class and port use attributes.

Select port A in magic:
```
port class input
port use signal
```
Select Y area
```
port class output
port use signal
```
Select VPWR area
```
port class inout
port use power
```
Select VGND area
```
port class inout
port use ground

```
## Custom cell naming and lef extraction.

Name the custom cell through tkcon window as ```sky130_vsdinv.mag```.

We generate lef file by command:

```
lef write

```
This generates sky130_vsdinv.lef file.


![Screenshot from 2023-09-16 19-22-30](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/adf3ce1e-e320-4da2-ba6d-f9d9481ff9ec)

## Steps to include custom cell in ASIC design

We have created a custom standard cell in previous steps of an inverter. Copy lef file, sky130_fd_sc_hd_typical.lib, sky130_fd_sc_hd_slow.lib & sky130_fd_sc_hd_fast.lib to src folder of picorv32a from libs folder vsdstdcelldesign. Then modify the config.tcl as follows.

```

# Design
set ::env(DESIGN_NAME) "picorv32a"

set ::env(VERILOG_FILES) "$::env(DESIGN_DIR)/src/picorv32a.v"

set ::env(CLOCK_PORT) "clk"
set ::env(CLOCK_NET) $::env(CLOCK_PORT)

set ::env(GLB_RESIZER_TIMING_OPTIMIZATIONS) {1}

set ::env(LIB_SYNTH) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_SLOWEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_FASTEST) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_TYPICAL) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"

set ::env(EXTRA_LEFS) [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]

set filename $::env(DESIGN_DIR)/$::env(PDK)_$::env(STD_CELL_LIBRARY)_config.tcl
if { [file exists $filename] == 1} {
	source $filename
}

```

To integrate standard cell in openlane flow after `` make mount `` , perform following commands:

```
prep -design picorv32a -tag RUN_2023.09.09_20.37.18 -overwrite 
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
run_synthesis

```
Synthesis log-

![Screenshot from 2023-09-18 13-48-49](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/420d29cf-a7dd-4103-b4dd-11bc448b9f53)


STA report 

![Screenshot from 2023-09-18 13-50-48](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/6afe8ae1-d07d-42f7-b13b-8dd678eddba9)

## Delay Tables

Basically, Delay is a parameter that has huge impact on our cells in the design. Delay decides each and every other factor in timing. 
For a cell with different size, threshold voltages, delay model table is created where we can it as timing table.
```Delay of a cell depends on input transition and out load```. 
Lets say two scenarios, 
we have long wire and the cell(X1) is sitting at the end of the wire : the delay of this cell will be different because of the bad transition that caused due to the resistance and capcitances on the long wire.
we have the same cell sitting at the end of the short wire: the delay of this will be different since the tarn is not that bad comapred to the earlier scenario.
Eventhough both are same cells, depending upon the input tran, the delay got chaned. Same goes with o/p load also.

VLSI engineers have identified specific constraints when inserting buffers to preserve signal integrity. They've noticed that each buffer level must maintain consistent sizing, but their delays can vary depending on the load they drive. To address this, they introduced the concept of "delay tables," which essentially consist of 2D arrays containing values for input slew and load capacitance, each associated with different buffer sizes. These tables serve as timing models for the design.

When the algorithm works with these delay tables, it utilizes the provided input slew and load capacitance values to compute the corresponding delay values for the buffers. In cases where the precise delay data is not readily available, the algorithm employs a technique of interpolation to determine the closest available data points and extrapolates from them to estimate the required delay values.

![Screenshot from 2023-09-17 16-04-27](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/f2629ab9-a437-4f53-afcc-d19b0910a3ee)

## Openlane steps with custom standard cell

Custom Cell inclusion in OpenLane Flow

We have seen till the synthesis for the custom standard cell in OpenLane flow, and verified the synthesis and STA log files. We will pick it from there now.

First check the slack for the synthesis.

The slack was positive, therefore we can proceed, else would have to work on the slack.

Now we run the floorplan and placement processes.
We perform synthesis and found that it has positive slack and met timing constraints.

During Floorplan,``` 504 endcaps, 6731 tapcells ``` got placed. Design has 275 original rows

Now ``` run_placement```

![Screenshot from 2023-09-18 13-53-29](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/7d0ddb96-3e9c-48a1-a8fe-42893a40c231)

After placement, we check for legality &To check the layout invoke magic from the results/placement directory:

```
magic -T /home/priyansh/OpenLane/vsdstdcelldesign/libs/sky130A.tech lef read ../../tmp/merged.nom.lef def read picorv32.def &


```
![Screenshot from 2023-09-18 15-44-40](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/8043c2f3-d1eb-429a-b032-1c17fde409d1)


### Clock Tree Synthesis using Tritoncts
Clock tree synthesis (CTS) can be implemented in various ways, and the choice of the specific technique depends on the design requirements, constraints, and goals. Here are some different types or approaches to clock tree synthesis:

Balanced Tree CTS: In a balanced tree CTS, the clock signal is distributed in a balanced manner, often resembling a binary tree structure. This approach aims to provide roughly equal path lengths to all clock sinks (flip-flops) to minimize clock skew. It's relatively straightforward to implement and analyze but may not be the most power-efficient solution.

H-tree CTS: An H-tree CTS uses a hierarchical tree structure, resembling the letter "H." It is particularly effective for distributing clock signals across large chip areas. The hierarchical structure can help reduce clock skew and optimize power consumption.

Star CTS: In a star CTS, the clock signal is distributed from a single central point (like a star) to all the flip-flops. This approach simplifies clock distribution and minimizes clock skew but may require a higher number of buffers near the source.

Global-Local CTS: Global-Local CTS is a hybrid approach that combines elements of both star and tree topologies. The global clock tree distributes the clock signal to major clock domains, while local trees within each domain further distribute the clock. This approach balances between global and local optimization, addressing both chip-wide and domain-specific clocking requirements.

Mesh CTS: In a mesh CTS, clock wires are arranged in a mesh-like grid pattern, and each flip-flop is connected to the nearest available clock wire. It is often used in highly regular and structured designs, such as memory arrays. Mesh CTS can offer a balance between simplicity and skew minimization.

Adaptive CTS: Adaptive CTS techniques adjust the clock tree structure dynamically based on the timing and congestion constraints of the design. This approach allows for greater flexibility and adaptability in meeting design goals but may be more complex to implement.


### crosstalk in VLSI:
Impact: Crosstalk is a significant concern in VLSI design due to the high integration density of components on a chip. Uncontrolled crosstalk can lead to data corruption, timing violations, and increased power consumption. Mitigation: VLSI designers employ various techniques to mitigate crosstalk, such as optimizing layout and routing, using appropriate shielding, implementing proper clock distribution strategies, and utilizing clock gating to reduce dynamic power consumption when logic is idle

### Clock Net Shielding in VLSI:
Purpose: In VLSI circuits, the clock distribution network is crucial for synchronous operation. Clock signals must reach all parts of the chip while minimizing skew and maintaining signal integrity. Shielding Techniques: VLSI designers may use shielding techniques to isolate the clock network from other signals, reducing the risk of interference. This can include dedicated clock routing layers, clock tree synthesis algorithms, and buffer insertion to manage clock distribution more effectively. Clock Domain Isolation: VLSI designs often have multiple clock domains. Shielding and proper clock gating help ensure that clock signals do not propagate between domains, avoiding metastability issues and maintaining synchronization.

### CTS LAB
The below command is used to run CTS in OpenLANE
```
run_cts 
```

![Screenshot from 2023-09-18 22-31-04](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/a7a29c6c-7c00-4263-8e47-d7bcbfffe00d)

After CTS run, my slack values are setup:12.76, Hold:0.19
Here also both values are not violating.

## Day 5 Final steps in RTL2GDS
### Maze Routing and Lee's algorithm

 Routing is the process of establishing a physical connection between two pins. Algorithms designed for routing take source and target pins and aim to find the most efficient path between them, ensuring a valid connection exists.

The Maze Routing algorithm, such as the Lee algorithm, is one approach for solving routing problems. In this method, a grid similar to the one created during cell customization is utilized for routing purposes. The Lee algorithm starts with two designated points, the source and target, and leverages the routing grid to identify the shortest or optimal route between them.

The algorithm assigns labels to neighboring grid cells around the source, incrementing them from 1 until it reaches the target (for instance, from 1 to 7). Various paths may emerge during this process, including L-shaped and zigzag-shaped routes. The Lee algorithm prioritizes selecting the best path, typically favoring L-shaped routes over zigzags. If no L-shaped paths are available, it may resort to zigzag routes. This approach is particularly valuable for global routing tasks.

However, the Lee algorithm has limitations. It essentially constructs a maze and then numbers its cells from the source to the target. While effective for routing between two pins, it can be time-consuming when dealing with millions of pins. There are alternative algorithms that address similar routing challenges.

![Screenshot from 2023-09-17 18-25-05](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/1d3e79a5-897d-4608-98e1-1db8bca60b0e)

# Design Rule Check (DRC)

DRC verifies whether a design meets the predefined process technology rules given by the foundry for its manufacturing. DRC checking is an essential part of the physical design flow and ensures the design meets manufacturing requirements and will not result in a chip failure. It defines the Quality of chip. They are so many DRCs, let us see few of them

Design rules for physical wires

Minimum width of the wire
Minimum spacing between the wires
Minimum pitch of the wire To solve signal short violation, we take the metal layer and put it on to upper metal layer. we check via rules
Via width
via spacing

![Screenshot from 2023-09-17 18-29-11](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/9d1b7d1d-1c76-45dd-9f6f-d0ab9b843108)

### Power Distribution Network and Routing 

- Unlike the general ASIC flow, Power Distribution Network generation is not a part of floorplan run in OpenLANE. PDN must be generated after CTS and post-CTS STA analyses:
- We can check whether PDN has been created or no by check the current def environment variable:  ``` echo $::env(CURRENT_DEF) ```

```bash
gen_pdn
```
- gen_pdn Generates the power distribution network.

- The power distribution network has to take the design_cts.def as the input def file.

- Power rings,strapes and rails are created by PDN.

- From VDD and VSS pads, power is drawn to power rings.

- Next, the horizontal and vertical strapes connected to rings draw the power from strapes.

- Stapes are connected to rings and these rings are connected to std cells. So, standard cells get power from rails.

- Here are definitions for the straps and the rails. In this design, straps are at metal layer 4 and 5 and the standard cell rails are at the metal layer 1. Vias connect accross the layers as required.


![Screenshot from 2023-09-18 22-13-42](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/878fd194-85ac-46e6-9c03-d1ede0794154)
  
### Routing 

In the realm of routing within Electronic Design Automation (EDA) tools, such as both OpenLANE and commercial EDA tools, the routing process is exceptionally intricate due to the vast design space. To simplify this complexity, the routing procedure is typically divided into two distinct stages: Global Routing and Detailed Routing.

- The two routing engines responsible for handling these two stages are as follows:
  - *Global Routing:*

    In this stage, the routing region is subdivided into rectangular grid cells and represented as a coarse 3D routing graph. This task is accomplished by the "FASTE ROUTE" engine.
  - *Detailed Routing:*

     Here, finer grid granularity and routing guides are employed to implement the physical wiring. The "tritonRoute" engine comes into play at this stage. "Fast Route" generates initial routing guides, while "Triton Route" utilizes the Global Route information and further refines the routing, employing various strategies and optimizations to determine the most optimal path for connecting the pins.


***Key Features of TritonRoute***
- *Initial Detail Routing*:

  TritonRoute initiates the detailed routing process, providing the foundation for the subsequent routing steps.

- *Adherence to Pre-Processed Route Guides*:

  TritonRoute places significant emphasis on following pre-processed route guides. This involves several actions:

- *Initial Route Guide Analysis*:

  TritonRoute analyzes the directions specified in the preferred route guides. If any non-directional routing guides are identified, it breaks them down into unit widths.

- *Guide Splitting:*

  In cases where non-directional routing guides are encountered, TritonRoute divides them into unit widths to facilitate routing.

- *Guide Merging:*

  TritonRoute merges guides that are orthogonal (touching guides) to the preferred guides, streamlining the routing process.

- *Guide Bridging:*

  When it encounters guides that run parallel to the preferred routing guides, TritonRoute employs an additional layer to bridge them, ensuring efficient routing within the preprocessed guides.

Assumes route guide for each net satisfy inter guide connectivity Same metal layer with touching guides or neighbouring metal layers with nonzero vertically overlapped area( via are placed ).each unconnected termial i.e., pin of a standard cell instance should have its pin shape overlapped by a routing guide( a black dot(pin) with purple box(metal1 layer))

### TritonRoute problem statement
```bash
Inputs : LEF, DEF, Preprocessed route guides
Output : Detailed routing solution with optimized wire length and via count
Constraints : Route guide honoring, connectivity constraints and design rules.
```

The space where the detailed route takes place has been defined. Now TritonRoute handles the connectivity in two ways.

- *Access Point(AP)* : An on-grid point on the metal of the route guide, and is used to connect to lower-layer segments, pins or IO ports,upper-layer segments. Access Point Cluster(APC) : A union of all the Aps derived from same lower-layer segment, a pin or an IO port, upper-layer guide.

***TritonRoute run for routing***

Make sure the CURRENT_DEF is set to pdn.def

- Start routing by using

```bash
run_routing
```
![Screenshot from 2023-09-18 22-13-25](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/282e12d1-5c3a-44d4-b208-91a59babc2b1)

Layout in magic tool post routing:

![Screenshot from 2023-09-18 22-34-59](https://github.com/Priyanshiiitb/Physicaldesign-openlane/assets/140998626/c5c86eb1-6b90-4ebe-9bb1-6ea19d899b58)

## Openlane Interactive flow:

```
cd /home/priyansh/OpenLane/

./flow.tcl -interactive
package require openlane 0.9
prep -design picorv32a
run_synthesis
run_floorplan
detailed_placement
run_cts
gen_pdn
run_routing
```
## OpenLANE non-interactive flow

```
cd /home/priyansh/OpenLane 
make mount
./flow.tcl -design picorv32a
```


## Acknowledgement

- Kunal Ghosh,VSD Corp.Pvt.Ltd.
- ChatGPT
- Shant Rakshit,Colleague,IIIT-B
- Emil J. Lal , Colleague, IIITB

## Reference

- https://www.vsdiat.com
- https://github.com/Devipriya1921/Physical_Design_Using_OpenLANE_Sky130
- https://github.com/nickson-jose/vsdstdcelldesign

