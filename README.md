# VSD OpenLane Codespace Lab

This repo is my running VLSI practice log using the VSD OpenLane Codespace environment. The Codespace setup installs OpenLane, Magic, the Sky130 PDK, and noVNC support so the flow can be run from the browser without a local EDA install.

When the container build finishes, the setup log shows `[VSD OPENLANE CODESPACE setup]`. Open a terminal, start OpenLane from `~/Desktop/OpenLane`, and use the noVNC desktop on port `6080` when a GUI layout viewer is needed.

![Codespace Log](images/02_codespaceLog.jpg)
![Setup Complete](images/03_CodespaceCreationCompleteAndOpenTerminal.jpg)

### Codespace setup notes

The Codespace is started from GitHub with **Create Codespace on main**. During setup, the container builds the Linux environment, installs the Sky130 PDK, and configures OpenLane, Magic, and VNC support.

![Launch Codespace](images/01_launchCodeSpace.jpg)

To check that the OpenLane install is working, I used the included test flow:

```bash
cd ~/Desktop/OpenLane
make test
```

A successful setup reports:

```text
[SUCCESS]: Flow complete.
[INFO]: There are no setup or hold violations in the design.
```

![Test OpenLane](images/04_testOpenlaneInstallation.jpg)
![Test Design and VNC](images/05_TestDesignSucessfulyCompleteAndStepsToOpenVNC.jpg)

For GUI layout viewing, the Codespace exposes noVNC on port `6080`. Opening that forwarded port launches the VSD desktop environment. If a directory listing appears first, `vnc_lite.html` or `vnc.html` opens the XFCE desktop.

![noVNC Desktop](images/06_openVNC.jpg)
![vnc lite](images/07_vnc_lite.jpg)

I also used Magic as a quick layout-viewing check with the test design:

```bash
cd ~/Desktop/OpenLane/designs/spm/runs/openlane_test/results/final
magic -T /home/vscode/.ciel/sky130A/libs.tech/magic/sky130A.tech
```

Inside Magic, the LEF and DEF can be loaded from the console:

```tcl
lef read ../../tmp/merged.nom.lef
def read ./def/spm.def
```

![Launch Magic](images/08_DirectoryStructureAndLaunchMagic.jpg)
![LEF Read](images/09_lefRead.jpg)
![DEF Read](images/10_defRead.jpg)
![Layout View](images/11_layout.jpg)

For custom designs such as `picorv32a`, the design folder needs a `config.tcl` file and a `src/` directory for RTL and constraint files. A complete non-interactive flow can be launched from the mounted OpenLane container with:

```bash
cd ~/Desktop/OpenLane
make mount
./flow.tcl -design picorv32a -verbose 1
```

After a full custom-design run, the final routed layout can be opened from the run's `results/final` directory with Magic and the generated LEF/DEF files.

![Steps to Run Custom Design](images/12_StepsToRunCustomDesignLikePicorv32a.jpg)
![Open Layout for Custom Design](images/13_StepsToOpenLayoutForCustomDesignLikePicorv32a.jpg)
![Final Layout for Custom Design](images/14_FinalLayoutForCustomDesignLikePicorv32a.jpg)

---

## 1. OpenLane Synthesis and Flop Ratio Calculation

In this section, I ran OpenLane synthesis for the `picorv32a` design. The synthesis step converted the RTL into a gate-level netlist mapped to Sky130 standard cells. This section only covers synthesis and the flop ratio calculation from the generated synthesis reports.

I started OpenLane in interactive mode:

```bash
cd ~/Desktop/OpenLane
make mount
cd /openlane
./flow.tcl -interactive
```

Inside the OpenLane Tcl prompt, I loaded OpenLane, prepared the `picorv32a` design with a fixed run tag, and ran synthesis:

```tcl
package require openlane 0.9
prep -design picorv32a -tag section1_synth -overwrite
run_synthesis
exit
```

After synthesis completed, I inspected the synthesis reports:

```bash
grep -R "Number of cells:" designs/picorv32a/runs/section1_synth/reports/synthesis
grep -R "sky130_fd_sc_hd__df" designs/picorv32a/runs/section1_synth/reports/synthesis
```

The cell count reports showed:

```text
1-synthesis_pre.stat:   Number of cells: 16844
1-synthesis_dff.stat:   Number of cells: 18508
1-synthesis.AREA_0.stat.rpt:   Number of cells: 15762
```

