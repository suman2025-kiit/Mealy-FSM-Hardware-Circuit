# Mealy-FSM-Hardware-Circuit
### Mealy-FSM Circuit Validation with Holo-Circuit###

Step-by-Step QSPICE Workflow
Validate Mealy Machine Outputs vs Hardware Circuit Outputs (TX_EN / TRIGGER / ALERT / ENC_VALID)

Step 0: Prepare Signals and Naming
Decide the exact validation vector and node names. Minimum recommended:
•	Inputs (digital): VD_D, VP_D, ERR_D, CLK_D (plus any paper-specific signals).
•	DUT outputs: TRIGGER_HW, TX_EN_HW, ENC_HW (ENC_VALID), ALERT_HW.
•	Expected outputs: TRIGGER_EXP, TX_EN_EXP, ENC_EXP, ALERT_EXP.
•	Mismatch: MIS_TRIG, MIS_TX, MIS_ENC, MIS_ALR, MIS_ANY.

Step 1: Build/Import the Real Hardware DUT as a Subcircuit
In QSPICE, export your existing real-time platform schematic into a .SUBCKT netlist (e.g., HOLOGRAM_PLATFORM_TOP.sub). Ensure the port order is clearly defined.
; Include the exported DUT subcircuit
.include "HOLOGRAM_PLATFORM_TOP.sub"

Step 2: Create Input Stimuli (Analog or Digital)
Use PWL/PULSE sources to generate realistic transitions for depth-valid, pose-valid, error, and clock. If your sensors are analog, drive analog nodes and digitize them in Step 3.
; Example stimuli (edit timing to match your test cases)
VVD  vd  0 PWL(0 0  30u 0  40u 3.3  90u 3.3  100u 0)
VVP  vp  0 PWL(0 0  50u 3.3 80u 3.3  85u 0)
VERR err 0 PWL(0 0  140u 3.3 150u 0)
VCLK clk 0 PULSE(0 3.3 0 1n 1n 5u 10u)

Step 3: Digitize Analog Signals (Comparators / Behavioral Sources)
Convert analog inputs to clean logic rails (0/VDD). Use the same thresholds as the hardware comparators in the paper to avoid false mismatches.
.param VDD=3.3  VTH=1.65
BVD   VD_D   0  V=if(V(vd) > VTH, VDD, 0)
BVP   VP_D   0  V=if(V(vp) > VTH, VDD, 0)
BERR  ERR_D  0  V=if(V(err)> VTH, VDD, 0)
BCLK  CLK_D  0  V=if(V(clk)> VTH, VDD, 0)

Step 4: Instantiate the Real DUT (Replace Xdut)
Replace the placeholder Xdut line with the actual platform subcircuit. Make sure pin mapping matches exactly (inputs then outputs).
; DUT instance (example pin order)
Xdut VD_D VP_D CLK_D ERR_D  TRIGGER_HW TX_EN_HW ENC_HW ALERT_HW  HOLOGRAM_PLATFORM_TOP

Step 5: Implement the Golden Mealy Machine (δ and λ)
Implement the Mealy machine in parallel: (i) a state register (q bits) updated on CLK edges, (ii) next-state combinational logic δ(q,x), and (iii) output logic λ(q,x). Edit the equations to match the paper’s state encodings and transition rules.
; Example: fused-valid g(t)=VD_D & VP_D (edit if paper differs)
Bg   G 0 V={VDD*u(V(VD_D)-1)*u(V(VP_D)-1)}

; Decode Fusion and OutputStable states (example encodings)
BFUSION FUS 0 V={VDD*(1-u(V(q2)-1))*u(V(q1)-1)*u(V(q0)-1)}     ; 011
BSTABLE STB 0 V={VDD*u(V(q2)-1)*(1-u(V(q1)-1))*(1-u(V(q0)-1))} ; 100

; Outputs λ(q,x) (example mapping)
BTRIGGER_EXP TRIGGER_EXP 0 V={VDD*u(V(FUS)-1)*u(V(G)-1)}
BALERT_EXP   ALERT_EXP   0 V={VDD*u(V(ERR_D)-1)}
BTX_EN_EXP   TX_EN_EXP   0 V={VDD*u(V(STB)-1)*u(V(CLK_D)-1)}
BENC_EXP     ENC_EXP     0 V={VDD*u(V(TX_EN_EXP)-1)}

Step 6: Compare DUT vs Expected (Mismatch Flags)
Generate a mismatch flag for each output using XOR (digital) or absolute difference thresholding. Then OR them into a single MIS_ANY flag.
.param VTH=1.65
BMIS_TRIG MIS_TRIG 0 V=if(abs(V(TRIGGER_HW)-V(TRIGGER_EXP))>VTH, VDD, 0)
BMIS_TX   MIS_TX   0 V=if(abs(V(TX_EN_HW)-V(TX_EN_EXP))>VTH, VDD, 0)
BMIS_ENC  MIS_ENC  0 V=if(abs(V(ENC_HW)-V(ENC_EXP))>VTH, VDD, 0)
BMIS_ALR  MIS_ALR  0 V=if(abs(V(ALERT_HW)-V(ALERT_EXP))>VTH, VDD, 0)

BMIS_ANY MIS_ANY 0 V=if(V(MIS_TRIG)>VTH | V(MIS_TX)>VTH | V(MIS_ENC)>VTH | V(MIS_ALR)>VTH, VDD, 0)

Step 7: Run Measurements for PASS/FAIL and Mismatch Count
Use .meas to compute MIS_MAX (must be ~0 for pass), and MIS_CNT (approx mismatched cycles). PASS is 1 if MIS_MAX < VTH.
.param TCLK=10u  TSTOP=200u
.meas tran MIS_MAX  max   V(MIS_ANY) from=0 to=TSTOP
.meas tran MIS_AREA integ V(MIS_ANY) from=0 to=TSTOP
.meas tran MIS_CNT  param='MIS_AREA/(VDD*TCLK)'
.meas tran PASS     param='(MIS_MAX<VTH)'

.tran 0 TSTOP

Step 8: Generate Validation Outputs (Waveforms / Screenshots)
For reporting (paper-ready), capture these plots from QSPICE:
•	Normal/Real case: TRIGGER_HW vs TRIGGER_EXP, TX_EN_HW vs TX_EN_EXP, ENC_HW vs ENC_EXP, ALERT_HW vs ALERT_EXP.
•	Fraud/Attack case: show injected ERR/glitch and MIS_ANY rising to 1 during the fault window.
•	MIS_ANY plot + QSPICE .meas results (MIS_MAX, MIS_CNT, PASS) in the simulation log.
Step 9: Practical Notes (Avoiding False Mismatches)
•	Sample at the same clock edge used by the DUT (rising/falling).
•	If the DUT registers outputs (latency), delay expected outputs by the same cycles before comparing.
•	Use the same thresholds/hysteresis as hardware digitizers to avoid spurious toggles.
•	If Mealy outputs glitch within a cycle, compare only at sampling edges or register outputs.


