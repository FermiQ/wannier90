## Overview
The `w90_types` module defines various derived data types used throughout the Wannier90 code, particularly those shared between the main `wannier90.x` executable and the post-processing tool `postw90.x`. These types encapsulate related variables, parameters, and data structures, promoting better organization and modularity. They cover aspects like output control, system physical information, timing, Wigner-Seitz cell parameters, Wannier function properties, k-mesh details, disentanglement manifold information, atomic data, and k-point paths for plotting.

## Key Components

### Module `w90_types`
- **Description:** The central module for defining shared derived data types.

### Type `print_output_type`
- **Description:** Contains variables to control output file formatting and verbosity levels.
- **Fields:**
    - `iprint` (integer): Controls the general verbosity of the output.
    - `timing_level` (integer): Controls the verbosity of timing information.
    - `length_unit` (character(len=20)): Unit for length (e.g., "ang", "bohr").
    - `lenconfac` (real(kind=dp)): Conversion factor associated with `length_unit`.

### Type `w90_system_type`
- **Description:** Holds physical information about the material system being calculated.
- **Fields:**
    - `num_valence_bands` (integer): Number of valence bands.
    - `num_elec_per_state` (integer): Number of electrons per state (1 for spinors, 2 otherwise).
    - `spinors` (logical): True if the Wannier functions are spinors.

### Type `timing_data_type`
- **Description:** Stores data for an individual stopwatch/timer.
- **Fields:**
    - `ncalls` (integer): Number of times this stopwatch has been called.
    - `ctime` (real(kind=DP)): Total accumulated CPU time on this stopwatch.
    - `ptime` (real(kind=DP)): Temporary record of CPU time when the stopwatch was started.
    - `label` (character(len=60)): A descriptive label for what is being timed.

### Type `timer_list_type`
- **Description:** An array of `timing_data_type` to manage multiple stopwatches.
- **Fields:**
    - `clocks(nmax)` (type(timing_data_type)): Array of individual stopwatches. `nmax` is a parameter (default 100).
    - `nnames` (integer): Number of currently active stopwatches.
    - `overflow` (logical): True if `nnames` exceeds `nmax`.

### Type `ws_region_type`
- **Description:** Parameters related to the Wigner-Seitz cell construction and distance calculations.
- **Fields:**
    - `use_ws_distance` (logical): Flag to enable Wigner-Seitz distance usage.
    - `ws_distance_tol` (real(kind=dp)): Tolerance for Wigner-Sietz distance comparisons.
    - `ws_search_size(3)` (integer): Maximum extension in each direction for searching points within the Wigner-Seitz cell.

### Type `ws_distance_type`
- **Description:** Stores data related to Wigner-Seitz cell distances between Wannier functions.
- **Fields:**
    - `irdist(:, :, :, :, :)` (integer, allocatable): Integer shifts to bring WF_j into WS cell of WF_i for R-vector.
    - `crdist(:, :, :, :, :)` (real(DP), allocatable): Cartesian version of `irdist_ws`.
    - `ndeg(:, :, :)` (integer, allocatable): Number of equivalent vectors for each (i,j,R) pair.
    - `done` (logical): Flag indicating if these properties have been calculated.

### Type `wannier_data_type`
- **Description:** Contains information about the resulting Maximally Localised Wannier Functions (MLWFs).
- **Fields:**
    - `centres(:, :)` (real(kind=dp), allocatable): Cartesian coordinates of the MLWF centers.
    - `spreads(:)` (real(kind=dp), allocatable): Spreads of the MLWFs.

### Type `kmesh_input_type`
- **Description:** User-provided parameters for k-mesh determination.
- **Fields:**
    - `num_shells` (integer): Number of k-point neighbor shells to use (often determined automatically).
    - `skip_B1_tests` (logical): If true, skips the B1 completeness condition check.
    - `shell_list(:)` (integer, allocatable): List of shell indices to use if specified manually.
    - `search_shells` (integer): Number of shells to search for neighbors.
    - `tol` (real(kind=dp)): Tolerance for k-mesh operations.

### Type `proj_input_type`
- **Description:** User-provided information for defining the initial guess projections.
- **Fields:**
    - `site(:, :)` (real(kind=dp), allocatable): Coordinates of projection centers.
    - `l(:)` (integer, allocatable): Angular momentum quantum number (s, p, d, f, or hybrids).
    - `m(:)` (integer, allocatable): Magnetic quantum number or specific orbital index.
    - `s(:)` (integer, allocatable): Spin index (for spinors).
    - `s_qaxis(:, :)` (real(kind=dp), allocatable): Spin quantization axis.
    - `z(:, :)` (real(kind=dp), allocatable): z-axis orientation for the orbital.
    - `x(:, :)` (real(kind=dp), allocatable): x-axis orientation for the orbital.
    - `radial(:)` (integer, allocatable): Index of the radial part of the projection (e.g., for multi-zeta).
    - `zona(:)` (real(kind=dp), allocatable): Diffusivity (zona factor) of the projection.
    - `auto_projections` (logical): Flag to enable automatic generation of projections.