The D flip-flop count from the mapped reports was:

```text
1-synthesis_dff.stat:     sky130_fd_sc_hd__dfxtp_2     1613
1-synthesis.AREA_0.stat.rpt:     sky130_fd_sc_hd__dfxtp_2     1613
```

For the final flop ratio calculation, I used the final mapped synthesis report, `1-synthesis.AREA_0.stat.rpt`.

```text
Number of D flip-flops = 1613
Total number of cells = 15762

Flop Ratio = 1613 / 15762 = 0.102334729
DFF Percentage = 10.2334729%
```

Rounded final values:

```text
Flop Ratio = 0.1023
DFF Percentage = 10.23%
```

---

## 2. Floorplan and Placement for `picorv32a`

For this run, I used OpenLane v1.0.2 and continued with the same `picorv32a` design. This section records the floorplanning and placement steps, the generated result directories, the Magic viewing commands, and the die area calculation from the DEF output.

I started from the OpenLane working directory and entered the interactive flow:

```bash
cd ~/Desktop/OpenLane
make mount
cd /openlane
./flow.tcl -interactive
```

Inside OpenLane, I loaded the package, prepared a fresh run tag, and ran synthesis, floorplan, and placement:

```tcl
package require openlane
prep -design picorv32a -tag section2_place -overwrite
run_synthesis
run_floorplan
run_placement
```

This command sequence created a clean run named `section2_place`. The `run_synthesis` step generated the mapped gate-level synthesis outputs, `run_floorplan` generated the initial physical floorplan DEF, and `run_placement` generated the placed DEF with standard cells assigned to legal rows.

The floorplan step generated its outputs in:

```text
/openlane/designs/picorv32a/runs/section2_place/results/floorplan
```

The placement step generated its outputs in:

```text
/openlane/designs/picorv32a/runs/section2_place/results/placement
```

In this OpenLane version, both result directories use the DEF filename `picorv32a.def`. The output was not named `picorv32a.floorplan.def` or `picorv32a.placement.def`.

To view the floorplan in Magic, I ran the following command from the floorplan result directory:

```bash
cd /openlane/designs/picorv32a/runs/section2_place/results/floorplan
magic -T /home/vscode/.ciel/sky130A/libs.tech/magic/sky130A.tech lef read ../../tmp/merged.nom.lef def read picorv32a.def &
```

The LEF file used for viewing was:

```text
../../tmp/merged.nom.lef
```

The floorplan view shows the die boundary, core area, IO/pin locations, rows, and early physical structure before standard cells are placed into their final legal positions.

![Section 2 floorplan in Magic](images/section2_floorplan_magic.png)

The DEF file reported this die area:

```def
DIEAREA ( 0 0 ) ( 1279175 1289895 ) ;
```

The DEF database units are converted to microns by dividing by `1000`.

```text
Die width  = 1279175 / 1000 = 1279.175 um
Die height = 1289895 / 1000 = 1289.895 um
Die area   = 1279.175 * 1289.895 = 1,649,991.51 um^2
```

To view placement in Magic, I ran the same Magic command from the placement result directory:

```bash
cd /openlane/designs/picorv32a/runs/section2_place/results/placement
magic -T /home/vscode/.ciel/sky130A/libs.tech/magic/sky130A.tech lef read ../../tmp/merged.nom.lef def read picorv32a.def &
```

The placement view shows the standard cells assigned to placement rows inside the core. At this point, placement has happened, but detailed routing has not happened yet, so the view is still before the final routed layout stage.

![Section 2 placement in Magic](images/section2_placement_magic.png)

---

## Section 4: Timing Analysis and Clock Tree Synthesis

For this section, I started a new OpenLane run for the default `picorv32a` design using the run tag `section4_no_custom`. I intentionally skipped the custom inverter and custom standard-cell insertion path from Section 3 because my goal here was to understand the standard RTL-to-GDS flow, not custom standard-cell design.

I started the interactive OpenLane flow and prepared a clean run:

```tcl
package require openlane
prep -design picorv32a -tag section4_no_custom -overwrite
```

Before running synthesis, I set timing-focused synthesis options:

```tcl
set ::env(SYNTH_STRATEGY) "DELAY 3"
set ::env(SYNTH_SIZING) 1
```

