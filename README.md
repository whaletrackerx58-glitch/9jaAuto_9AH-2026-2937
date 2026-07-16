PLC Engineering Design Document: 9JA Automation Hub Capstone SolutionPlatform: Siemens TIA PortalProject Name: 9jaAuto_9AH-2026-2937. Date of Submission: July 16, 20261. System Block Architecture & Project TreeThe automation system is split into two physical S7-1500 controller stations and an HMI operator interface panel. The system architecture layout within TIA Portal is engineered as follows:PlaintextProject: 9jaAuto_xxxx
│
├── PLC_Section1 (CPU 1516-3 FNI-0AB0)
│   ├── Program Blocks
│   │   ├── OB1 [Main] - Core Cyclic Executive
│   │   └── OB30 [Cyclic Interrupt] - 2000ms Execution Interval
│   └── PLC Data Types
│       ├── UDT_ProcessData [User Defined Type]
│       └── DB_ProcessData [Global DB - Array 1..50 of UDT]
│
├── PLC_TankControl (CPU 1513-1 AL02-0AB0)
│   ├── Program Blocks
│   │   ├── OB1 [Main] - Calls Tank Control Function Block
│   │   ├── FB_TankLevelControl [FB1] - Core Tank Processing Logic
│   │   └── DB_TankLevelControl_Instance [DB2] - Instance Data Block
│   └── PLC Tags
│       └── Tag_Table_TankControl
│
└── HMI_TankControl (TP700 Comfort - 6AV2 124-0GC01-0AX0)
    └── Screens
        └── Tank_Level_Control [Main Operating Interface]
2. Section 1: Hardened PLC Infrastructure & Memory Formats2.1 Bi-Stable Flip-Flop (Toggle Logic) NarrativeTo achieve a reliable latching pulse where a single push-button toggles an output variable state dynamically, a positive-edge transition check must be executed.Logic Rule:$$\text{M\_FF\_RisingEdge} = \text{I\_FF\_PB} \cdot \overline{\text{M\_FF\_LastState}}$$Execution: On a valid rising edge execution, the memory bit Q_FF_Output state toggles inversion state via a dynamic NOT assignment window.2.2 Global DB Structure Memory ProfileThe global database DB_ProcessData leverages S7 Optimized Block Access. This removes old data-word/byte memory tracking offsets, utilizing direct name-space assignments for execution speed.User Defined Type Layout (UDT_ProcessData)DeviceID : Integer (Int)DeviceName : String [20 Bytes]InService : Boolean (Bool)Faulted : Boolean (Bool)RawValue : Integer (Int)ScaledValue : Floating Point (Real)Setpoint : Floating Point (Real)OutputValue : Floating Point (Real)RuntimeHours : Double Integer (DInt)LastUpdate_ms : Time Format (Time)Engineering Note: DB_ProcessData allocates an Array element block bounds matching Array[1..50] of UDT_ProcessData, completely satisfying the structural requirements.3. Section 2: Tank Level Control Architecture3.1 PLC Input/Output Tag Assignment TableThe explicit memory mapped interface links physical fieldwork sensors and actuators directly to data markers exposed to the HMI via PROFINET networking:Tag NameData TypeTIA Map AddressHMI AccessFunctional Engineering DescriptionLT001_RawInt%IW96NoRaw peripheral analog input from level transmitter (0–27648)LT001_LevelPctReal%MD20YesMathematically scaled current level value (0.0–100.0%)LT001_InServiceCmdBool%M10.0YesSoftware command toggle generated from HMI screen buttonLT001_InServiceBool%M10.1YesDefinitive active operational state indicator of level probeLT001_FailedBool%I0.2YesPhysical trip signal/Fault status bit from level transmitterLCV100_ControlEnableCmdBool%M11.0YesRemote software command from operator interface panelLCV100_ControlEnabledBool%M11.1YesInterlocked runtime permissive logic verifying system is safeLCV100_OpenPctReal%MD24YesComputational valve target position setpoint (0.0–100.0%)LCV100_AO_RawInt%QW96NoOutput peripheral analog raw word mapped to Control ValveLCV100_LowLevelLockoutBool%M12.0YesHysteresis safety bit active when fluid drops below 5.0%LCV100_PipeGreenBool%M12.1YesDynamic animation graphic state relay for HMI UI pipingSystem_FaultBool%M1.0YesGlobal alarm grouping master indicator bit3.2 Engineering Control Philosophy & Formulaic BreakdownThe control routine transforms raw industrial signal words to standardized scaled ranges, implementing strict safety interlocks.A. Analog Scaling FormulasThe peripheral field transmitter provides raw feedback spanning $0$ to $27648$ units. The scaling algorithm applies the following linear formula to map this to engineering units ($0.0\%$ to $100.0\%$):$$\text{LT001\_LevelPct} = \frac{\text{INT\_TO\_REAL}(\text{LT001\_Raw}) \times 100.0}{27648.0}$$Similarly, the valve command output converts a $0.0\%$ to $100.0\%$ floating-point demand back into standard S7 analog counts:$$\text{LCV100\_AO\_Raw} = \text{REAL\_TO\_INT}\left(\frac{\text{LCV100\_OpenPct} \times 27648.0}{100.0}\right)$$B. Hysteresis Low-Level Lockout Safety LoopTo protect pumps and downline instrumentation from short-cycling or running dry, a clear deadband envelope is established:Trip Boundary: If $\text{LT001\_LevelPct} < 5.0\%$, the safety flag LCV100_LowLevelLockout latches TRUE.Reset Reset Boundary: The lockout flag remains strictly active until fluid levels recover safely past the threshold: $\text{LT001\_LevelPct} \ge 20.0\%$.C. Proportional Control Curve CalculationsWhen the system is operational and the fluid level reaches $\ge 70.0\%$, the valve opens proportionally based on the following linear slope equation:$$\text{LCV100\_OpenPct} = 65.0 + \left[(\text{LT001\_LevelPct} - 70.0) \times \frac{35.0}{30.0}\right]$$4. HMI Control Room UI Layout PlanThe TP700 Comfort Panel graphic elements link directly to the database tags. Below is the layout configuration design:Plaintext===================================================================================
                               TANK LEVEL CONTROL SCREEN
