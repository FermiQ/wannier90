## Overview
The `w90_geninterp` module, part of `postw90.x`, provides a generic facility for interpolating band structures (and optionally their k-derivatives/velocities) onto a user-specified list of k-points. The k-points are read from an external file (`seedname_geninterp.kpt`). This module is useful for obtaining band energies and velocities at arbitrary points in the Brillouin zone, beyond what is typically calculated for standard band structure plots along high-symmetry lines or on regular meshes.

## Key Components

### Module `w90_geninterp`
- **Description:** The main module for generic k-point interpolation.

### Subroutine `geninterp_main`
- **Description:** This is the primary routine that orchestrates the generic interpolation. It first reads the list of k-points from `seedname_geninterp.kpt`. Then, for each k-point, it constructs the Hamiltonian in the Wannier basis $H_{mn}(\mathbf{k})$ by Fourier transforming the real-space Hamiltonian $H_{mn}(\mathbf{R})$. It diagonalizes $H(\mathbf{k})$ to get the interpolated band energies. If requested (`pw90_geninterp%alsofirstder = .true.`), it also computes the band velocities $\nabla_{\mathbf{k}} E_{n\mathbf{k}}$ using routines from `w90_wan_ham`. The results (k-point coordinates, energies, and optionally velocities) are written to output files. The output can be a single file or one file per MPI process if `pw90_geninterp%single_file` is false.
- **Parameters:**
    - `pw90_geninterp` (type `pw90_geninterp_mod_type`): Contains control parameters like `alsofirstder` (calculate derivatives) and `single_file` (output to one file or multiple).
    - `HH_R` (complex, allocatable): Real-space Hamiltonian matrix elements $\langle \mathbf{0}n | H | \mathbf{R}m \rangle$.
    - `kpt_latt` (real): Coordinates of the original k-mesh (used by `get_HH_R`).
    - `u_matrix`, `v_matrix`: Unitary matrices from wannierisation (if effective model).
    - `eigval`: Original Bloch eigenvalues.
    - `real_lattice`, `scissors_shift`, `mp_grid`, `num_bands`, `num_kpts`, `num_wann`, `num_valence_bands`: System and dimension parameters.
    - `effective_model`, `have_disentangled`: Logical flags.
    - `seedname`, `stdout`, `timer`, `error`, `comm`.
- **Returns:** Creates output file(s) (`seedname_geninterp.dat` or `seedname_geninterp_XXXXX.dat`) containing the interpolated band data.

### Subroutine `internal_write_header`
- **Description:** An internal helper routine to write a standard header to the output data file(s). The header includes a timestamp, the input file comment, and column descriptions (k-point index, kx, ky, kz, Energy, and optionally EnergyDer_x/y/z).
- **Parameters:** `outdat_unit` (output file unit), `commentline` (comment from input k-point file), `pw90_geninterp` (to check `alsofirstder`).

## Important Variables/Constants

- **`pw90_geninterp_mod_type` (from `postw90_types.F90`):**
    - `alsofirstder` (logical): If true, calculate and output band velocities in addition to energies.
    - `single_file` (logical): If true, all MPI processes write to a single output file (root gathers data). If false, each process writes its own file.
- **Input k-point file (`seedname_geninterp.kpt`):**
    - Line 1: A comment line.
    - Line 2: Coordinate type ('crystal', 'frac' for fractional, or 'cart', 'abs' for Cartesian in $1/\text{Ang}$).
    - Line 3: Number of k-points to interpolate (`nkinterp`).
    - Following `nkinterp` lines: `kpoint_index kx ky kz`.

## Usage Examples
This module is invoked by `postw90.x` when `geninterp = .true.` is set in the `seedname.win` input file. The user must also provide a `seedname_geninterp.kpt` file.

**Example `seedname.win` section:**
```
geninterp = .true.
geninterp_alsofirstder = .true.  ! Optional: to get velocities
geninterp_single_file = .true.   ! Optional: to get a single output file in MPI
```

**Example `seedname_geninterp.kpt` file:**
```
# My custom k-points for interpolation
crystal  ! or cart
3
1  0.0  0.0  0.0   ! Gamma point
2  0.5  0.0  0.0   ! X point (example)
3  0.1  0.2  0.3   ! An arbitrary k-point
```
`postw90.x` will then read these k-points and generate `seedname_geninterp.dat` with the interpolated bands (and velocities if requested) at these specific k-points.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `pi`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication (`comms_bcast`, `comms_array_split`, `comms_scatterv`, `comms_gatherv`).
    - `w90_io`: For file I/O (`io_file_unit`, `io_date`) and timing.
    - `w90_types`: For shared data structures.
    - `w90_postw90_types`: For `pw90_geninterp_mod_type`.
    - `w90_utility`: For `utility_diagonalize`, `utility_recip_lattice_base`.
    - `w90_postw90_common`: For Fourier transform routines (`pw90common_fourier_R_to_k`).
    - `w90_get_oper`: To get the real-space Hamiltonian `HH_R`.
    - `w90_wan_ham`: For `wham_get_eig_deleig` to get interpolated energies and k-derivatives.

- **External Libraries:**
    - **LAPACK/BLAS:** Used implicitly via matrix operations (e.g., `utility_diagonalize` which calls `ZHPEVX`).

- **Interactions:**
    - `geninterp_main` is the entry point called by `postw90.x`.
    - Requires MLWFs and the real-space Hamiltonian matrix elements from a prior `wannier90.x` run.
    - Reads a user-supplied list of k-points from `seedname_geninterp.kpt`.
    - Outputs interpolated band energies and, if requested, band velocities to `seedname_geninterp.dat` (or multiple files if `geninterp_single_file` is false and running in parallel).
```
