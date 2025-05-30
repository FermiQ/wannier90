## Overview
The `w90_postw90_types` module defines derived data types that are specifically used by the `postw90.x` executable. These types are generally not used by the main `wannier90.x` program. They serve to encapsulate parameters and settings for various post-processing calculations, such as Density of States (DOS), Berry phase related properties (AHC, orbital magnetization, Kubo conductivity, SHC), k-path and k-slice plotting, gyrotropic effects, generic band interpolation, and Boltzmann transport. This modular approach to type definition helps in organizing the large number of parameters associated with these diverse functionalities.

## Key Components

### Module `w90_postw90_types`
- **Description:** The central module for defining `postw90.x`-specific derived data types.

### Type `pw90_calculation_type`
- **Description:** Contains logical flags that determine which high-level post-processing tasks `postw90.x` will perform.
- **Fields:** (all logical)
    - `kpath`: Perform calculations along a k-path.
    - `kslice`: Perform calculations on a k-slice.
    - `dos`: Calculate Density of States.
    - `berry`: Calculate Berry phase related properties.
    - `gyrotropic`: Calculate gyrotropic effects.
    - `geninterp`: Perform generic band interpolation from a k-point file.
    - `boltzwann`: Perform Boltzmann transport calculations.
    - `spin_moment`: Calculate spin magnetic moments.
    - `spin_decomp`: Perform spin decomposition for relevant properties.

### Type `pw90_oper_read_type`
- **Description:** Flags to control the format for reading operator files (e.g., `.spn`, `.uHu`).
- **Fields:**
    - `spn_formatted` (logical): True if `seedname.spn` is formatted text, false if binary.
    - `uHu_formatted` (logical): True if `seedname.uHu` is formatted text, false if binary.

### Type `kmesh_spacing_type`
- **Description:** Defines a k-mesh either by a uniform spacing or by explicit grid dimensions.
- **Fields:**
    - `spacing` (real(kind=dp)): Desired k-point spacing (e.g., in Angstrom$^{-1}$). If positive, `mesh` is derived from this.
    - `mesh(3)` (integer): Explicit dimensions of the k-mesh ($N_1 \times N_2 \times N_3$).

### Type `pw90_spin_mod_type`
- **Description:** Parameters for spin-related calculations in `postw90.x`.
- **Fields:**
    - `axis_polar` (real(kind=dp)): Polar angle ($\theta$) of the spin quantization axis.
    - `axis_azimuth` (real(kind=dp)): Azimuthal angle ($\phi$) of the spin quantization axis.
    - `kmesh` (type(kmesh_spacing_type)): K-mesh settings for spin calculations.

### Type `pw90_band_deriv_degen_type`
- **Description:** Parameters for handling degenerate bands when calculating band derivatives (velocities).
- **Fields:**
    - `use_degen_pert` (logical): If true, use degenerate perturbation theory.
    - `degen_thr` (real(kind=dp)): Energy threshold to consider bands degenerate.

### Type `pw90_kpath_mod_type`
- **Description:** Control parameters for k-path calculations (e.g., band structure plots).
- **Fields:**
    - `task` (character(len=20)): Specifies tasks for k-path (e.g., 'bands', 'curv').
    - `num_points` (integer): Number of points for the first segment of the k-path.
    - `bands_colour` (character(len=20)): How to color bands (e.g., 'none', 'spin', 'shc').

### Type `pw90_kslice_mod_type`
- **Description:** Control parameters for k-slice calculations.
- **Fields:**
    - `task` (character(len=20)): Specifies tasks for k-slice (e.g., 'fermi_lines', 'curv_heatmap').
    - `corner(3)` (real(kind=dp)): Fractional coordinates of the slice origin.
    - `b1(3)`, `b2(3)` (real(kind=dp)): Vectors spanning the slice (fractional coordinates).
    - `kmesh2d(2)` (integer): Dimensions of the 2D k-mesh on the slice.
    - `fermi_lines_colour` (character(len=20)): How to color Fermi lines (e.g., 'none', 'spin').

### Type `pw90_smearing_type`
- **Description:** Parameters for energy smearing schemes.
- **Fields:**
    - `use_adaptive` (logical): If true, use adaptive smearing based on band velocities.
    - `adaptive_prefactor` (real(kind=dp)): Prefactor for adaptive smearing width.
    - `type_index` (integer): Code for smearing type (Gaussian, M-P, Cold Smearing).
    - `fixed_width` (real(kind=dp)): Smearing width for fixed smearing schemes (in eV).
    - `adaptive_max_width` (real(kind=dp)): Maximum width for adaptive smearing (in eV).
    - `max_arg` (real(kind=dp)): Cutoff for smearing function arguments (units of smearing width).

### Type `pw90_dos_mod_type`
- **Description:** Parameters for Density of States (DOS) calculations.
- **Fields:**
    - `task` (character(len=20)): Task for DOS (e.g., 'dos_plot').
    - `smearing` (type(pw90_smearing_type)): Nested smearing parameters.
    - `energy_min`, `energy_max`, `energy_step` (real(kind=dp)): Energy range and step for DOS.
    - `num_project` (integer): Number of WFs to project DOS onto.
    - `project(:)` (integer, allocatable): List of WF indices for projection.
    - `kmesh` (type(kmesh_spacing_type)): K-mesh for DOS calculation.

