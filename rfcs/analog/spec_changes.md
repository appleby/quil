Add section 6 to spec:

## 6. Pulse Level Control

**Frames**

```
Frame :: String
```

A frame encapsulates any rotating frame relative to which control/readout
waveforms may be defined. Frames are defined by simple strings.

There are no built-in frames, they are dependent on the architecture.

Frames (and associated sample rates) need to be provided to the user prior to
construction of a program. Rigetti has a set of canonical frames (some examples
are below) but this is subject to change.

Examples:
```
"xy"  # eg. for the drive line
"ff"  # eg. for a generic flux line
"cz"  # eg. for a flux pulse for enacting CZ gate
"iswap"
"ro"  # eg. for the readout pulse
"out" # eg. for the capture line
```

**Waveforms**

```
Waveform :: Name
Waveform :: flat ( duration: Expression, iq: Expression )
Waveform :: gaussian ( duration: Expression, fwhm: Expression, t0: Expression )
Waveform :: draggaussian ( duration: Expression, fwhm: Expression, t0: Expression,
    anh: Expression, alpha: Expression )
Waveform :: erfsquare ( duration: Expression, risetime: Expression,
    padleft: Expression, padright: Expression )
```

Waveforms are referenced either by name or by a built-in waveform generator.

The built-in waveform generators are:
- `flat(duration, iq)` creates a flat waveform where:
    - `duration` is a rational number representing the duration of the
      waveform in seconds
    - `iq` is a complex number representing the IQ value to play for the
      duration of the waveform
- `gaussian(duration, fwhm, t0)` creates a Gaussian waveform where:
    - `duration` is a rational number representing the duration of the
      waveform in seconds
    - `fwhm` is a rational number representing the full-width-half-max of
      the waveform in seconds
    - `t0` is a rational number representing the center time coordinate of
      the waveform in seconds
- `draggaussian(duration, fwhm, t0, anh, alpha)` creates a DRAG gaussian pulse where:
    - `duration` is a rational number representing the duration of the
      waveform in seconds
    - `fwhm` is a rational number representing the full-width-half-max of
      the waveform in seconds
    - `t0` is a rational number representing the center time coordinate of
      the waveform in seconds
    - `anh` is a rational number representing the anharmonicity of the qubit in
      Hertz
    - `alpha` is a rational number for the dimensionless drag parameter
- `erfsquare(duration, risetime, padleft, padright)` creates a pulse with a flat
    top and edges that are error functions (erfs) where:
    - `duration` is a rational number representing the duration of the
      waveform in seconds
    - `risetime` is a rational number representing the rise and fall sections of
      the pulse in seconds
    - `padleft` is a rational number representing the amount of zero-amplitude
      padding to add to the left of the pulse
    - `padright` is a rational number representing the amount of zero-amplitude
      padding to add to the right of the pulse

**Defining new waveforms**

```
WaveformDefinition :: DEFWAVEFORM Name ( Parameter+ ) : MatrixRow
MatrixRow :: Indent (Expression ,)+
```

New waveforms may be defined by listing out all the IQ values as complex
numbers, separated by commas. Waveform definitions may also be parameterized,
although note that Quil has no support for vector level operations.

Example:
```
DEFWAVEFORM my_custom_waveform:
    1+2i, 3+4i, 5+6i

DEFWAVEFORM my_custom_paramterized_waveform(%a)
    (1+2i)*%a, (3+4i)*%a, (5+6i)*%a
```

**Pulses**

```
Pulse :: PULSE Qubit+ Frame Waveform
```

Pulses can be played on the frame of a particular qubit by listing the qubit,
frame name, waveform name (or generator).

Examples:
```
# Simple pulse with previously defined waveform
PULSE 0 "xy" my_custom_waveform

# Pulse with previously defined parameterized waveform
PULSE 0 "xy" my_custom_parameterized_waveform(0.5)

# Pulse with built-in waveform generator
PULSE 0 "xy" flat(duration: 1e-6, iq: 2+3i)

# Pulse on a flux line
PULSE 0 1 "cz" flat(duration: 1e-6, iq: 2+3i)
```

**Frequency**

```
SetFrequency :: SET-FREQUENCY Qubit+ Frame Float
```

Each frame has a frequency which is tracked throughout the program. Initially
the frequency starts out as not defined.

Set instructions are local to the surrounding scope.

```
SET-FREQUENCY 0 "xy" 5.4e9
SET-FREQUENCY 0 "ro" 6.1e9
```

