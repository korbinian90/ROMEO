# ROMEO Unwrapping
Unwrapping of 3D and 4D datasets.
Coil combination of 5D datasets.  

![ROMEO](https://user-images.githubusercontent.com/1307522/144416428-0c51a0e5-cb07-4b7d-8571-9fa2dc33f580.png)

## Get ROMEO
Ordered from simple to more involved approaches.
### Option 1: [Download standalone executable within mritools (Linux and Windows)](https://github.com/korbinian90/CompileMRI.jl/releases)
**Download link for mritools as plain text: https://github.com/korbinian90/CompileMRI.jl/releases**  
*Note: It might not work on every Linux distribution due to shared library incompatibilities (https://github.com/korbinian90/ROMEO/issues/16)*  
*If a compiled MacOS version is required, see [Known-issues/MacOS](https://github.com/korbinian90/ROMEO#macos), but the suggested way is Option 4*

### Option 2: Run in [Neurodesk](https://neurodesk.github.io/) (every OS)
Neurodesk is an analysis environment for reproducible neuroimaging running in a docker container.
It comes with a range of useful tools preinstalled, including ROMEO.
(https://neurodesk.github.io/)

### Option 3: Use the julia command line caller (every OS)
Step by step explanation here: https://github.com/korbinian90/RomeoApp.jl

### Option 4: Compile ROMEO (every OS)
Follow the steps in https://github.com/korbinian90/CompileMRI.jl

### Option 5: Use the julia package [ROMEO.jl](https://github.com/korbinian90/ROMEO.jl) (every OS) 
All the flexibility, but requires handling phase offsets and B0 calculation yourself (see [MriResearchTools.jl](https://github.com/korbinian90/MriResearchTools.jl))

## Publication
**ROMEO**: Dymerska, B., Eckstein, K., Bachrata, B., Siow, B., Trattnig, S., Shmueli, K., Robinson, S.D., 2020. Phase Unwrapping with a Rapid Opensource Minimum Spanning TreE AlgOrithm (ROMEO). Magnetic Resonance in Medicine. https://doi.org/10.1002/mrm.28563

**MCPC-3D-S Coil Combination**:
Eckstein, K., Dymerska, B., Bachrata, B., Bogner, W., Poljanc, K., Trattnig, S., Robinson, S.D., 2018. Computationally Efficient Combination of Multi-channel Phase Data From Multi-echo Acquisitions (ASPIRE). Magnetic Resonance in Medicine 79, 2996–3006. https://doi.org/10.1002/mrm.26963

## Getting Started
### Prerequisites
Phase (and optionally Magnitude) images in NIfTI fileformat.  
For multi-echo/multi-timepoint data, 4D-NIfTI files are used with echoes/timepoints in the 4th dimension.   
Individual 3D files can be merged into 4D files using [fslmerge](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/Fslutils).  
If 5D-NIfTI datasets with channels in the 5th dimension are given, coil combination is performed as first step. 

### Run ROMEO
ROMEO is a command line application. The binary is in the folder `mritools/bin`

Example usage for B0-mapping with a 5-echo scan with TE = [3,6,9,12,15] ms:  
`$ romeo ph.nii -m mag.ii -B -t 3:3:15 -o outputdir`

Example usage for single-echo data:  
`$ romeo ph.nii -m mag.ii -k nomask -o outputdir`

Example for multiple time points with identical echo time (fMRI):  
`$ romeo ph.nii -m mag.ii -k nomask -t epi -o outputdir`

Example usage for a 3-echo scan with TE = [3,6,9] ms:  
`$ romeo ph.nii -m mag.ii -t [3,6,9] -o outputdir`

Note that echo times are required for unwrapping multi-echo data.

## Different Use Cases
### Multi-Echo
If multi-echo data is available, supplying ROMEO with multi-echo information should improve the unwrapping accuracy. The same is true for magnitude information.

### Coil Combination
Coil combination will be automatically performed for 5D datasets using **MCPC-3D-S**. The echoes have to be in the 4th dimension and the channels in the 5th dimension. For bipolar datasets use `--phase-offset-correction bipolar` as additional argument (bipolar correction requires >= 3 echoes).

### Repeated Measurements (EPI)
4D data with an equal echo time for all volumes should be unwrapped as 4D for best accuracy and temporal stability. The echo times can be set to `-t epi`.

### Setting the Template Echo
In certain cases, the phase of the first echo/time-point looks differently than the rest of the acquisition, which can occur due to flow compensation of only the first echo or not having reached the steady state in fMRI. This might cause template unwrapping to fail, as the first echo is chosen as the template by default.  
With the optional argument `--template 2`, this can be changed to the second (or any other) echo/time-point.

### Phase Offsets
If the multi-echo data contains *large phase offsets* (phase at echo time zero), default template unwrapping might fail. Setting the `--individual-unwrapping` flag is a solution, as it performs spatial unwrapping for each echo instead. The computed B0 map is not corrected for remaining phase offsets.

For proper handling, the phase offsest can be removed using **MCPC-3D-S** with the option `--phase-offset-correction`. This works for monopolar and bipolar data, already combined or uncombined channels. However, this requires "linear phase evolution". If the phase is already "corrupted" by other coil combination algorithms, it might not be possible to estimate and remove the phase offsets.

### Disconnected Regions
For datasets with disconnected regions, the `--max-seeds`, `--correct-regions` and `--merge-regions` options might be of interest.

The option `--max-seeds` creates multiple regions, which are unwrapped independently. The detected regions are written to the output file `regions.nii`.

### Fat and Water - Multi-Echo
For acquisitions, in which fat and water signals mix, the assumption of *linear phase evolution* might be broken. Both temporal unwrapping and MCPC-3D-S phase offset removal depend on linear phase evolution and might produce incorrect results, leading to multi-echo unwrapping problems and remaining phase offsets in the resulting B0 map.

Using the `--individual-unwrapping` flag might improve the unwrapping performance by disabling temporal unwrapping. Still, a calculated B0 map might contain unwanted phase offsets.

## Common Pitfalls
### Phase Input
The input data is automatically rescaled to [-π;π]. For example, the input data can be given from [0;4095], which is rescaled to the range [-π;π]. In case the data is already in radians, this can be deactivated using the flag `--no-rescale`. This might be necessary for phase difference data, where calculating the phase difference increases the range to [-2π;2π], but no rescaling to [-π;π] should occur.

## Help on arguments:
```
$ ./bin/romeo

usage: <PROGRAM> [-p PHASE] [-m MAGNITUDE] [-o OUTPUT]
                 [-t ECHO-TIMES [ECHO-TIMES...]] [-k MASK [MASK...]]
                 [-u] [-e UNWRAP-ECHOES [UNWRAP-ECHOES...]]
                 [-w WEIGHTS] [-B]
                 [--phase-offset-correction [PHASE-OFFSET-CORRECTION]]
                                  [--phase-offset-smoothing-sigma-mm PHASE-OFFSET-SMOOTHING-SIGMA-MM [PHASE-OFFSET-SMOOTHING-SIGMA-MM...]]
                 [--write-phase-offsets] [-i] [--template TEMPLATE]
                 [-N] [--no-rescale] [--threshold THRESHOLD] [-v] [-g]
                 [-q] [-Q] [-s MAX-SEEDS] [--merge-regions]
                 [--correct-regions] [--wrap-addition WRAP-ADDITION]
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
                        | <mask_file>. <threshold>=0.1 for qualitymask
                        in [0;1] (default: ["robustmask"])
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
  -B, --compute-B0      Calculate combined B0 map in [Hz]. This
                        activates MCPC3Ds phase offset correction
                        (monopolar) for multi-echo data.
  --phase-offset-correction [PHASE-OFFSET-CORRECTION]
                        on | off | bipolar. Applies the MCPC3Ds method
                        to perform phase offset determination and
                        removal (for multi-echo). "bipolar" removes
                        eddy current artefacts (requires >= 3 echoes).
                        (default: "off", without arg: "on")
  --phase-offset-smoothing-sigma-mm PHASE-OFFSET-SMOOTHING-SIGMA-MM [PHASE-OFFSET-SMOOTHING-SIGMA-MM...]
                        default: [7,7,7] Only applied if
                        phase-offset-correction is activated. The
                        given sigma size is divided by the voxel size
                        from the nifti phase file to obtain a
                        smoothing size in voxels. A value of [0,0,0]
                        deactivates phase offset smoothing (not
                        recommended).
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
                        default: 1)
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

## Related Repositories
The sourcecode is available under [ROMEO.jl](https://github.com/korbinian90/ROMEO.jl).  
The binaries are a standalone compiled version of [RomeoApp.jl](https://github.com/korbinian90/RomeoApp.jl). The compilation scripts and recent releases are in [CompileMRI.jl](https://github.com/korbinian90/CompileMRI.jl).

## Known issues
### v1.4
- single echo unwrapping with magnitude and default mask leads to ReadOnlyMemory error. Fixed in v1.4.1
- binary data type for mask not supported. Fixed in v2.0.1
### v2.0.1
- input scaling issue (with INT16). Fixed in v2.0.2
### v3.1
- quality map output is corrupted. Fixed in v3.1.1
### MacOS
To run romeo executables on MacOS, multiple files have to be flagged as save to execute. Additionally, the executable might only run on specific OS versions. You can try the [newest](https://github.com/korbinian90/CompileMRI.jl/releases) MacOS executable, [v3.2.2](https://github.com/korbinian90/ROMEO/releases/tag/v3.2.2) and [v3.1](https://github.com/korbinian90/ROMEO/releases/tag/v3.1). This problem is still unsolved and no clear way how to improve the compatibility. Please look at the different ways to run it on MacOS above.

### Issues when calling from MATLAB  

- `libstdc++` error (libstdc++.so.6: version `GLIBCXX_3.4.29' not found)  
- Segmentation fault (error code 139)  

Problem: ROMEO crashes with version conflicts of shared dependencies when called within MATLAB via `unix()` or `system()`.  
Reason: MATLAB runs other programs with a [modified environment variable](https://discourse.julialang.org/t/ann-juliafrommatlab-jl-call-julia-from-matlab/66882/2)  
Solution: clean LD_LIBRARY_PATH before execution and restore it afterwards  
Example:  
```matlab
% remove matlab specific LD_LIBRARY_PATH
if isunix; paths = getenv('LD_LIBRARY_PATH'); setenv('LD_LIBRARY_PATH'); end
success = system(<romeo call>);
% restore paths
if isunix; setenv('LD_LIBRARY_PATH', paths); end
```

## Feedback
Feature requests and bug reports are welcome!
