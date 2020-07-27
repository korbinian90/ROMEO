# ROMEO Unwrapping
Unwrapping of 3D and 4D datasets.

Please cite [ROMEO publication] if you use it! The link will follow once it is published.

The sourcecode is available under [ROMEO.jl](https://github.com/korbinian90/ROMEO.jl).
The binaries are a standalone compiled version of [RomeoApp.jl](https://github.com/korbinian90/RomeoApp.jl).

## Getting Started
### Prerequisites
Magnitude and Phase images in NIfTI fileformat (4D images with echoes in the 4th dimension).

### Compiled Version
Compiled versions for windows and linux are attached to the release.
ROMEO is a command line application.

Example usage on linux:

`$ romeo ph.nii -m mag.ii -k nomask -o outputdir`

On windows (cmd or powershell):

`>romeo.exe ph.nii -m mag.ii -k nomask -o outputdir`

### Help on arguments:
```
$ romeo
usage: <PROGRAM> [-m MAGNITUDE] [-o OUTPUT] [-t ECHO-TIMES] [-k MASK]
                 [-i] [-e UNWRAP-ECHOES] [-w WEIGHTS] [-B] [-N]
                 [-T THRESHOLD] [-v] [-g] [-q] [-Q] [--version] [-h]
                 [phase]

positional arguments:
  phase                 The phase image used for unwrapping

optional arguments:
  -m, --magnitude MAGNITUDE
                        The magnitude image (better unwrapping if
                        specified)
  -o, --output OUTPUT   The output path or filename (default:
                        "unwrapped.nii")
  -t, --echo-times ECHO-TIMES
                        The relative echo times required for temporal
                        unwrapping (default is 1:n) specified in array
                        or range syntax (eg. "[1.5,3.0]" or
                        "3.5:3.5:14") or for multiple volumes with the
                        same time: "ones(<nr_of_time_points>)".
                        Warning: No spaces allowed!! ("[1, 2, 3]" is
                        invalid!)
  -k, --mask MASK       nomask | robustmask | <mask_file> (default:
                        "robustmask")
  -i, --individual-unwrapping
                        Unwraps the echoes individually (not
                        temporal). Temporal unwrapping only works when
                        phase offset is removed (ASPIRE)
  -e, --unwrap-echoes UNWRAP-ECHOES
                        Unwrap only the specified echoes (default:
                        ":")
  -w, --weights WEIGHTS
                        romeo | romeo2 | romeo3 | romeo4 | bestpath |
                        <4d-weights-file> | <flags>. <flags> are four
                        bits to activate individual weights (eg.
                        "1010"). The weights are (1)phasecoherence
                        (2)phasegradientcoherence (3)phaselinearity
                        (4)magcoherence (default: "romeo")
  -B, --compute-B0      EXPERIMENTAL! Calculate combined B0 map in
                        [rad/s]
  -N, --no-mmap         Deactivate memory mapping. Memory mapping
                        might cause problems on network storage
  -T, --threshold THRESHOLD
                        <maximum number of wraps>. Threshold the
                        unwrapped phase to the maximum number of wraps
                        and sets exceeding values to 0 (type: Float64,
                        default: Inf)
  -v, --verbose         verbose output messages
  -g, --correct-global  Phase is corrected to remove global n2π phase
                        offset. The median of phase values (inside
                        mask if given) is used to calculate the
                        correction term
  -q, --write-quality   Writes out the ROMEO quality map as a 3D image
                        with one value per voxel
  -Q, --write-quality-all
                        Writes out an individual quality map for each
                        of the ROMEO weights.
  --version             show version information and exit
  -h, --help            show this help message and exit

```

## Known issues
### v1.4
- single echo unwrapping with magnitude and default mask leads to ReadOnlyMemory error. Fixed in v1.4.1
- binary data type for mask not supported. Fixed in v2.0.1
### v2.0.1
- input scaling issue (with INT16). Fixed in v2.0.2

## Feedback
Feature requests and bug reports are welcome!