**Phase**

```
SetPhase :: SET-PHASE Qubit+ Frame Float
ShiftPhase :: SHIFT-PHASE Qubit+ Frame Expression
SwapPhases :: SWAP-PHASES Qubit+ Frame Qubit+ Frame
```

Each frame has a phase which is tracked throughout the program. Initially the
phase starts out as 0. It may be set or shifted up and down, as well as swapped
with other frames.

The phase must be a rational real number. There is also support for
shifted the phase based on some expression, as long as that expression returns
a real number.

Set instructions are local to the surrounding scope, however shifting and
swapping have global effect to the frame across the entire program.

Example:
```
SET-PHASE 0 "xy" pi/2

SHIFT-PHASE 0 "xy" -pi
SHIFT-PHASE 0 "xy" %theta*2/pi

SWAP-PHASE 0 "xy" 1 "xy"
```

**Scale**

```
SetScale :: SET-SCALE Qubit+ Frame Float
```

Each frame has a scale which is tracked throughout the program. Initially the
scale starts out as 1.

Set instructions are local to the surrounding scope.

Example:
```
SET-SCALE 0 "xy" 0.75
```

**Capture**

```
Capture :: CAPTURE Qubit Frame Waveform MemoryReference
RawCapture :: RAW-CAPTURE Qubit Frame Expression MemoryReference
```

The capture instruction opens up the readout on a qubit and measures its state.
An integration waveform will be applied to the raw IQ points and the result is
placed in classical memory.

The waveform will define the length of the capture. The memory reference must be
able to store a complex number.

In the case of a raw capture the waveform is replaced with a rational number
representing the duration of the capture.

Example:
```
# Simple capture of an IQ point
DECLARE iq REAL[2]
CAPTURE 0 "out" flat(1e-6, 2+3i) iq

# Raw capture
DECLARE iqs REAL[400] # length needs to be determined based on the sample rate
CAPTURE 0 "out" 200e-6 iqs
```

**Defining Calibrations**

```
CalibrationDefinition :: DEFCAL Name ( Parameter+ ) Qubit+ : Instruction+
MeasureCalibrationDefinition :: DEFCAL Name Qubit Parameter : Instruction+
```

Calibrations for high-level gates can be defined by mapping a combination of
(gate name, parameters, qubits) to a sequence of analog control instructions.

Calibrations with the same gate name as a built-in gate definition or custom
gate definition are assumed to be the same.

Multiple calibration definitions can be defined for different parameter and
qubit values. When a gate is translated into control instructions the
calibration definitions are enumerated in reverse order of definition and the
first match will be taken.

For example, given the following list of calibration definitions in this order:
1. `DEFCAL RX(%theta) %qubit:`
2. `DEFCAL RX(%theta) 0:`
3. `DEFCAL RX(pi/2) 0:`
The instruction `RX(pi/2) 0` would match (3), the instruction `RX(pi) 0` would
match (2), and the instruction `RX(pi/2) 1` would match (1).

The same system applies for `MEASURE` although `MEASURE` cannot be
parameterized, it takes only a single qubit as input, and it has an additional
parameter for the memory reference in which to read out the result.

Examples:
```
# Simple non-parameterized gate on qubit 0
DEFCAL X 0:
    PULSE 0 "xy" gaussian(duration: 1, fwhm: 2, t0: 3)

# Parameterized gate on qubit 0
DEFCAL RX(%theta) 0:
    PULSE 0 "xy" flat(duration: 1e-6, iq: 2+3i)*%theta/(2*pi)

# Applying RZ to any qubit
DEFCAL RZ(%theta) %qubit:
    SHIFT-PHASE %qubit "xy" %theta

# Measurement and classification
DEFCAL MEASURE 0 %dest:
    DECLARE iq REAL[2]
    CAPTURE 0 "out" flat(1e-6, 2+3i) iq
    LT %dest iq[0] 0.5 # thresholding
```

**Timing Control**

```
Delay :: DELAY Qubit Expression
Fence :: FENCE Qubit+
```

Delay allows for the insertion of a gap within a list of pulses or gates with
a specified duration in seconds.

Fence ensures that all operations on the specified qubits that proceed the
fence statement happen after the end of the right-most operation of that set
of qubits.

Examples:
```
X 0
FENCE 0 1
X 1 # This X gate will be applied to qubit 1 AFTER the X gate on qubit 0

# Simple T1 experiment
X 0
DELAY 0 100e-6
MEASURE 0 ro[0]
```
