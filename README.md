# BB Hive Robot Control
### Applying Busy Beaver State Machines to Robot Body Control

---

## 1. What is Busy Beaver

The Busy Beaver problem asks: given a Turing machine with N states, what is the maximum useful work it can do before halting?

Key known values:

| N states | Max 1s written | Max steps |
|----------|---------------|-----------|
| 1 | 1 | 1 |
| 2 | 4 | 6 |
| 3 | 6 | 21 |
| 4 | 13 | 107 |
| 5 | ≥4098 | ≥47,176,870 |

The function is **uncomputable** — it grows faster than any algorithm can calculate.
BB(5) was only confirmed in 2024. BB(6) touches set theory.

---

## 2. The Core Insight for Robotics

BB reframed for robots:

> *Given N internal states, what is the most capable behaviour you can produce?*

The discipline: **resist adding complexity to the message. Add states instead. Let complexity emerge from how states connect.**

---

## 3. How a BB Machine Works

Each state does exactly three things:

```
1. READ   → read current tape cell (0 or 1)
2. WRITE  → write 0 or 1 to that cell
3. MOVE   → shift tape head L or R
4. GOTO   → jump to next state
```

Transition table format:

```
STATE  READ  WRITE  MOVE  NEXT
──────────────────────────────
A      0     1      R     B
A      1     1      L     B
B      0     1      L     A
B      1     1      R     HALT
```

**The tape is not a stack.** It is flat, infinite both ways, nothing is consumed when read. It is a shared whiteboard — any machine can walk back and re-read previous cells.

```
... [ 0 ][ 0 ][ 1 ][ 0 ][ 1 ][ 0 ] ...
                    ▲
                   head
```

---

## 4. The Robot

A simple wheeled robot with one arm:

**Drive:** differential drive, 2 motors (left/right)
- Combined states: forward, back, turn left, turn right, spin, stop

**Arm:** 3 DOF
- Joint 1: base rotation
- Joint 2: shoulder
- Joint 3: elbow
- Each joint: servo, commanded to a position (section 25, Stage 7; DC motor with PID is a later option)

**Gripper:** 3-claw screw drive, 1 motor
- Open, close, hold
- Screw drive is backdrive-proof — motor off = grip holds mechanically
- Stall current detection = grip achieved without force sensor

Total actuators: 6
Naive state combinations: 3^6 = 729 (mostly garbage)

---

## 5. Layered Control Architecture

Instead of one giant state machine, break into layers. Each layer has its own BB problem.

```
[ TASK LEVEL ]        "pick up the cup"
      │
[ BEHAVIOUR LEVEL ]   "approach → grab → carry → release"
      │
[ MOTION LEVEL ]      "extend joint2, close gripper..."
      │
[ ACTUATOR LEVEL ]    "motor A: pulse on"
```

Each layer asks: **minimum states needed here for maximum capability?**

The body handles some of it — the screw drive self-limits grip force, wheels naturally track. Less brain needed.

---

## 6. Gripper State Machine (Pre-BB)

Before applying strict BB rules, the gripper logic as a conventional state machine:

```
                        [grab signal]
                             │
                             ▼
                    ┌─────────────────┐
                    │     CLOSING     │◄──────────────────┐
                    │  motor: forward │                   │
                    │  start timer    │                   │
                    └────────┬────────┘                   │
                             │                            │
              ┌──────────────┼──────────────┐             │
              │              │              │             │
    [current>=STALL]  [current>=GRIP]  [timer expires]   │
              │              │              │             │
              ▼              ▼              ▼             │
         ┌────────┐   ┌─────────────┐  ┌────────┐        │
         │ FAULT  │   │   HOLDING   │  │ FAULT  │        │
         │  F2    │   │  motor: off │  │  F1    │        │
         └────────┘   │  watchdog   │  └────────┘        │
                      └──────┬──────┘                    │
                             │                           │
           ┌─────────────────┼──────────────┐            │
           │                 │              │            │
      [current drop]   [current spike] [watchdog]        │
           │                 │         expires           │
           ▼                 ▼              │            │
    ┌─────────────┐  ┌──────────────┐       ▼            │
    │    NUDGE    │  │  BACKING OFF │   ┌────────┐       │
    │ short fwd   │  │ short rev    │   │ FAULT  │       │
    │ pulse       │  │ pulse        │   │  F4    │       │
    │ inc retries │  └──────┬───────┘   └────────┘       │
    └──────┬──────┘         │                            │
           │                │                            │
  [current OK]        [current OK]                       │
           │                │                            │
           └────────────────┘                            │
                    │                                    │
                    ▼                                    │
             back to HOLDING ───────────────────────────►┘
                    │
              [release signal]
                    │
                    ▼
           ┌─────────────────┐
           │    OPENING      │
           │  motor: reverse │
           └────────┬────────┘
                    │
              [open limit]
                    │
                    ▼
           ┌─────────────────┐
           │      IDLE       │
           └─────────────────┘
```

9 states. Two grip thresholds:

| Threshold | Meaning |
|-----------|---------|
| GRIP_THRESHOLD | enough grip — stop here |
| STALL_THRESHOLD | too much — back off |

Fault codes:

| Code | Meaning | Layer above response |
|------|---------|---------------------|
| F1 | timeout on close, nothing grabbed | reposition arm, retry |
| F2 | hard stall, hit obstacle | abort task |
| F3 | drop detected, retries exhausted | slow motion, regrip |
| F4 | held too long | check task logic |

---

## 7. The Hive Architecture

Split gripper control into three separate BB machines, each with one job:

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│   FEEL BB   │   │ STRENGTH BB │   │  OPEN/CLOSE │
│             │   │             │   │     BB      │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                  ┌──────▼──────┐
                  │  GROUP BB   │
                  │ (arbitrates)│
                  └─────────────┘
