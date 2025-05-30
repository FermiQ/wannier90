## Overview
The `w90_kslice` module, part of `postw90.x`, is designed to calculate and visualize various k-space properties on a 2D slice of the Brillouin zone (BZ). The slice is defined by a corner point and two basis vectors. The module can:
-   Plot Fermi lines (intersections of the Fermi surface with the slice).
-   Create heatmap plots of quantities like Berry curvature, orbital magnetization components, or spin Hall conductivity components on the slice.
-   Color-code Fermi lines by spin projection.

## Key Components

### Module `w90_kslice`
- **Description:** The main module for calculations on a 2D k-space slice.

### Subroutine `k_slice`
- **Description:** This is the primary driver routine for this module. It orchestrates the calculation of properties on the specified k-slice.
    1.  Determines which tasks to perform (Fermi lines, curvature heatmap, etc.) based on `pw90_kslice%task`.
    2.  Calls `get_*_R` routines from `w90_get_oper` to ensure necessary real-space operator matrix elements are available.
    3.  Defines the 2D k-mesh on the slice based on `pw90_kslice%corner`, `pw90_kslice%b1`, `pw90_kslice%b2`, and `pw90_kslice%kmesh2d`.
    4.  Iterates through the k-points on this 2D mesh:
        - If Fermi lines or heatmaps are requested, it calculates band energies $E_{n\mathbf{k}}$ by Fourier transforming $H(\mathbf{R})$ and diagonalizing $H(\mathbf{k})$.
        - If spin-coloring for Fermi lines is requested, it computes spin expectation values.
        - If Berry curvature, orbital magnetization, or SHC heatmaps are requested, it calls the respective `berry_get_*_klist` routines.
    5.  Gathers results (if running in parallel).
    6.  Writes data to output files (`seedname-kslice-coord.dat`, `seedname-kslice-bands.dat`, `seedname-kslice-curv.dat`, etc.) and generates Gnuplot and/or Python scripts for visualization.
- **Parameters:**
    - `pw90_kslice` (type `pw90_kslice_mod_type`): Contains k-slice specific settings (task, slice definition, 2D k-mesh).
    - Other parameters are similar to those in `w90_berry_main` or `w90_kpath_main`, providing system information, Wannier data, operator matrix elements, and control flags.
- **Returns:** Generates various data and script files for visualizing properties on the k-slice.

### Subroutine `kslice_print_info`
- **Description:** Prints information about the requested k-slice tasks and parameters to the standard output.
- **Parameters:** `plot_fermi_lines`, `fermi_lines_color`, `plot_curv`, `plot_morb`, `plot_shc`, `stdout`, `pw90_berry`, `fermi_energy_list`, `error`, `comm`.

### Private Helper Subroutines for Plotting Scripts
- **`write_data_file(stdout, filename, fmt, data)`:** A generic routine to write 2D array data to a file.
- **`write_coords_file(stdout, filename, fmt, coords, vals, mask, blocklen)`:** Writes data associated with coordinates, potentially masked or formatted in blocks (e.g., for Gnuplot's grid data).
- **`script_common(scriptunit, areab1b2, square, seedname)`:** Writes common header and setup information for Python plotting scripts (imports, loading coordinate data).
- **`script_fermi_lines(scriptunit, seedname, fermi_energy_list)`:** Writes Python code for plotting Fermi lines using `matplotlib.pyplot.contour`.

## Important Variables/Constants

- **`pw90_kslice_mod_type` (from `postw90_types.F90`):**
    - `task` (character): String specifying tasks (e.g., 'fermi_lines', 'curv', 'morb', 'shc').
    - `corner(3)` (real(dp)): Fractional coordinates of the corner of the k-slice.
    - `b1(3)`, `b2(3)` (real(dp)): Two vectors (fractional coordinates) spanning the k-slice.
    - `kmesh2d(2)` (integer): Number of points along `b1` and `b2` directions for the 2D mesh.
    - `fermi_lines_colour` (character): How to color Fermi lines ('none', 'spin').
- **Output data arrays:** `coords` (k-point coordinates on slice), `bandsdata` (energies), `spndata` (spin values for coloring), `spnmask` (mask for Fermi surface points), `zdata` (heatmap values like curvature).

## Usage Examples
This module is invoked by `postw90.x` when `kslice = .true.` is set in the `seedname.win` input file.

**Example `seedname.win` section:**
```
kslice = .true.
kslice_task = 'fermi_lines,curv' ! Plot Fermi lines and Berry curvature heatmap
kslice_corner = 0.0 0.0 0.0      ! Corner of the slice
kslice_b1 = 1.0 0.0 0.0          ! First vector spanning the slice
kslice_b2 = 0.0 1.0 0.0          ! Second vector spanning the slice
kslice_2dkmesh = 50 50           ! 50x50 grid on the slice

fermi_energy = 0.0               ! Needed for Fermi lines and curvature

! Other relevant parameters for Berry curvature (e.g., berry_kmesh for adaptive smearing if used by berry_get_imf_klist)
berry = .true. ! Enable berry module if curv, morb or shc is needed.
berry_kmesh = 20 20 20
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `twopi`, `eps8`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication.
    - `w90_io`: For file I/O and timing.
    - `w90_types`: For shared data structures.
    - `w90_postw90_types`: For `postw90`-specific types.
    - `w90_utility`: For `utility_diagonalize`, `utility_recip_lattice_base`.
    - `w90_get_oper`: To obtain real-space operator matrix elements.
    - `w90_postw90_common`: For Fourier transform routines.
    - `w90_wan_ham`: For `wham_get_eig_deleig`.
    - `w90_berry`: Uses `berry_get_imf_klist`, `berry_get_imfgh_klist`, `berry_get_shc_klist`.
    - `w90_spin`: For `spin_get_nk` if `fermi_lines_colour = 'spin'`.

- **External Libraries:**
    - **LAPACK/BLAS:** Used implicitly for matrix operations.

- **Interactions:**
    - `k_slice` is the entry point called by `postw90.x`.
    - Reads slice definition and task from `pw90_kslice` type, populated from `seedname.win`.
    - Relies on real-space matrix elements from `w90_get_oper`.
    - Outputs data files (e.g., `seedname-kslice-coord.dat`, `seedname-kslice-bands.dat`, `seedname-kslice-curv.dat`) and corresponding Gnuplot/Python scripts for visualization of properties on the defined 2D k-space slice.
```
