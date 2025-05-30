## Overview
The `w90_kpath` module, part of `postw90.x`, is designed to calculate and output various physical properties along a user-defined path in k-space. This is commonly used for generating band structure plots, but can also be used to visualize other k-dependent quantities like Berry curvature, orbital magnetization components, or spin Hall conductivity contributions along specific directions in the Brillouin zone.

## Key Components

### Module `w90_kpath`
- **Description:** The main module for calculations along a k-path.

### Subroutine `k_path`
- **Description:** This is the main driver routine for this module. It orchestrates the calculation of properties along the k-path defined in the `kpoint_path` data structure (read from `seedname.win`).
    1.  It first determines which properties to calculate based on the `pw90_kpath%task` input string (e.g., 'bands', 'curv', 'morb', 'shc').
    2.  It calls the necessary `get_*_R` routines from `w90_get_oper` to ensure the required real-space operator matrix elements are available.
    3.  It calls `k_path_get_points` to generate the list of k-points along the specified path and their corresponding x-axis coordinates for plotting.
    4.  It then iterates through these k-points. For each k-point:
        - If band energies are requested, it calculates them by Fourier transforming $H(\mathbf{R})$ to $H(\mathbf{k})$ and diagonalizing. It can also color-code bands by spin projection or SHC contribution.
        - If Berry curvature ('curv') or orbital magnetization ('morb') components are requested, it calls `berry_get_imf_klist` or `berry_get_imfgh_klist` respectively.
        - If spin Hall conductivity ('shc') components are requested, it calls `berry_get_shc_klist`.
    5.  After processing all k-points, it gathers results (if running in parallel) and calls plotting routines to generate output files for Gnuplot and/or Python/Pylab.
- **Parameters:**
    - `pw90_kpath` (type `pw90_kpath_mod_type`): Contains k-path specific settings (task, color coding).
    - `kpoint_path` (type `kpoint_path_type`): Defines the k-path segments and labels.
    - Other parameters are similar to those in `w90_berry_main` or `w90_dos_main`, providing system information, Wannier data, operator matrix elements, and control flags.
- **Returns:** Generates various output files:
    - `seedname-path.kpt`: List of k-points along the path.
    - `seedname-bands.dat`: Band energies (and color data if requested).
    - `seedname-bands.gnu`: Gnuplot script for bands.
    - `seedname-bands.py`: Python script for bands.
    - `seedname-curv.dat`: Berry curvature components along the path.
    - `seedname-curv_X.gnu`/`.py`: Gnuplot/Python scripts for curvature components.
    - `seedname-morb.dat`: Orbital magnetization components along the path.
    - `seedname-morb_X.gnu`/`.py`: Gnuplot/Python scripts for morb components.
    - `seedname-shc.dat`: Spin Hall conductivity components along the path.
    - `seedname-shc.gnu`/`.py`: Gnuplot/Python scripts for SHC components.
    - `seedname-bands+shc.py` or `seedname-bands+curv_X.py`: Combined plots.

### Subroutine `k_path_get_points`
- **Description:** Generates the explicit list of k-points (`plot_kpoint`) along the path defined by `kpoint_path` and their corresponding scalar x-coordinates (`xval`) for plotting. The density of points is determined by `pw90_kpath%num_points` for the first segment, and other segments are scaled to have a similar density.
- **Parameters:**
    - `num_paths` (integer, out): Number of segments in the path.
    - `kpath_len(:)` (real(dp), allocatable, out): Length of each segment.
    - `total_pts` (integer, out): Total number of k-points generated.
    - `xval(:)` (real(dp), allocatable, out): X-coordinates for plotting.
    - `plot_kpoint(:, :)` (real(dp), allocatable, out): Fractional coordinates of k-points on the path.
    - `kpoint_path` (type `kpoint_path_type`, in): Input k-path definition.
    - `recip_lattice(3,3)` (real(dp), in): Reciprocal lattice vectors.
    - `pw90_kpath` (type `pw90_kpath_mod_type`, in): For `num_points`.
- **Returns:** Populates `num_paths`, `kpath_len`, `total_pts`, `xval`, and `plot_kpoint`.

### Subroutine `k_path_print_info`
- **Description:** Prints information about the requested k-path tasks and parameters to the standard output.
- **Parameters:** `plot_bands`, `plot_curv`, `plot_morb`, `plot_shc`, `fermi_energy_list`, `pw90_kpath`, `berry_curv_unit`, `stdout`, `error`, `comm`.

## Important Variables/Constants

- **`pw90_kpath_mod_type` (from `postw90_types.F90`):**
    - `task` (character): String specifying tasks (e.g., 'bands', 'curv', 'morb', 'shc').
    - `num_points` (integer): Number of points for the first segment of the k-path.
    - `bands_colour` (character): How to color bands ('none', 'spin', 'shc').
- **`kpoint_path_type` (from `w90_types.F90`):**
    - `labels(:)`: Labels for high-symmetry points.
    - `points(:, :)`: Fractional coordinates of high-symmetry points.
- Output arrays for k-resolved quantities: `eig` (energies), `color` (for band coloring), `curv` (Berry curvature), `morb` (orbital magnetization), `shc` (spin Hall conductivity).

## Usage Examples
This module is invoked by `postw90.x` when `kpath = .true.` is set in the `seedname.win` input file. The k-path itself is defined using the `begin kpoint_path ... end kpoint_path` block.

**Example `seedname.win` section:**
```
kpath = .true.
kpath_task = 'bands,curv' ! Calculate bands and Berry curvature
kpath_num_points = 50    ! Points per segment (approx)
bands_colour = 'spin'    ! Color bands by spin projection

begin kpoint_path
  G 0.00  0.00  0.00   X 0.50  0.00  0.00
  X 0.50  0.00  0.00   M 0.50  0.50  0.00
  M 0.50  0.50  0.00   G 0.00  0.00  0.00
end kpoint_path

! Other relevant parameters for berry curvature (e.g., berry_kmesh, fermi_energy)
! must also be set if 'curv', 'morb', or 'shc' tasks are included.
berry = .true. ! Enable berry module if curv, morb or shc is needed.
fermi_energy = 0.0
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `eps8`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication.
    - `w90_io`: For file I/O and timing.
    - `w90_types`: For `kpoint_path_type` and other shared types.
    - `w90_postw90_types`: For `pw90_kpath_mod_type` and other `postw90` types.
    - `w90_utility`: For `utility_diagonalize`, `utility_recip_lattice_base`, `utility_metric`.
    - `w90_get_oper`: To obtain real-space operator matrix elements.
    - `w90_postw90_common`: For Fourier transform routines.
    - `w90_wan_ham`: For `wham_get_eig_deleig` (energies and k-derivatives).
    - `w90_berry`: Uses `berry_get_imf_klist`, `berry_get_imfgh_klist`, `berry_get_shc_klist` for calculating k-dependent Berry-related quantities.
    - `w90_spin`: For `spin_get_nk` if `bands_colour = 'spin'`.

- **External Libraries:**
    - **LAPACK/BLAS:** Used implicitly for matrix diagonalizations and other linear algebra within called routines.

- **Interactions:**
    - `k_path` is the entry point called by `postw90.x`.
    - Reads k-path definition from the `kpoint_path` block in `seedname.win`.
    - Relies on real-space matrix elements (e.g., `HH_R`, `AA_R`) computed by `w90_get_oper`.
    - Outputs data files suitable for plotting with Gnuplot or Python/Pylab, visualizing the requested properties along the k-path.
```
