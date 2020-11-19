# ROMEO Unwrapping
Unwrapping of 3D and 4D datasets

### Download Executables
https://github.com/korbinian90/ROMEO/releases  
Executables are available for windows, linux and macOS (thanks to Joseph Woods for compiling the macOS version)

### Publication
Dymerska, B., Eckstein, K., Bachrata, B., Siow, B., Trattnig, S., Shmueli, K., Robinson, S.D., 2020. Phase Unwrapping with a Rapid Opensource Minimum Spanning TreE AlgOrithm (ROMEO). Magnetic Resonance in Medicine. https://doi.org/10.1002/mrm.28563

### Repositories
The sourcecode is available under [ROMEO.jl](https://github.com/korbinian90/ROMEO.jl).  
The binaries are a standalone compiled version of [RomeoApp.jl](https://github.com/korbinian90/RomeoApp.jl). The compilation scripts are in [CompileMRI.jl](https://github.com/korbinian90/CompileMRI.jl).

## Getting Started
### Prerequisites
Phase (and optionally Magnitude) images in NIfTI fileformat.  
For multi-echo/multi-timepoint data, 4D-NIfTI files are used with echoes/timepoints in the 4th dimension.
Individual 3D files can be merged into 4D files using [fslmerge](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/Fslutils).

### [Compiled Version](https://github.com/korbinian90/ROMEO/releases  )
ROMEO is a command line application.

Example usage for multiple time points with identical echo time (fMRI):  
`$ romeo ph.nii -m mag.ii -k nomask -o outputdir`

Example usage for a 3-echo Scan with TE = [3,6,9] ms  
`$ romeo ph.nii -m mag.ii -k nomask -t [3,6,9] -o outputdir`

### Help on arguments:
```
$ romeo
usage: <PROGRAM> [-m MAGNITUDE] [-o OUTPUT] [-t ECHO-TIMES] [-k MASK]
                 [-e UNWRAP-ECHOES] [-w WEIGHTS] [-B]
                 [--phase-offset-correction] [-i]
                 [--template TEMPLATE] [-N] [--no-rescale]
                 [--threshold THRESHOLD] [-v] [-g] [-q] [-Q]
                 [-s MAX-SEEDS] [--merge-regions] [--correct-regions]
                 [--wrap-addition WRAP-ADDITION]
                 [--temporal-uncertain-unwrapping] [--version] [-h]
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
                        unwrapping specified in array or range syntax
                        (eg. "[1.5,3.0]" or "3.5:3.5:14"). (default is
                        ones(<nr_of_time_points>) for multiple volumes
                        with the same time) Warning: No spaces
                        allowed!! ("[1, 2, 3]" is invalid!)
  -k, --mask MASK       nomask | robustmask | <mask_file> (default:
                        "robustmask")
  -e, --unwrap-echoes UNWRAP-ECHOES
                        Load only the specified echoes from disk
                        (default: ":")
  -w, --weights WEIGHTS
                        romeo | romeo2 | romeo3 | romeo4 | bestpath |
                        <4d-weights-file> | <flags>. <flags> are four
                        bits to activate individual weights (eg.
                        "1010"). The weights are (1)phasecoherence
                        (2)phasegradientcoherence (3)phaselinearity
                        (4)magcoherence (default: "romeo")
  -B, --compute-B0      Calculate combined B0 map in [Hz]. Phase
                        offset                correction might be
                        necessary if not coil-combined with
                        MCPC3Ds/ASPIRE.
  --phase-offset-correction
                        Applies the MCPC3Ds method to perform phase
                        offset determination and removal (for
                        multi-echo).
  -i, --individual-unwrapping
                        Unwraps the echoes individually (not
                        temporal). This might be necessary if there is
                        large movement (timeseries) or
                        phase-offset-correction is not applicable.
  --template TEMPLATE   Template echo that is spatially unwrapped and
                        used for temporal unwrapping (type: Int64,
                        default: 2)
  -N, --no-mmap         Deactivate memory mapping. Memory mapping
                        might cause problems on network storage
  --no-rescale          Deactivate rescaling of input images. By
                        default the input phase is rescaled to the
                        range [-π;π]. This option allows inputting
                        already unwrapped phase images without
                        wrapping them first.
  --threshold THRESHOLD
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
  -s, --max-seeds MAX-SEEDS
                        EXPERIMENTAL! Sets the maximum number of seeds
                        for unwrapping. Higher values allow more
                        seperated regions. (type: Int64, default: 1)
  --merge-regions       EXPERIMENTAL! Spatially merges neighboring
                        regions after unwrapping.
  --correct-regions     EXPERIMENTAL! Performed after merging. Brings
                        the median of each region closest to 0 (mod
                        2π).
  --wrap-addition WRAP-ADDITION
                        [0;π] EXPERIMENTAL! Usually the true phase
                        difference of neighboring voxels cannot exceed
                        π to be able to unwrap them. This setting
                        increases the limit and uses 'linear
                        unwrapping' of 3 voxels in a line. Neighbors
                        can have (π + wrap-addition) phase difference.
                        (type: Float64, default: 0.0)
  --temporal-uncertain-unwrapping
                        EXPERIMENTAL! Uses spatial unwrapping on
                        voxels that have high uncertainty values after
                        temporal unwrapping.
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
