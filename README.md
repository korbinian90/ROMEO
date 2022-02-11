# ROMEO Unwrapping
Unwrapping of 3D and 4D datasets.
Coil combination of 5D datasets.

![ROMEO](https://user-images.githubusercontent.com/1307522/144416428-0c51a0e5-cb07-4b7d-8571-9fa2dc33f580.png)

### Download Executables for [Linux](https://github.com/korbinian90/ROMEO/releases/tag/v3.2.7), [Windows](https://github.com/korbinian90/ROMEO/releases/tag/v3.2.8) and [macOS](https://github.com/korbinian90/ROMEO/releases/tag/v3.2.7)
https://github.com/korbinian90/ROMEO/releases

### Publication
**ROMEO**: Dymerska, B., Eckstein, K., Bachrata, B., Siow, B., Trattnig, S., Shmueli, K., Robinson, S.D., 2020. Phase Unwrapping with a Rapid Opensource Minimum Spanning TreE AlgOrithm (ROMEO). Magnetic Resonance in Medicine. https://doi.org/10.1002/mrm.28563

**MCPC-3D-S Coil Combination**:
Eckstein, K., Dymerska, B., Bachrata, B., Bogner, W., Poljanc, K., Trattnig, S., Robinson, S.D., 2018. Computationally Efficient Combination of Multi-channel Phase Data From Multi-echo Acquisitions (ASPIRE). Magnetic Resonance in Medicine 79, 2996–3006. https://doi.org/10.1002/mrm.26963

### Repositories
The sourcecode is available under [ROMEO.jl](https://github.com/korbinian90/ROMEO.jl).  
The binaries are a standalone compiled version of [RomeoApp.jl](https://github.com/korbinian90/RomeoApp.jl). The compilation scripts are in [CompileMRI.jl](https://github.com/korbinian90/CompileMRI.jl).

## Getting Started
### Prerequisites
Phase (and optionally Magnitude) images in NIfTI fileformat.  
For multi-echo/multi-timepoint data, 4D-NIfTI files are used with echoes/timepoints in the 4th dimension.   
Individual 3D files can be merged into 4D files using [fslmerge](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/Fslutils).  
If 5D-NIfTI datasets with channels in the 5th dimension are given, coil combination can be performed as first step. 

### [Compiled Version](https://github.com/korbinian90/ROMEO/releases)
ROMEO is a command line application.

Example usage for single-echo or multiple time points with identical echo time (fMRI):  
`$ romeo ph.nii -m mag.ii -k nomask -o outputdir`

Example usage for a 3-echo Scan with TE = [3,6,9] ms:  
`$ romeo ph.nii -m mag.ii -k nomask -t [3,6,9] -o outputdir`

Note that echo times are required for unwrapping multi-echo data.

### Help on arguments:
```
$ romeo
usage: <PROGRAM> [-p PHASE] [-m MAGNITUDE] [-o OUTPUT]
                 [-t ECHO-TIMES [ECHO-TIMES...]] [-k MASK [MASK...]]
                 [-u] [-e UNWRAP-ECHOES [UNWRAP-ECHOES...]]
                 [-w WEIGHTS] [-B]
                 [--phase-offset-correction [PHASE-OFFSET-CORRECTION]]
                 [-i] [--template TEMPLATE] [-N] [--no-rescale]
                 [--threshold THRESHOLD] [-v] [-g] [-q] [-Q]
                 [-s MAX-SEEDS] [--merge-regions] [--correct-regions]
                 [--wrap-addition WRAP-ADDITION]
                 [--temporal-uncertain-unwrapping] [--version] [-h]

optional arguments:
  -p, --phase PHASE     The phase image that should be unwrapped
  -m, --magnitude MAGNITUDE
                        The magnitude image (better unwrapping if
                        specified)
  -o, --output OUTPUT   The output path or filename (default:
                        "unwrapped.nii")
  -t, --echo-times ECHO-TIMES [ECHO-TIMES...]
                        The echo times required for temporal
                        unwrapping specified in array or range syntax
                        (eg. "[1.5,3.0]" or "3.5:3.5:14"). For
                        identical echo times, "-t epi" can be used
                        with the possibility to specify the echo time
                        as e.g. "-t epi 5.3" (for B0 calculation).
  -k, --mask MASK [MASK...]
                        nomask | qualitymask <threshold> | robustmask
                        | <mask_file>. <threshold> for qualitymask in
                        [0;1] (default: ["robustmask"])
  -u, --mask-unwrapped  Apply the mask on the unwrapped result. If
                        mask is "nomask", sets it to "robustmask".
  -e, --unwrap-echoes UNWRAP-ECHOES [UNWRAP-ECHOES...]
                        Load only the specified echoes from disk
                        (default: [":"])
  -w, --weights WEIGHTS
                        romeo | romeo2 | romeo3 | romeo4 | romeo6 |
                        bestpath | <4d-weights-file> | <flags>.
                        <flags> are up to 6 bits to activate
                        individual weights (eg. "1010"). The weights
                        are (1)phasecoherence
                        (2)phasegradientcoherence (3)phaselinearity
                        (4)magcoherence (5)magweight (6)magweight2
                        (default: "romeo")
  -B, --compute-B0      Calculate combined B0 map in [Hz]. Phase
                        offset correction might be necessary if not
                        coil-combined with MCPC3Ds/ASPIRE.
  --phase-offset-correction, --coil-combination [PHASE-OFFSET-CORRECTION]
                        on | off | bipolar. Applies the MCPC3Ds method
                        to perform phase offset determination and
                        removal (for multi-echo). This option also
                        allows 5D input, where the 5th dimension is
                        channels. Coil combination will be performed.
                        "bipolar" removes eddy current artefacts
                        (requires >= 3 echoes). (default: "off",
                        without arg: "on")
  --write-phase-offsets
                        Saves the estimated phase offsets to the
                        output folder
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
                        manually wrapping them first.
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
### v3.1
- quality map output is corrupted. Fixed in v3.1.1

## Feedback
Feature requests and bug reports are welcome!