### Type `kmesh_info_type`
- **Description:** Derived information about the k-mesh and its connectivity.
- **Fields:**
    - `nnh` (integer): Number of unique b-vector directions (excluding inversion).
    - `nntot` (integer): Total number of k-point neighbors for each k-point.
    - `nnlist(:, :)` (integer, allocatable): List of neighbor k-point indices.
    - `neigh(:, :)` (integer, allocatable): Maps to `bk` for unique directions.
    - `nncell(:, :, :)` (integer, allocatable): G-vector (in reciprocal lattice units) for each neighbor.
    - `wbtot` (real(kind=dp)): Sum of b-vector weights.
    - `wb(:)` (real(kind=dp), allocatable): Weights associated with each b-vector.
    - `bk(:, :, :)` (real(kind=dp), allocatable): The b-vectors $\mathbf{k'} + \mathbf{G} - \mathbf{k}$.
    - `bka(:, :)` (real(kind=dp), allocatable): Unique b-vector directions.
    - `explicit_nnkpts` (logical): True if nnkp info was read from file.

### Type `dis_manifold_type`
- **Description:** Information about the manifold of states used for disentanglement.
- **Fields:**
    - `win_min`, `win_max` (real(kind=dp)): Energy boundaries of the outer disentanglement window.
    - `froz_min`, `froz_max` (real(kind=dp)): Energy boundaries of the inner (frozen) disentanglement window.
    - `frozen_states` (logical): True if a frozen window is used.
    - `ndimwin(:)` (integer, allocatable): Number of states within the outer window at each k-point.
    - `lwindow(:, :)` (logical, allocatable): Mask indicating which states are within the outer window.

### Type `atom_data_type`
- **Description:** Stores information about the atomic structure of the system.
- **Fields:**
    - `pos_cart(:, :, :)` (real(kind=dp), allocatable): Cartesian coordinates of atoms, per species.
    - `species_num(:)` (integer, allocatable): Number of atoms of each species.
    - `label(:)` (character(len=maxlen), allocatable): User-defined labels for each atomic species (e.g., "Si1", "Si2").
    - `symbol(:)` (character(len=2), allocatable): Chemical symbol for each species (e.g., "Si").
    - `num_atoms` (integer): Total number of atoms.
    - `num_species` (integer): Number of distinct atomic species.

### Type `kpoint_path_type`
- **Description:** Defines the k-point path for band structure plotting.
- **Fields:**
    - `num_points_first_segment` (integer): Number of points for the first segment of the path (others scaled).
    - `labels(:)` (character(len=20), allocatable): Labels for the high-symmetry k-points.
    - `points(:, :)` (real(kind=dp), allocatable): Fractional coordinates of the high-symmetry k-points defining the path.

## Important Variables/Constants
- **`dp` (integer, parameter, from `w90_constants`):** Defines the kind for double precision real numbers.
- **`maxlen` (integer, parameter, from `w90_constants`):** Maximum length of character strings read from input.
- **`max_shells` (integer, parameter, value=6):** Maximum number of k-point neighbor shells considered in k-mesh generation.
- **`num_nnmax` (integer, parameter, value=12):** Maximum number of nearest neighbors a k-point can have (related to `max_shells`).
- **`nmax` (integer, parameter, value=100):** Maximum number of stopwatches for timing.

## Usage Examples
These types are used throughout Wannier90 to declare variables that hold structured data.

```fortran
! Example of declaring a variable of type print_output_type
use w90_types, only: print_output_type
type(print_output_type) :: output_params

! Accessing fields
output_params%iprint = 2
output_params%length_unit = 'ang'

! Example of using atom_data_type
use w90_types, only: atom_data_type
type(atom_data_type) :: atoms
! ... (atoms would be populated by reading input or from library calls) ...
if (atoms%num_atoms > 0) then
  write(*,*) "First atom of first species:", atoms%symbol(1), atoms%pos_cart(:,1,1)
end if
```
These types are primarily instantiated and populated by input reading routines (e.g., in `w90_readwrite` and `w90_wannier90_readwrite`) or by other computational modules (e.g., `kmesh_info_type` by `w90_kmesh`).

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: Uses `dp` for precision and `maxlen` for string length.

- **External Libraries:** None.

- **Interactions:**
    - This module is fundamental and is `USE`d by almost all other modules in Wannier90 that need to work with these shared data structures.
    - It provides a standardized way to group and pass related data between different parts of the code.
    - The definitions here are critical for how data is read from input files, stored during calculations, and passed to output routines.
```