```

Each BB only speaks 0/1. The GROUP BB does not micromanage — it resolves conflicts only.

---

## 8. Shared Tape Layout

All machines communicate via one shared tape. No pulses between machines. No messages. They read what others wrote.

```
position:  0            1          2             3
          [SENSOR_GRIP] [FEEL_OUT] [STRENGTH_OUT] [OPENCLOSE_OUT]
```

This is the first four cells of the gripper only. The full 30-cell layout is in §23.

---

## 9. BB Transition Tables

### FEEL BB
Reads SENSOR cell. Writes to FEEL_OUT cell.

Every rule writes the value it reads. The anomaly information is held in the state, not the output. Filtering comes from the Schmitt thresholds (section 14), not from this table.

```
STATE  READ  WRITE  MOVE  NEXT
──────────────────────────────
A      0     0      R     A      normal, write OK
A      1     1      R     B      anomaly, flag it
B      0     0      L     A      recheck, clear
B      1     1      L     A      confirmed problem
```

### STRENGTH BB
Reads FEEL_OUT cell. Writes to STRENGTH_OUT cell.

Meaning of STRENGTH_OUT on the tape: `1` = pulse train on (rate set by the motor layer), `0` = pulse train off. A single bit cannot carry a rate.

```
STATE  READ  WRITE  MOVE  NEXT
──────────────────────────────
A      0     0      R     A      feel OK, hold
A      1     1      R     B      feel bad, increase
B      0     1      L     A      still holding firm
B      1     0      L     A      backed off
```

### GROUP BB
Reads STRENGTH_OUT cell. Writes to GRIP_CMD cell (section 23).

Every rule writes the value it reads. The state decides when HALT happens. HALT means grip locked. On HALT the code sets `GRIP_LOCKED = 1`, `GRIP_DONE = 1` and turns the motor off. The screw drive holds the grip mechanically with the motor off.

```
STATE  READ  WRITE  MOVE  NEXT
──────────────────────────────
A      0     0      R     A      all good, no change
A      1     1      R     B      command motor
B      0     0      L     A      motor done
B      1     1      L     HALT   lock grip
```

---

## 10. Full BB Map

```
SHARED TAPE
┌──────────┬──────────┬────────────┬──────────┐
│ 0:SENSOR │  1:FEEL  │ 2:STRENGTH │  3:MOTOR │
└──────────┴──────────┴────────────┴──────────┘
      │           │           │           │
      ▼           │           │           │
┌───────────┐     │           │           │
│  FEEL BB  │     │           │           │
│           │     │           │           │
│  A──────────────►B          │           │
│  │read 0  │  read 1         │           │
│  │write 0 │  write 1        │           │
│  │move R  │  move R         │           │
│  │        │                 │           │
│  B──────────────►A          │           │
│   read 0     read 1         │           │
│   write 0    write 1        │           │
│   move L     move L         │           │
└───────────┘     │           │           │
                  ▼           │           │
            ┌───────────┐     │           │
            │STRENGTH BB│     │           │
            │           │     │           │
            │  A──────────────►B          │
            │  │read 0  │  read 1         │
            │  │write 0 │  write 1        │
            │  │move R  │  move R         │
            │  │        │                 │
            │  B──────────────►A          │
            │   read 0     read 1         │
            │   write 1    write 0        │
            │   move L     move L         │
            └───────────┘     │           │
                              ▼           │
                        ┌───────────┐     │
                        │ GROUP BB  │     │
                        │           │     │
                        │  A ─0→ A    write 0, move R
                        │  A ─1→ B    write 1, move R  (command motor)
                        │  B ─0→ A    write 0, move L  (motor done)
                        │  B ─1→ HALT write 1, move L  (lock grip)
                        └───────────┘     │
                                          ▼
                                    ┌──────────┐
                                    │  MOTOR   │
                                    │  0 = off │
                                    │  1 = on  │
                                    └──────────┘
```

Chain reads top to bottom:

```
SENSOR fires
  → FEEL BB reads, writes to tape
    → STRENGTH BB reads, writes to tape
      → GROUP BB reads, writes to tape
        → MOTOR acts
          → or HALT (grip locked)
```

---

## 11. Strength as Rate Coding

The STRENGTH BB does not encode force as a value. It encodes it as **pulse frequency** — exactly how biological neurons encode grip strength.

```
WEAK    → pulse every 100ms   ░░░░░░░░░░
MEDIUM  → pulse every 50ms    █░█░█░█░█░
FIRM    → pulse every 20ms    ███░███░██
HOLD    → no pulses           ──────────
```

This is rate coding. The motor driver sees average power from pulse frequency. The BB machine never knows about voltage or duty cycle — it just decides how often to fire.

---

## 12. Key Principles

**Tape vs Stack**

| | tape | stack |
|--|------|-------|
| access | one cell, move L or R | only top |
| memory | infinite both ways | grows one way |
| history | still there, revisit | gone when popped |
| analogy | shared whiteboard | pile of plates |

**BB Elegance**

- Keep pulses dumb (0/1 only)
- Keep each state simple (few rules)
- Let the network of states produce complex behaviour
- Complexity lives in how states connect, not in what each message carries
- Resist adding data to the pulse — add a state instead

**Why not bytes per pulse?**

Sending a byte per pulse is just programming. You design the complexity in manually. Nothing emerges. BB's power comes from its constraint — small alphabet, simple rules, surprising complexity falls out of the graph structure.

---

## 13. Next Steps

```
1. Add OPEN/CLOSE BB
   → sits before FEEL on the tape

2. Add ARM layer above
   → GROUP BB output becomes a cell the arm layer reads

3. Add FAULT BB
   → watches for HALT states, handles recovery

4. Scale to full robot
   → drive BB hive
   → arm BB hive
   → task BB above all hives
