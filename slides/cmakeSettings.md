## CMake Settings

notes:
As a first example of simple errors should have simple solutions,
let's look at CMake.

***

### Role of CMake in GCP

![imgs](imgs/cmake.png) <!-- .element: height="500px" -->

Single location for global constants (e.g. number of receivers)

notes:
Current problem: there is dependence between some variables
If things don't match, GCP crashes and disappears!

***

## CMake Additions

`GCP` now takes into account the `docker` slot of each `SMuRF` layer

***

#### `scripts/controlSystem.in`

```perl [|8-18]
#-----------------------------------------------------------------------
# Basic checking that CMakeCache.txt variables regarding BA MCES
# and SMURF are consistent with one another
#-----------------------------------------------------------------------
sub checkCMakeVariables {
    $FIX = "To change these values, run the following in gcp_build directory:\n\n\tccmake .\n\nLists must have the form of values separated by semicolons, i.e. 1;2;3.";

    if ($GCP_TOTAL_SMURFS ne scalar @GCP_SMURFD_HOST) {
        die "GCP_BA_SMURFS must equal size of GCP_SMURF_HOSTS array in gcp_build/CMakeCache.txt.\n$FIX";
    }

    if ($GCP_TOTAL_SMURFS ne scalar @GCP_SMURF_SLOTS) {
        die "GCP_BA_SMURFS must equal size of GCP_SMURF_SLOTS array in gcp_build/CMakeCache.txt.\n$FIX";
    }

    if ($GCP_TOTAL_MCES ne scalar @GCP_MASD_HOST) {
        die "GCP_BA_MCES must equal size of GCP_MCE_HOSTS array in gcp_build/CMakeCache.txt.\n$FIX";
    }
}

```

***

## demo

```shell []
GCP_BA_SMURFS=1, GCP_SMURF_HOSTS=localhost, GCP_SMURF_SLOTS=4
GCP_BA_SMURFS=1, GCP_SMURF_HOSTS=localhost, GCP_SMURF_SLOTS=4;10
GCP_BA_SMURFS=2, GCP_SMURF_HOSTS=localhost, GCP_SMURF_SLOTS=4;10
GCP_BA_SMURFS=2, GCP_SMURF_HOSTS=localhost;localhost, GCP_SMURF_SLOTS=4;10
```

notes:
1. Fail
2. Fail
3. Fail
4. Correct