### Type `pw90_berry_mod_type`
- **Description:** Parameters for Berry module calculations (AHC, orbital mag, Kubo, SC, SHC, k.p).
- **Fields:** (Many fields, see `berry.F90` or `w90_postw90_readwrite.F90` for full list)
    - `task` (character(len=120)): String listing tasks.
    - `kmesh` (type(kmesh_spacing_type)): K-mesh for interpolation.
    - `kubo_nfreq` (integer): Number of frequency points.
    - `kubo_freq_list(:)` (complex(dp), allocatable): Frequencies for optical properties.
    - `kubo_smearing` (type(pw90_smearing_type)): Smearing for Kubo formula.
    - `curv_adpt_kmesh`, `curv_adpt_kmesh_thresh`, `curv_unit`: Adaptive mesh for curvature.
    - `kdotp_kpoint(3)`, `kdotp_bands(:)`: For k.p calculations.

### Type `pw90_spin_hall_type`
- **Description:** Parameters for Spin Hall Conductivity (SHC) calculations.
- **Fields:**
    - `freq_scan` (logical): If true, scan SHC vs. frequency.
    - `alpha`, `beta`, `gamma` (integer): Cartesian indices for $\sigma_{\alpha\beta}^{\gamma}$.
    - `bandshift`, `bandshift_firstband`, `bandshift_energyshift`: Scissor operator for SHC.
    - `method` (character(len=120)): SHC calculation method ('qiao' or 'ryoo').

### Type `pw90_gyrotropic_type`
- **Description:** Parameters for gyrotropic effects calculations.
- **Fields:** (Many fields, see `gyrotropic.F90` or `w90_postw90_readwrite.F90`)
    - `task` (character(len=120)): String listing tasks.
    - `kmesh` (type(kmesh_spacing_type)): K-mesh.
    - `nfreq`, `freq_list(:)`: Frequency parameters.
    - `box_corner(3)`, `box(3,3)`: Defines k-space integration box.
    - `smearing` (type(pw90_smearing_type)): Smearing parameters.

### Type `pw90_geninterp_mod_type`
- **Description:** Parameters for generic band interpolation.
- **Fields:**
    - `alsofirstder` (logical): If true, also calculate band derivatives.
    - `single_file` (logical): If true, write all output to a single file (MPI).

### Type `pw90_boltzwann_type`
- **Description:** Parameters for Boltzmann transport calculations.
- **Fields:** (Many fields, see `boltzwann.F90` or `w90_postw90_readwrite.F90`)
    - `calc_also_dos` (logical): If true, also calculate DOS.
    - `dir_num_2d` (integer): Specifies 2D transport direction.
    - `dos_smearing`, `tdf_smearing` (type(pw90_smearing_type)): Smearing for DOS and TDF.
    - `mu_min`, `mu_max`, `mu_step`: Chemical potential range.
    - `temp_min`, `temp_max`, `temp_step`: Temperature range.
    - `kmesh` (type(kmesh_spacing_type)): K-mesh for BoltzWann.
    - `relax_time` (real(kind=dp)): Constant relaxation time.

### Type `wigner_seitz_type`
- **Description:** Stores information about the Wigner-Seitz cell R-vectors.
- **Fields:**
    - `irvec(:, :)` (integer, allocatable): Integer components of R-vectors.
    - `crvec(:, :)` (real(dp), allocatable): Cartesian coordinates of R-vectors.
    - `ndegen(:)` (integer, allocatable): Degeneracy of each R-vector.
    - `nrpts` (integer): Number of R-vectors.
    - `rpt_origin` (integer): Index of the R=0 vector.

### Type `kpoint_dist_type`
- **Description:** Stores k-points read from an external file (e.g., `kpoint.dat`).
- **Fields:**
    - `max_int_kpts_on_node`, `num_int_kpts` (integer): Max/total number of k-points.
    - `num_int_kpts_on_node(:)` (integer, allocatable): Number of k-points per MPI node.
    - `int_kpts(:, :)` (real(dp), allocatable): K-point coordinates.
    - `weight(:)` (real(dp), allocatable): Weight of each k-point.

## Important Variables/Constants
- **`dp` (integer, parameter, from `w90_constants`):** Defines the kind for double precision real numbers.
- **`maxlen` (integer, parameter, from `w90_constants`):** Default maximum length for character strings.

## Usage Examples
These types are instantiated in `postw90.F90` and other `postw90` modules. They are populated by the routines in `w90_postw90_readwrite.F90` based on the user's `seedname.win` input file.

```fortran
! Example declaration in a postw90 module
use w90_postw90_types, only: pw90_dos_mod_type
type(pw90_dos_mod_type) :: dos_parameters

! dos_parameters would then be filled by w90_postw90_readwrite_read_dos
! and passed to dos_main.
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp` and `maxlen`.
    - `w90_comms`: For `w90comm_type` (though not directly used in field definitions, it's relevant for how data might be distributed if these types were part of MPI operations).

- **External Libraries:** None.

- **Interactions:**
    - This module is central to `postw90.x`, providing the data structures that define the parameters for all its specific calculation tasks.
    - It is used by `w90_postw90_readwrite` to store parsed input values.
    - Instances of these types are passed to the main computational routines in modules like `w90_berry`, `w90_dos`, `w90_boltzwann`, etc., to control their execution.
```