```

---

*Session: 2026-10-27*

---

## 14. Hardware Implementation — Schmitt Triggers and 555 Circuits

BB machines at 2 states and binary alphabet map almost 1:1 to basic analog circuits.

### Schmitt Trigger = FEEL BB

A Schmitt trigger is a 2-state machine with hysteresis:

```
input rising past upper threshold  → output flips to 1
input falling past lower threshold → output flips to 0
```

This IS the FEEL BB transition table.

```
FEEL BB          Schmitt Trigger
───────────      ───────────────
state A → B      input > Vt_high
state B → A      input < Vt_low
write 1          output HIGH
write 0          output LOW
```

Hysteresis prevents chatter between states — critical for noisy current sensing.

### 555 Astable = STRENGTH BB

The 555 in astable mode generates pulses. Frequency controlled by RC values.
This is rate coding in hardware.

```
STRENGTH BB      555 astable
───────────      ───────────
pulse rate       frequency (RC)
hold             monostable latch
increase         lower R or C
```

Feed FEEL BB output as CV input → frequency changes with grip state automatically.

### Tape Cells = Flip-flops

```
each tape cell  →  D flip-flop
clock pulse     →  step signal
0/1 value       →  Q output
```

### Full Hive in Hardware

```
current sensor
      │
      ▼
[Schmitt trigger]  ← FEEL BB
      │
      ▼
[555 astable + CV] ← STRENGTH BB
      │
      ▼
[SR flip-flop]     ← GROUP BB
      │
      ▼
[motor driver]     ← MOTOR
```

No microcontroller needed. A handful of chips.

---

## 15. Discrete Circuits vs Arduino Nano

### Speed comparison

```
Discrete circuit:   1-10 nanoseconds   (speed of electrons)
Arduino Nano:       62 nanoseconds     (16MHz clock)
                    + code overhead
                    + interrupt latency
                    = realistically 4-100 microseconds
```

### For a gripper, speed is not the bottleneck

```
Screw drive motor:   moves in milliseconds
Human grip reflex:   ~150 milliseconds
Gripper full close:  50-200ms
```

Physics is thousands of times slower than either option.

### When discrete speed matters

```
✓ switching power supplies   (kHz-MHz switching)
✓ motor commutation          (BLDC timing)
✓ RF circuits
✓ safety cutoffs             (guaranteed response)
```

### When it does not matter

```
✗ gripper open/close         (mechanical limit)
✗ arm positioning            (servo speed)
✗ wheel drive                (inertia)
```

### Real reason to use discrete

Not speed. **Reliability.** A Schmitt trigger on the current sensor will never crash, hang, or have a firmware bug. That matters more than nanoseconds for safety-critical inputs like over-grip detection.

### Best of both worlds

```
[Schmitt trigger] ── raw sensor cleanup ──► Nano pin
                                              │
                                         Nano runs
                                         BB logic in
                                         software
                                              │
                                         PWM out ──► 555 ──► motor
```

Discrete for sensing and output. Nano for state logic.

---

## 16. Max BB Nodes on One Arduino Nano

### Nano specs

```
Clock:        16 MHz
RAM:          2KB (2048 bytes)
Flash:        32KB
Loop speed:   ~1000-4000 iterations/second realistically
```

### Cost per BB node

```
current state:    1 byte
transition table: 4 entries × 3 bytes = 12 bytes RAM
input pin:        1 byte
output pin:       1 byte
─────────────────────────
per node:         ~15 bytes RAM
```

### RAM limit

```
2048 bytes available
÷ 15 bytes per node
= ~136 nodes theoretical max

1024 bytes usable (reserve half for stack/variables)
÷ 15 bytes per node
= ~68 nodes comfortable
```

### Speed limit

```
16,000,000 cycles/sec
÷ 30 instructions per node
= 533,000 node updates/sec
÷ 1000 updates/sec minimum per node
= 533 nodes max by speed
```

### Real bottleneck: RAM not speed

```
theoretical:   136 nodes
comfortable:   68 nodes
practical:     30-40 nodes (leaving room for comms, sensors, PWM)
```

### Node budget for this robot

```
gripper hive:     3 nodes
arm hive:         6 nodes  (3 joints × 2)
drive hive:       2 nodes
group BBs:        3 nodes
task layer:       2 nodes
───────────────────────────
total:            ~16 nodes
```

One Nano handles the whole robot comfortably with room to spare.

---

*Session: 2026-10-28*

---

## 17. Separating and Layering Requirements — Fan Out as a Tree

The hardest design problem in the architecture. The answer is **functional decomposition** — split by what a function *knows about*, not what it *does*.

### The rule

```
each BB node knows about exactly ONE thing
```

A grip strength node knows nothing about arm position.
An arm position node knows nothing about the task.
A task node knows nothing about motors.

### Fan out as a tree

```
                    [ TASK BB ]
                         │
           ┌─────────────┼─────────────┐
           │             │             │
      [DRIVE BB]     [ARM BB]     [GRIPPER BB]
           │             │             │
      ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
  [LEFT]  [RIGHT] [J1] [J2] [J3] [OPEN] [STRENGTH] [FEEL]
    │        │     │    │    │      │       │          │
  [motor] [motor] [servo×3]      [screw] [555]    [schmitt]
```

Each level only talks to the level directly below it.

### How to separate requirements correctly

Ask three questions per function:

```
1. what does it sense?
2. what does it control?
3. what does it report upward?
```

If the answer touches more than one domain — split it.

### Example: gripper layer

```
FEEL BB
  senses:   current draw
  controls: nothing
  reports:  0=ok 1=problem

STRENGTH BB
  senses:   FEEL output
  controls: pulse rate
  reports:  0=holding 1=at limit

OPEN/CLOSE BB
  senses:   GROUP command
  controls: screw direction
  reports:  0=moving 1=at limit

GROUP BB
  senses:   STRENGTH + OPEN/CLOSE outputs
  controls: arbitrates conflicts
  reports:  0=busy 1=done or HALT
```

No overlap. Clean boundaries.

### Meshing vs fanning

Fan out = tree, one parent per node:

```
        TASK
       /    \
     ARM    DRIVE
    / | \
  J1  J2  J3