Then I ran the main implementation steps through post-CTS timing analysis:

```tcl
run_synthesis
run_floorplan
run_placement
run_cts
run_post_cts_sta
```

Before CTS, the clock is treated more ideally during timing analysis. After CTS, OpenLane inserts a real buffered clock tree so the clock can be physically distributed to the sequential elements in the placed design.

CTS generated the post-CTS DEF at:

```text
/openlane/designs/picorv32a/runs/section4_no_custom/results/cts/picorv32a.def
```

I opened the post-CTS DEF in Magic using the merged LEF and CTS DEF from the Section 4 run:

```text
/openlane/designs/picorv32a/runs/section4_no_custom/tmp/merged.nom.lef
/openlane/designs/picorv32a/runs/section4_no_custom/results/cts/picorv32a.def
```

![Section 4 post-CTS layout in Magic](images/section4_post_cts_magic.png)

The post-CTS timing results were:

```text
TNS = 0.00
WNS = 0.00
Worst setup slack = 3.73 ns
Worst hold slack  = 0.05 ns
Clock skew        = 0.41 ns
```

![Section 4 CTS timing summary](images/section4_cts_timing_summary.png)

The CTS report showed:

```text
Clock roots = 1
Clock buffers inserted = 385
Clock subnets = 385
Clock sinks = 1613
Leaf buffers = 342
Clock tree path depth = 4 to 5
Max clock tree level = 5
```

The area and power reports showed:

```text
Design area = 283214 µm²
Utilization = 13%
Total power = 0.130 W
Sequential power = 0.0109 W, 8.3%
Combinational power = 0.120 W, 91.7%
```

The main takeaway is that CTS converts the design from an ideal-clock placement result into a more physically realistic implementation with an inserted clock distribution network. The design still met timing after CTS, with no negative setup or hold slack. This step is directly relevant to future RTL-to-GDS work because timing closure depends not only on RTL and synthesis, but also on physical placement, clock distribution, skew, and post-CTS STA.

---

## Section 5: Post-Placement and Final Routing Results

For this section, I completed the later OpenLane physical design stages for the default `picorv32a` flow. I collected post-placement statistics and verified that detailed routing completed successfully with zero routing violations.

### Placement / Post-Placement Statistics

| Metric | Value |
| --- | ---: |
| Total instances | 25,355 |
| Fixed instances | 7,708 |
| Nets | 17,749 |
| Design area | 517,276.1 µm² |
| Fixed area | 10,965.5 µm² |
| Movable area | 213,417.2 µm² |
| Utilization | 42% |
| Utilization padded | 60% |
| Rows | 264 |
| Row height | 2.7 µm |
| Original HPWL | 895,297.0 µm |
| Legalized HPWL | 910,806.5 µm |
| HPWL delta after legalization | 2% |
| Mirrored instances | 6,650 |
| HPWL after mirroring | 895,297.0 µm |
| Final HPWL delta after mirroring | -1.7% |

![Section 5 placement statistics](images/section5_post_placement_stats.png)

### Final Detailed Routing Statistics

| Metric | Value |
| --- | ---: |
| Final routing violations | 0 |
| Total wire length | 1,103,187 µm |
| Total vias | 145,509 |
| Detailed-route CPU time | 00:29:13 |
| Detailed-route elapsed time | 00:32:37 |
| Peak memory | 854.19 MB |

![Section 5 detailed routing results](images/section5_detailed_route_results.png)

### Wire Length by Layer

| Layer | Wire length |
| --- | ---: |
| li1 | 2,639 µm |
| met1 | 483,058 µm |
| met2 | 482,317 µm |
| met3 | 122,196 µm |
| met4 | 12,976 µm |
| met5 | 0 µm |

### Via Breakdown

| Layer | Vias |
| --- | ---: |
| li1 | 60,472 |
| met1 | 78,129 |
| met2 | 6,549 |
| met3 | 359 |
| met4 | 0 |

These results show that the design successfully completed detailed routing with zero DRC routing violations. Most of the routing demand was concentrated on `met1` and `met2`, while the higher metal layers were used much less heavily. The placement stage reported 42% utilization, increasing to 60% with padding, and legalization only increased HPWL by about 2%. Instance mirroring recovered the HPWL back to the original estimate, giving a final HPWL delta of about -1.7%.
