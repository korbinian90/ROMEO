# ROMEO Unwrapping
Unwrapping of 3D and 4D datasets.
The sourcecode will be released after the publication of ROMEO.

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
                 [-T THRESHOLD] [-v] [-g] [-h] [phase]

positional arguments:
  phase                 The phase image used for unwrapping

optional arguments:
  -m, --magnitude MAGNITUDE
                        The magnitude image (better unwrapping if
                        specified)
  -o, --output OUTPUT   The output path and filename (default:
                        "unwrapped.nii")
  -t, --echo-times ECHO-TIMES
                        The relative echo times required for temporal
                        unwrapping (default is 1:n) specified in array
                        or range syntax (eg. [1.5,3.0] or 2:5)
                        Warning: No spaces allowed!! ([1, 2, 3] is
                        invalid!)
  -k, --mask MASK       nomask | robustmask | <mask_file> (default:
                        "robustmask")
  -i, --individual-unwrapping
                        Unwraps the echoes individually (not temporal)
                        Temporal unwrapping only works with ASPIRE
  -e, --unwrap-echoes UNWRAP-ECHOES
                        Unwrap only the specified echoes (default:
                        ":")
  -w, --weights WEIGHTS
                        romeo | romeo2 | romeo3 | bestpath |
                        <4d-weights-file>  (default: "romeo3")
  -B, --compute-B0      EXPERIMENTAL! Calculate combined B0 map in
                        [rad/s]
  -N, --no-mmap         Deactivate memory mapping. Memory mapping
                        might cause problems on network storage
  -T, --threshold THRESHOLD
                        <maximum number of wraps> Threshold the
                        unwrapped phase to the maximum number of wraps
                        Sets exceeding values to 0 (type: Float64,
                        default: Inf)
  -v, --verbose         verbose output messages
  -g, --correct-global  phase is corrected to remove global n2π phase
                        offset. The median of phase values (inside
                        mask if given) is used to calculate the
                        correction term
  -h, --help            show this help message and exit


```