```

Mesh = multiple parents, used when two nodes share context:

```
     ARM ──── DRIVE
      \       /
       BALANCE BB
```

Use mesh sparingly. Only when two branches genuinely need to coordinate — like balance depending on both arm position and wheel speed simultaneously.

### When to add a mesh node

```
if node C needs to know about BOTH A and B
and A and B are in different branches
→ C is a mesh node, sits between both branches
→ give it its own tape cells
→ it reads from both, writes to one
```

### Full robot tree

```
TASK
├── DRIVE
│   ├── LEFT MOTOR BB
│   └── RIGHT MOTOR BB
├── ARM
│   ├── JOINT 1 BB
│   ├── JOINT 2 BB
│   ├── JOINT 3 BB
│   └── ARM GROUP BB
└── GRIPPER
    ├── FEEL BB
    ├── STRENGTH BB
    ├── OPEN/CLOSE BB
    └── GRIPPER GROUP BB
```

Only add mesh nodes when the tree cannot solve a problem — like if the arm needs to know the gripper has locked before it lifts.

---

## 18. Sideways Coordination — Siblings That Need to Talk

Pure tree breaks down when siblings need to coordinate. The arm needs to know the grip is locked before it lifts. The drive needs to know the arm is tucked before it turns.

### The problem

```
        TASK
       /    \
     ARM    GRIPPER
      │         │
    LIFT      LOCK
```

LIFT cannot start until LOCK is done. But they are siblings. The tree has no sideways wire.

### Solution 1 — Let the parent coordinate

TASK BB reads both ARM and GRIPPER tape cells. Only signals ARM to lift when GRIPPER reports locked.

```
TASK tape:
[ARM_CMD] [ARM_STATUS] [GRIP_CMD] [GRIP_STATUS]

TASK BB:
  read GRIP_STATUS
  if 1 (locked) → write ARM_CMD = 1 (lift)
  if 0 (not yet) → write ARM_CMD = 0 (wait)
```

Parent becomes the coordinator. Clean. No sideways wires.

### Solution 2 — Add a coordinator BB between siblings

```
        TASK
          │
     [SEQUENCE BB]
       /       \
     ARM        GRIPPER
```

SEQUENCE BB has one job — enforce order:

```
STATE  READ         WRITE       NEXT
A      GRIP=0       ARM_hold    A     waiting for grip
A      GRIP=1       ARM_go      B     grip locked, signal arm
B      ARM=1        done        HALT  arm lifted, done
```

### Solution 3 — Shared tape cell as a semaphore

Both nodes watch one shared cell:

```
tape: ... [GRIP_LOCKED] ...

GRIPPER GROUP BB writes 1 to GRIP_LOCKED when done
ARM BB reads GRIP_LOCKED before moving
  → 0 = wait
  → 1 = proceed
```

No new node needed. The tape cell IS the coordination signal.

### For this robot specifically

```
move arm while gripping → parallel, no dependency
lift after grip locked  → sequence, use semaphore cell
drive while arm moves   → parallel if arm tucked, use semaphore
```

### General rule

```
siblings run in parallel by default
add a semaphore cell when one must wait for another
add a coordinator BB only when logic is too complex for one cell
let the parent coordinate when the task layer needs to know anyway
```

---

## 19. Interpolation — Least Path, Least Energy, Least Pain

Not just coordination. Optimisation across the whole tree simultaneously.

```
least path    =  shortest route to goal
least energy  =  minimum motor effort
least pain    =  avoid joint limits, over-grip, instability
```

Biology calls this **motor synergy**. The brain does not command each muscle individually. It finds the lowest cost solution across all muscles at once.

### How it fits the BB hive

Each BB node currently does its own job in isolation. To add least-cost you need something that looks across the whole tree and asks:

```
of all the ways to achieve this task
which one costs least?
```

That is a different kind of computation. Not sequential state transitions. It is minimisation.

### Solution 1 — Cost tape cells

Each node writes a cost alongside its status:

```
tape: [GRIP_STATUS] [GRIP_COST] [ARM_STATUS] [ARM_COST]
                        │                        │
                        └──────────┬─────────────┘
                              [COST BB]
                            sums and compares
                            picks lowest path
```

Still pure BB. Cost is just another 0/1 cell. Cheap = 0, expensive = 1.

### Solution 2 — Gradient on the tape

Each cell holds a value that gets written and overwritten as nodes negotiate:

```
high cost  →  node writes 1
low cost   →  node writes 0
neighbour reads and adjusts its own behaviour
```

Like water finding a path downhill. No central controller. Cost flows through the tape. Close to how **ant colony optimisation** works.

### Solution 3 — Dedicated COST BB above the tree

```
            [ COST BB ]
           /     |     \
      [DRIVE] [ARM] [GRIPPER]
```

Reads all status cells. Computes which combination of states costs least. Writes permission signals down.

Not a coordinator. Not a parent. A **referee** that only speaks when the tree would waste energy.

### The biological parallel

```
cerebellum   →  COST BB      (smooth, efficient motion)
motor cortex →  parent BB    (what to do)
spinal cord  →  sibling BBs  (how to do it)
muscles      →  actuator BBs (raw movement)
```

The hive is already shaped like a nervous system. Adding least-cost is adding the cerebellum layer.

### Honest take

Start with semaphores and parent coordination. Get it working. Then measure actual energy use and pain points. Add cost cells only where the real robot actually wastes energy.

Do not design the optimiser before you know what needs optimising.

---

## 20. Tape vs Stack — Full Clarification

These are fundamentally different memory models.

```
STACK                        TAPE
─────────────────────────    ─────────────────────────────
last in first out            random access via head movement
only see the top             see any cell by moving L or R
data consumed when popped    data persists after reading
grows one direction          infinite both directions
pile of plates               shared whiteboard
```

For the BB hive the tape matters because:

- SENSOR value is still there after FEEL BB reads it
- GROUP BB can walk left and re-read any earlier cell
- Nothing is destroyed by reading
- Multiple machines can read the same cell independently

A stack would break the hive — reading a status would consume it and the next machine would see nothing.

---

## 21. HALT — What It Means Across the Whole System

HALT in a single BB machine means that machine stops stepping. But in a hive of machines the meaning depends on context.

### Local HALT

Only that branch stops. The rest of the hive keeps running.

```
GRIPPER GROUP BB → HALT (grip locked)
ARM BB           → still running (lifting)
DRIVE BB         → still running (holding position)
```

This is the default. HALT means task complete for that node, not system shutdown.

### Propagating HALT

A HALT writes a signal to the tape. The parent reads it and decides what to do next.

```
GRIPPER → HALT → writes 1 to GRIP_DONE cell
ARM GROUP BB reads GRIP_DONE
  → 0 = wait
  → 1 = proceed to lift