===================================================================================

     [   LT001 LEVEL DISPLAY   ]                   [  SYSTEM DIAGNOSTICS/ALARMS  ]
     +-------------------------+                   +-----------------------------+
     |        xxx.x %          |                   |  [ ] LT001 Healthy          |
     +-------------------------+                   |  [ ] LT001 Failed           |
                                                   |  [ ] Low Level Lockout      |
          /===============\                        |  [ ] Master System Fault    |
         /                 \                       +-----------------------------+
        |   ============    |
        |   |  FLUID   |    |                     [     OPERATOR COMMANDS     ]
        |   |  LEVEL   |    |                     +-----------------------------+
        |   | ANIMATION|    |                     |  [ LT001 IN-SERVICE TOGGLE ]|
        |   ============    |                     |  (Green=ON, Grey=OFF,       |
        |                   |                     |   Red Flash=Hardware Fault) |
         \                 /                      |                             |
          \===============/                       |  [ VALVE CONTROL ENABLE ]   |
                  ||                              |  (Green=ACTIVE, Grey=OFF)   |
                  ||                              +-----------------------------+
                  ||
   ===============[#]=========================>   [      LCV100 VALVE STATUS    ]
   Pipe Dynamic Color Element:                    +-----------------------------+
   [ LCV100_PipeGreen = Green if Open > 2% ]      |  Opening Level:  xxx.x %     |
   [ LCV100_PipeGreen = Red if Open <= 2%  ]      +-----------------------------+
===================================================================================
5. Verification & Final Commissioning ChecklistThis checklist tracks and verifies programmatic output variables across all test cases prior to system commissioning:Test SequenceSimulated Inputs & ConditionsExpected Output Parameters & StatesStatus01LT001_Failed = TRUELT001_InService forced FALSE; Valve drops to 0.0% immediately; HMI indicator turns Red.[ ] Passed02LT001_InServiceCmd = FALSEValve drops to 0.0%; HMI toggle switches to Grey color state.[ ] Passed03LCV100_ControlEnableCmd = FALSEValve drops to 0.0%; Valve control animation shifts to Grey mode.[ ] Passed04Tank level drops $< 5.0\%$LCV100_LowLevelLockout latches TRUE; Valve output clamps to 0.0%.[ ] Passed05Tank level rises to $10.0\%$ (Post-Lockout)Valve output must remain strictly at 0.0% (Lockout condition holds).[ ] Passed06Tank level rises to $\ge 20.0\%$Lockout latch drops out to FALSE; system readies for process activation.[ ] Passed07Tank level at $50.0\%$, Normal OperationsValve output tracks at 0.0% (Level is below the $70.0\%$ execution slope line).[ ] Passed08Tank level at exactly $70.0\%$Valve output calculates to exactly $65.0\%$.[ ] Passed09Tank level at exactly $85.0\%$Valve output calculates to exactly $82.5\%$.[ ] Passed10Tank level reaches maximum $100.0\%$Valve output calculates to exactly $100.0\%$ full open path.[ ] Passed11Valve output command $> 2.0\%$LCV100_PipeGreen maps high; flow pipework graphic turns Green.[ ] Passed12Valve output command $\le 2.0\%$LCV100_PipeGreen drops low; flow pipework graphic turns Red.[ ] Passed
