# expand_netlist.py

A plugin for multiple instance and bus naming for KiCad's Eeschema.

Use this script in Eeschema's BOM plugin interface to generate Pcbnew netlists with multiple instance and bus naming. Components, pins, and nets with "plural" names, as described below, will be replicated and reconnected appropriately. Hierarchical sheets are not supported yet.

## Installing:
Copy `expand_netlist.py` and `expand_bom.py` into the plugins folder of your KiCad installation. These scripts will work with the unmodified `kicad_netlist_reader.py` located there. You can also copy in the tweaked version of that file from this repository, which has a few minor improvements. The default location of the plugins folder on Windows is `C:\Program Files\KiCad\bin\scripting\plugins`.

To use the scripts in Eeschema, add them as BOM plugins. The BOM plugin interface is preferable to the netlist export interface because it displays the documentation and reports error messages.

## Name expansion rules:
Components references, pin names, and net names (labels) are expanded using the following operators:
  
#### Lists of names are separated by commas:
`A,B,C` -> `A`, `B`, `C`
- Whitespace is not removed, so be careful: `A, B` -> `A`, ` B`

#### Ranges are shorthand for lists of numbers or letters:
Both `first:last` and `first:last:step` are valid syntax.
- `first` and `last` must be non-negative integers or uppercase letters.
- `step` must be a positive integer.
- `first` and `last` will both be included in the range unless the `step` doesn't align.
- `last` may be less than `start`, in which case the range will count down. A negative `step` need not be specified.
- Valid letters are in `ABCDEFGHJKLMNPRTUVWY`. `IOQSXZ` are excluded, which is typical of BGA grid lettering. After 'Y' comes 'AA'.

Examples:
- `1:3` -> `1`, `2`, `3`
- `3:1` -> `3`, `2`, `1`
- `0:10:5` -> `0`,`5`,`10`
- `A:D` -> `A`, `B`, `C`, `D`
- `A:D:2` -> `A`, `C` (This is an example of `step` preventing `last` from being included.)

#### The vertical bar `|` acts on lists and/or ranges, pairing each combination together:
`A,B|1:3` -> `A1`, `A2`, `A3`, `B1`, `B2`, `B3`

#### Brackets `[]` replicate the preceeding and following text for each item within:
`A[1,2,3]B` -> `A1B`, `A2B`, `A3B`
- To keep the brackets, double them: `A[[1]]B` -> `A[1]B`

If multiple replications occur in the same name it behaves similarly to the vertical bar:
- `A[1,2].B[3,4]` -> `A1.B3`, `A1.B4`, `A2.B3`, `A2.B4`
  
#### A backtick `` ` `` indicates that all following text should be discarded:
``A[1:5]`1`` -> `A1`, `A2`, ...
- This is to keep KiCad's annotation checker happy on netlist export. That is, Eeschema will not recognize `A[1:5]` as a valid reference, which would prevent you from running this script, but it will accept ``A[1:5]`1``.

## Netlist connection rules:
When expanding nets, there are two valid cases for how they are connected:
1) If the net name or any of the nodes the net connects to are singular,
   then all of the nodes are shorted together.
   - E.g. Component `U[0:7]` pin `VCC` (`U[0:7].VCC`) is connected to `+3V3` by an unlabeled wire.
   Each of `U0.VCC` through `U7.VCC` will be shorted to `+3V3`.
2) If the net name and each node it connects to expand to name lists with the same
   cardinality, then a net is created connecting each corresponding node.
   - E.g. Component `U1.OUT[0:7]` is connected to `D[15:8].A` by an unlabeled wire.
   8 nets are created connecting `U1.OUT0` to `D15.A`, `U1.OUT1` to `D14.A`, ...

**Any other situations are treated as an error.**

## To Do:
- [x] BoM generation.
- [x] Deterministic tstamps so that renaming components doesn't destroy the PCB.
- [ ] Images of example schematics.
- [ ] Add syntax checking to the parser.
- [ ] Add error messages.
- [ ] Support hierarchical schematics.
- [ ] Convert to C.
- [ ] https://bugs.launchpad.net/kicad/+bug/1797038