```

HALT becomes a synchronisation signal, not a stop signal.

### Emergency HALT

A fault condition that must stop everything. Written to a dedicated ESTOP cell.

The ESTOP cell is a second layer, not the only stop. A hardware E-stop in the motor supply line must cut motor power independently of the logic (section 25, Stage 0).

```
tape: ... [ESTOP] ...

any BB can write 1 to ESTOP
TASK BB watches ESTOP
  → 1 = write 0 to all CMD cells
  → all branches see no command
  → all motors stop
```

### Reset and boot defaults

A reset clears the tape to zero. Zero in ESTOP means "no stop", so ESTOP cannot be the boot safety.

```
rule         motors are disabled at boot until a boot check has run and passed
watchdog     enable the ATmega328P watchdog so a hang stops motor drive
```

### Summary

```
local HALT      =  branch done, others continue
propagating HALT =  semaphore, triggers next step
emergency HALT  =  ESTOP cell, whole system stops
```

Never assume HALT means global shutdown unless you explicitly wire it that way.

---

## 22. Fault Propagation Up the Tree

Faults detected at the bottom of the tree need to reach the top. The hive handles this via dedicated fault cells on the tape.

### Fault cell per branch

```
tape: [GRIP_FAULT] [ARM_FAULT] [DRIVE_FAULT] [ESTOP]
```

Each branch group BB writes to its fault cell when something goes wrong.

### Propagation rules

```
FEEL BB detects slip → writes 1 to GRIP_FAULT
GRIPPER GROUP BB reads GRIP_FAULT
  → retries internally
  → if retries exhausted → writes fault code to ARM_FAULT propagation cell
ARM GROUP BB reads
  → decides: abort, reposition, or escalate
  → if escalating → writes 1 to TASK_FAULT
TASK BB reads TASK_FAULT
  → decides: retry whole task, go to safe position, ESTOP
```

### Fault codes on tape

Each fault cell can carry more than 0/1 if you allow multi-cell encoding:

```
two cells per fault:
[FAULT_FLAG] [FAULT_TYPE]

0  0  = no fault
1  0  = recoverable (retry)
1  1  = unrecoverable (abort)
```

Still binary. Two cells instead of one. The parent reads both.

### Safe state

Every branch should have a defined safe state it returns to on fault:

```
GRIPPER   →  open fully
ARM       →  retract to home position
DRIVE     →  stop
```

The TASK BB knows these safe states and writes them as commands when a fault propagates to the top.

---

## 23. Full Tape Layout — Whole Robot

```
pos   cell              written by        read by
────  ────────────────  ────────────────  ──────────────────
0     SENSOR_GRIP       hardware          FEEL BB
1     FEEL_OUT          FEEL BB           STRENGTH BB
2     STRENGTH_OUT      STRENGTH BB       GRIPPER GROUP BB
3     OPENCLOSE_OUT     OPENCLOSE BB      GRIPPER GROUP BB
4     GRIP_CMD          GRIPPER GROUP BB  OPENCLOSE BB
5     GRIP_STATUS       GRIPPER GROUP BB  ARM GROUP BB, TASK BB
6     GRIP_FAULT        GRIPPER GROUP BB  ARM GROUP BB
7     GRIP_DONE         GRIPPER GROUP BB  ARM GROUP BB

8     SENSOR_J1         hardware          JOINT1 BB
9     SENSOR_J2         hardware          JOINT2 BB
10    SENSOR_J3         hardware          JOINT3 BB
11    J1_OUT            JOINT1 BB         ARM GROUP BB
12    J2_OUT            JOINT2 BB         ARM GROUP BB
13    J3_OUT            JOINT3 BB         ARM GROUP BB
14    ARM_CMD           ARM GROUP BB      JOINT1/2/3 BB
15    ARM_STATUS        ARM GROUP BB      TASK BB
16    ARM_FAULT         ARM GROUP BB      TASK BB
17    ARM_DONE          ARM GROUP BB      TASK BB

18    SENSOR_LEFT       hardware          LEFT MOTOR BB
19    SENSOR_RIGHT      hardware          RIGHT MOTOR BB
20    LEFT_OUT          LEFT MOTOR BB     DRIVE GROUP BB
21    RIGHT_OUT         RIGHT MOTOR BB    DRIVE GROUP BB
22    DRIVE_CMD         DRIVE GROUP BB    LEFT/RIGHT MOTOR BB
23    DRIVE_STATUS      DRIVE GROUP BB    TASK BB
24    DRIVE_FAULT       DRIVE GROUP BB    TASK BB

25    GRIP_LOCKED       GRIPPER GROUP BB  ARM GROUP BB (semaphore)
26    ARM_TUCKED        ARM GROUP BB      DRIVE GROUP BB (semaphore)
27    ESTOP             any BB            TASK BB (emergency)

28    TASK_CMD          TASK BB           ARM/DRIVE/GRIPPER GROUP BB
29    TASK_STATUS       TASK BB           external / operator
```

Semaphore cells at positions 25-27 handle all sideways coordination between branches.

---

## 24. How It All Comes Together — Building and Expanding Real Robots

### Start at the bottom, prove upward

Never start at the task layer. Start at the metal.

```
1. wire one actuator
2. write one BB node for it
3. prove it works in isolation
4. add the next actuator
5. repeat
```

Each node is testable alone before anything above it exists.

### Build order for this robot

```
phase 1 — actuators
  ├── left motor BB working
  ├── right motor BB working
  ├── joint 1 BB working
  ├── joint 2 BB working
  ├── joint 3 BB working
  └── gripper screw BB working

phase 2 — sensors
  ├── schmitt trigger on grip current
  ├── joint limit switches
  └── wheel encoders

phase 3 — group BBs
  ├── gripper group BB
  ├── arm group BB
  └── drive group BB

phase 4 — semaphores
  ├── GRIP_LOCKED cell wired
  └── ARM_TUCKED cell wired

phase 5 — task BB
  └── only wire this when everything below is solid
```

### Expanding a robot you built

Adding a new sensor:

```
1. add a tape cell for it
2. write one BB node that reads it
3. wire its output cell to whoever needs it
4. nothing else changes
```

Adding a new actuator:

```
1. add a tape cell for its command
2. add a tape cell for its status
3. write one BB node
4. connect to nearest group BB
5. group BB gets one new transition
```

The rest of the hive does not know or care.

### Expanding a robot you did not build

Most existing robots have monolithic firmware. To BB-ify it:

```
1. identify the actuators and sensors
2. map current behaviour as states (reverse engineer)
3. pull out one subsystem at a time
4. replace it with a BB node
5. leave everything else alone until proven
```

Do not rewrite everything at once. Peel one layer at a time.

### Physical wiring mirrors the tape

On a Nano:

```
tape cells      →  byte array in RAM
BB nodes        →  functions called in sequence
clock           →  main loop tick or interrupt
```

On discrete hardware:

```
tape cells      →  flip-flops
BB nodes        →  schmitt + 555 + SR latch
clock           →  555 astable master clock
```

### The expansion rule

```
never modify an existing node to add new behaviour
instead add a new node and wire it in
```

Existing behaviour cannot break because existing nodes did not change.

### Debugging in the field

Because every node reads and writes one tape cell:

```
attach oscilloscope or logic analyser to tape cells
watch the 0/1 transitions
fault is always visible at the exact cell where it stops propagating
```

No guessing. No printf debugging. The tape tells you exactly where the chain broke.

### Is this a good way to experiment?

Yes, for some things. Less so for others.

**Good for:**

```
isolation     each node is small enough to test alone on the bench
visibility    every signal is a named cell you can log or probe
discrete      grip locked or not, fault or not, which sequence step is active
learning      designing the machines forces explicit states and transitions
```

**Less good for:**

```
continuous control   holding an angle, balancing, regulating motor speed
                     needs a feedback controller (PID). 0/1 cannot express
                     that well. Rate coding is a workaround.
the BB framing       BB is a theory result about maximum machine behaviour.
                     What is useful here is small states, explicit
                     transitions, and a shared tape. Ordinary state machines
                     and a blackboard give you the same thing.
unproven claims      "no firmware bugs" for discrete circuits and the Nano
                     node budget are plausible estimates, not measurements.
```

**Safety gap:** the emergency stop in §21 is a software ESTOP cell. It must not be the only stop. Add:

```
hardware E-stop      cuts motor power independently of the logic
current limit        on the motor driver, so a logic bug cannot burn out
                     the gripper or drive the arm into something
```

**Low-risk way to try it:**

```
1. one motor, one current sensor, one current limit, on the bench
2. simulate FEEL, STRENGTH and GROUP tables in Python before wiring
3. run on hardware and log tape cells over time
4. add the arm only after step 3 is solid
5. add PID for any joint that must hold a position
```

Treat the tape design as the supervisor layer above proper motor control, not a replacement for it.

### One line summary

```
build bottom up
prove each node alone
wire upward only when lower layer is solid
expand by adding nodes never modifying existing ones
the tape is your bus, your debugger, and your documentation
```

---

## 25. Build Plan — Arm and Gripper in Stages

Target board: **Arduino Nano** (ATmega328P, 16 MHz, 2 KB RAM, 32 KB flash, 10-bit ADC so `analogRead` returns 0–1023, 5 V logic).

Rules for every stage:

```
1. do not start stage N+1 until stage N passes its verification
2. every stage has a fault path you have deliberately triggered
3. the E-stop is tested before any motor is energised under logic control
4. record measurements (Serial log or notes) so thresholds come from data
```

### Stage 0 — Safety before power

```
bench supply       current-limited, limit set below the motor's stall current
                   (check the motor datasheet; do not guess)
E-stop             hardware, in the motor supply line, cuts motor power
                   while the Nano stays powered
grounds            one common ground between Nano and motor driver
supply             motor supply separate from the Nano's 5 V
gripper            disconnected from the screw drive until Stage 1 passes
```

Verify:
- Press the E-stop with the Nano running. Measure the motor supply at the driver with a multimeter: it must read 0 V.
- Release it. The Nano must keep running with no motor power applied.

Troubleshoot:
- E-stop does not cut motor power → it is in the wrong branch of the wiring. Check it is in the motor supply, not the logic supply.

### Stage 1 — Gripper motor alone, no logic

Wire the H-bridge to two direction pins and one PWM enable pin. Pin numbers are placeholders; use your wiring.

```cpp
const uint8_t PIN_IN1 = 4;   // direction
const uint8_t PIN_IN2 = 5;   // direction
const uint8_t PIN_EN  = 6;   // PWM speed (must be a PWM-capable pin)

void setup() {
  pinMode(PIN_IN1, OUTPUT);
  pinMode(PIN_IN2, OUTPUT);
  pinMode(PIN_EN, OUTPUT);
}

void loop() {
  // forward 0.5 s, off 0.5 s, reverse 0.5 s, off 0.5 s
  digitalWrite(PIN_IN1, HIGH); digitalWrite(PIN_IN2, LOW);  analogWrite(PIN_EN, 80);
  delay(500);
  analogWrite(PIN_EN, 0);
  delay(500);
  digitalWrite(PIN_IN1, LOW);  digitalWrite(PIN_IN2, HIGH); analogWrite(PIN_EN, 80);
  delay(500);
  analogWrite(PIN_EN, 0);
  delay(500);
}
```

Verify:
- Forward phase closes the gripper, reverse phase opens it. Label which is which now.
- The motor stops in each off phase.
- No heating at the driver or motor while idle.

Troubleshoot:
- Direction reversed → swap the motor wires, not the code, so the code stays readable.
- No motion → check the enable pin, the motor supply voltage at the driver, and the common ground.
- Buzzes without turning → duty cycle too low for the load. Raise it briefly on the bench, under the current limit.

### Stage 2 — Current sensing and threshold calibration

Add a current sensor on the motor path: a shunt with an amplifier into an analog pin, or an I2C current monitor module. The sketch below reads an analog pin; if you use an I2C monitor, log its milliamp value instead and calibrate in those units.

```cpp
const uint8_t PIN_CURRENT = A0;

void setup() {
  Serial.begin(115200);
}

void loop() {
  // Log raw ADC values. Run this in three conditions and note each one:
  //   idle (motor off), gripping an object, stalled (jaws blocked, motor on)
  Serial.println(analogRead(PIN_CURRENT));
  delay(50);
}
```

Verify:
- Idle, grip, and stall readings are each repeatable over at least 20 cycles.
- The three conditions are clearly separated. Set `low` and `high` for the Schmitt trigger and `GRIP_THRESHOLD` and `STALL_THRESHOLD` from the logged data, with margin on each side.

Troubleshoot:
- Readings jump around → look for PWM noise on the sensor line. Check the ground path and keep sensor wiring away from the motor leads.
- Reading does not change with load → check the sensor's gain and wiring. Add a known load and check the output moves.
- Idle reading drifts with time → let the sensor warm up and average several samples.

### Stage 3 — FEEL node (the tested core)

Use the core below. It is the code that was compiled and tested on the PC and compiled for the Nano build.

```cpp
// bb_core.h
#pragma once
#include <stdint.h>

// Tape cell positions, matching the full layout in section 23.
enum : uint8_t {
  SENSOR_GRIP   = 0,
  FEEL_OUT      = 1,
  STRENGTH_OUT  = 2,
  OPENCLOSE_OUT = 3,
  GRIP_CMD      = 4,
  GRIP_STATUS   = 5,
  GRIP_FAULT    = 6,
  GRIP_DONE     = 7,
  GRIP_LOCKED   = 25,
  ESTOP         = 27,
  TAPE_SIZE     = 30
};

// One transition. The MOVE column from the doc is not stored: on a Nano the
// tape is a byte array, so there is no head to move.
struct Rule { uint8_t state, read, write, next; };
struct Machine { const Rule *rules; uint8_t count; uint8_t state; };

// Returns false if no rule matches. Treat false as a fault, never as "ok".
inline bool bb_step(Machine &m, uint8_t input, uint8_t &output) {
  for (uint8_t i = 0; i < m.count; i++) {
    const Rule &r = m.rules[i];
    if (r.state == m.state && r.read == input) {
      output = r.write;
      m.state = r.next;
      return true;
    }
  }
  return false;
}

// Hysteresis on a raw reading. Changes only when the reading crosses the far
// threshold. Requires low < high.
inline uint8_t schmitt(uint16_t adc, uint8_t prev, uint16_t low, uint16_t high) {
  if (prev == 0 && adc > high) return 1;
  if (prev == 1 && adc < low) return 0;
  return prev;
}
```

FEEL node sketch (uses the core above):

```cpp
#include "bb_core.h"

// FEEL BB, section 9. States: A = 0, B = 1.
static const Rule FEEL_RULES[] = {
  {0, 0, 0, 0}, {0, 1, 1, 1}, {1, 0, 0, 0}, {1, 1, 1, 0},
};

Machine feel = {FEEL_RULES, 4, 0};
uint8_t tape[TAPE_SIZE];

void setup() { Serial.begin(115200); }

void loop() {
  uint8_t out;
  // Analog front end writes the hardware cell (the Schmitt output).
  tape[SENSOR_GRIP] = schmitt(analogRead(A0), tape[SENSOR_GRIP], 300, 500);
  // FEEL BB reads SENSOR_GRIP and writes FEEL_OUT.
  if (bb_step(feel, tape[SENSOR_GRIP], out)) {
    tape[FEEL_OUT] = out;
  } else {
    tape[GRIP_FAULT] = 1;
  }
  delay(10);
}
```

The `300` and `500` thresholds are placeholders. Replace them with the values from Stage 2.

Verify on the PC (passes now):
- The host test suite covers: every (state, input) pair has a rule; an undefined state reports failure and leaves the output unchanged; the Schmitt output does not chatter when the reading wobbles around one threshold.
- On the bench, turn a potentiometer on A0 slowly across both thresholds. `FEEL_OUT` must change only at the far threshold, not on the way back.

Troubleshoot:
- Output flickers → thresholds are too close together. Widen the gap between `low` and `high`.
- Output never changes → check `analogRead` values over Serial first. Then check the thresholds are inside the real range.

**Finding from checking the doc:** in the section 9 FEEL table, every rule writes the same value it reads. The output is a copy of the input. The anomaly information is held in the state, and the filtering comes from the Schmitt thresholds, not from the table. The design does not do more than this, and it should be described that way.

### Stage 4 — STRENGTH node

Decide, before wiring, what `STRENGTH_OUT` means on the tape. The doc does not say. A single bit cannot carry a rate, so define it explicitly:

```
STRENGTH_OUT = 1   pulse train on, rate set by the motor layer
STRENGTH_OUT = 0   pulse train off
```

Use the table from section 9 with the same machine code as Stage 3. Only the rule table and cells change.

Verify:
- The host test `test_strength_sequence` passes: feel 1, 1, 0 gives outputs 1, 0, 0.
- On a logic analyser, `STRENGTH_OUT` changes state exactly where the table predicts, with the same input sequence.

Troubleshoot:
- Output lags the feel input by one step → this is the table, not a bug. Check the input is the cell you intended, `FEEL_OUT`.

### Stage 5 — OPENCLOSE node and timeout fault (F1)

Add two limit switches (open limit, close limit) and a timer. Use the close timeout as fault F1 from section 6.

Verify, with each case deliberately triggered:
- Close limit reached → motor stops, `OPENCLOSE_OUT` shows the limit.
- Close limit never reached (unplug it) → the timeout fires, `GRIP_FAULT` is set, motor stops in bounded time.
- Open limit reached → motor stops.

Troubleshoot:
- Motor keeps running past the limit → the limit switch is wired normally-closed when you expected normally-open, or the pin mode is wrong. Check the switch state with Serial before trusting the motor logic.
- Timeout fires too early → the timeout is shorter than the real closing time. Measure the time from Stage 2 logs and add margin.

### Stage 6 — GRIP group and HALT (grip locked)

Implement the GROUP BB from section 9. The doc says HALT means "lock grip", but code has to define what happens. Use this:

```
HALT reached   →  GRIP_LOCKED = 1, GRIP_DONE = 1, motor off
                  (the screw drive is backdrive-proof, so the grip holds with the motor off)
```

Verify:
- After HALT, the motor is off and the current reading stays steady for at least 30 seconds.
- Press the E-stop during a grip. Motor power must cut, and the tape must not show a grip held that is not there. Check `GRIP_LOCKED` after the stop.

Troubleshoot:
- Grip releases after the motor stops → the screw drive is not holding as expected. Check that the screw is not back-driven by the jaws, and check the mechanical design, not the logic.

### Stage 7 — First arm joint, alone

The doc is inconsistent on the joints. Section 4 says each joint has `+`, `-`, `hold`, which describes a motor. Section 17 says `servo×3`. Choose one before you build:
- **Servos** have built-in position feedback, so the first joint needs no encoder. Start here.
- **DC motors** need a position sensor and a PID loop (section 24 "Is this a good way to experiment?"). Add these after servos work, not before.

Section 4 and section 17 now both say servos.

Verify:
- Command the joint to three positions in sequence. The measured position matches the command within your tolerance.
- The hardware limit switch stops the joint before the software limit is reached. Test by commanding past the limit on the bench, with the arm unloaded.

Troubleshoot:
- Joint jitters at rest → check the power supply under load. Servos can brown out the Nano if they share its 5 V supply.
- Joint overshoots → reduce the speed or add a deceleration step in software.

### Stage 8 — Joints 2 and 3, arm group, and the semaphore

Add one joint at a time, each with its own test from Stage 7. Then add the ARM group.

Coordination rule from section 18: the arm must not lift until the grip is locked.

```
ARM group reads GRIP_LOCKED
  0 → hold, do not move
  1 → proceed
```

Verify:
- With `GRIP_LOCKED` forced to 0 in software, the arm does not move, for at least 10 seconds.
- With it set to 1, the arm moves.
- Set `GRIP_LOCKED` to 0 mid-move. Confirm the arm stops at the next safe point, not mid-motion, and check the behaviour is what you designed.

Troubleshoot:
- Arm moves while grip is not locked → the semaphore cell is not being read. Confirm the read happens every loop, before the move command.

### Stage 9 — Integration and faults

Test each fault path with the whole system running:

```
E-stop pressed during: grip close, grip hold, arm lift, arm idle
sensor unplugged: current sensor, each limit switch, each joint's feedback
stall: block the jaws during close, block the arm during lift
power: remove and restore Nano power while the motor is running
```

Reset behaviour (important):
- A reset clears the tape to zero. Zero in `ESTOP` means "no stop", which is the unsafe default.
- Make the boot default the safe one: all motor outputs off until a boot check passes.
- Enable the ATmega328P's watchdog timer so a hang stops the motors.

Verify:
- Each row above has a recorded result: expected outcome, observed outcome, and pass or fail.
- After power is restored, the motors do not move until the boot check has run.

### Known gaps: resolved

```
1. STRENGTH_OUT meaning          defined in section 9 (1 = pulse on, 0 = off)
2. FEEL and GROUP copy input     noted in section 9; behaviour lives in the state
3. joints motor vs servo         section 4 now says servo (section 25, Stage 7)
4. HALT had no effect in code    defined in section 9 GROUP BB and Stage 6
5. software-only E-stop          hardware E-stop required, section 21 and Stage 0
6. reset defaults not stated     section 21 "Reset and boot defaults", Stage 9
```

### Troubleshooting summary

```
symptom                          first check
motor does not move              enable pin, motor supply at driver, common ground
motor runs but wrong direction   swap motor wires
readings noisy                   PWM noise, ground path, sensor wire routing
output flickers                  threshold gap too small
arm moves with grip unlocked     semaphore read order in the loop
Nano resets under load           separate motor and logic supplies, check brownout
```

### Verification checklist

```
[ ] Stage 0: E-stop cuts motor supply with the Nano running
[ ] Stage 1: forward closes, reverse opens, off holds
[ ] Stage 2: idle, grip, and stall readings separated and repeatable
[ ] Stage 3: host tests pass; Schmitt output moves only at far threshold on hardware
[ ] Stage 4: STRENGTH_OUT toggles as the table predicts on a logic analyser
[ ] Stage 5: limit stop and F1 timeout both work
[ ] Stage 6: HALT locks grip, motor off, current steady
[ ] Stage 7: joint hits three commanded positions within tolerance
[ ] Stage 8: arm holds while GRIP_LOCKED = 0
[ ] Stage 9: every fault row recorded; boot defaults are safe
```

---

*Session: 2026-10-28*
