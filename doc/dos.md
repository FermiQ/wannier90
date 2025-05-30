## Overview
The `w90_dos` module, part of `postw90.x`, is responsible for calculating the electronic Density of States (DOS). It can compute the total DOS and, if spin polarization or spinor calculations are involved, can project the DOS onto spin-up and spin-down components. Furthermore, it allows for projection of the DOS onto selected Wannier functions. The module supports various smearing schemes, including adaptive smearing based on band velocities (as described in PRB 75, 195121 (2007) by Yates, Wang, Vanderbilt, and Souza - YWVS07).

## Key Components

### Module `w90_dos`
- **Description:** The main module for all DOS calculation functionalities.

### Subroutine `dos_main`
- **Description:** This is the primary driver routine for DOS calculations. It sets up the energy grid for the DOS, iterates over a k-mesh (either a regular Monkhorst-Pack grid or a user-supplied list from `kpoint.dat`), and calls `dos_get_k` for each k-point to get its contribution. The results are then summed (reduced in MPI runs) and written to `seedname-dos.dat`.
- **Parameters:**
    - `pw90_dos` (type `pw90_dos_mod_type`): Contains DOS-specific input parameters (energy range, k-mesh, smearing, projections).
    - `pw90_berry` (type `pw90_berry_mod_type`): Used if `wanint_kpoint_file` is true to get k-point distribution.
    - Other parameters are similar to those in `w90_berry_main` or `w90_boltzwann_main`, providing system information, Wannier function data, Hamiltonian and matrix elements (`HH_R`, `SS_R` if spin-decomposed), `eigval`, `real_lattice`, etc.
- **Returns:** Generates the `seedname-dos.dat` output file.

### Subroutine `dos_get_k`
- **Description:** Calculates the contribution to the DOS from a single k-point. It takes the eigenvalues $E_{n\mathbf{k}}$ at that k-point and, for each band, adds a smeared delta function $\delta(E - E_{n\mathbf{k}})$ to the appropriate energy bins in `dos_k`. It handles different smearing types (fixed width or adaptive) and can project onto specified Wannier functions using the `UU` matrix (eigenvectors of $H(\mathbf{k})$ in Wannier basis). It also handles spin decomposition if `spin_decomp` is true, using spin expectation values `spn_nk`.
- **Parameters:**
    - `num_elec_per_state` (integer): Number of electrons per state (1 or 2).
    - `kpt(3)` (real(dp)): Current k-point coordinates.
    - `EnergyArray(:)` (real(dp)): Energy grid for DOS.
    - `eig_k(:)` (real(dp)): Eigenvalues at `kpt`.
    - `dos_k(:, :)` (real(dp), out): DOS contribution from this k-point.
    - `smearing` (type `pw90_smearing_type`): Smearing parameters.
    - `levelspacing_k(:)` (real(dp), optional): Band-resolved level spacing, required for adaptive smearing.
    - `UU(:, :)` (complex(dp), optional): Unitary matrix diagonalizing $H(\mathbf{k})$, needed for projections.
    - Other parameters include system info and Wannier data.
- **Returns:** Populates `dos_k` with the DOS contribution from the given k-point.

### Subroutine `dos_get_levelspacing`
- **Description:** Calculates the "level spacing" for each band at a given k-point. This quantity is related to the magnitude of the band velocity $|\nabla_{\mathbf{k}} E_{n\mathbf{k}}|$ multiplied by a characteristic k-space distance $\Delta k$ (derived from `kmesh`). It's used as the effective width for adaptive smearing.
- **Parameters:**
    - `del_eig(:, :)` (real(dp)): Band velocities $\nabla_{\mathbf{k}} E_{n\mathbf{k}}$.
    - `kmesh(3)` (integer): Dimensions of the k-mesh used for interpolation.
    - `levelspacing(num_wann)` (real(dp), out): Output array of level spacings.
    - `recip_lattice(3,3)` (real(dp)): Reciprocal lattice vectors.
- **Returns:** Populates `levelspacing`.

## Important Variables/Constants

- **`pw90_dos_mod_type` (from `postw90_types.F90`):**
    - `kmesh%mesh(3)`: Dimensions of the k-mesh for DOS interpolation.
    - `energy_min`, `energy_max`, `energy_step`: Defines the energy grid for DOS.
    - `smearing%type_index`: Integer code for smearing type (Gaussian, Methfessel-Paxton, Cold Smearing).
    - `smearing%fixed_width`: Smearing width for fixed smearing (in eV).
    - `smearing%use_adaptive`: Logical flag for adaptive smearing.
    - `smearing%adaptive_prefactor`: Prefactor for adaptive smearing width.
    - `smearing%adaptive_max_width`: Maximum width for adaptive smearing.
    - `project(:)`: Array of Wannier function indices for projected DOS. If not allocated or size is `num_wann`, total DOS is computed.
    - `num_project`: Number of WFs to project onto.
- **`smearing_cutoff` (from `w90_constants.F90`):** Cutoff for smearing functions (in units of smearing width).
- **`min_smearing_binwidth_ratio` (from `w90_constants.F90`):** If smearing width / energy bin width is less than this, no smearing is applied (direct binning).

## Usage Examples
This module is invoked by `postw90.x` when `dos = .true.` is set in the `seedname.win` input file.

**Example `seedname.win` section for DOS:**
```
dos = .true.
dos_kmesh = 50 50 50     ! Dense k-mesh for DOS interpolation
dos_energy_min = -10.0
dos_energy_max = 10.0
dos_energy_step = 0.01
dos_smr_type = 'adaptive' ! or 'gaussian', 'm-p1', etc.
dos_adpt_smr_fac = 1.0    ! Prefactor for adaptive smearing
dos_adpt_smr_max = 0.5    ! Max adaptive smearing width in eV
! dos_smr_fixed_en_width = 0.1 ! For fixed smearing

! To project onto specific Wannier functions (e.g., 1 and 3):
! dos_project = 1 3
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `smearing_cutoff`, `min_smearing_binwidth_ratio`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication (`comms_reduce`, `mpirank`, `mpisize`).
    - `w90_io`: For file I/O and timing.
    - `w90_types`: For shared data structures.
    - `w90_postw90_types`: For `postw90`-specific types like `pw90_dos_mod_type`.
    - `w90_utility`: For `utility_w0gauss` (smearing function), `utility_diagonalize`, `utility_recip_lattice_base`.
    - `w90_get_oper`: To get real-space Hamiltonian (`HH_R`) and spin matrices (`SS_R`).
    - `w90_postw90_common`: For Fourier transform routines and `pw90common_kmesh_spacing`.
    - `w90_wan_ham`: For `wham_get_eig_deleig` to get band energies and their k-derivatives.
    - `w90_spin`: If `spin_decomp` is true, uses `spin_get_nk` for spin expectation values.
    - `w90_readwrite`: For `w90_readwrite_get_smearing_type`.

- **External Libraries:**
    - **LAPACK/BLAS:** Used implicitly via matrix operations (e.g., `utility_diagonalize` which calls `ZHPEVX`).

- **Interactions:**
    - `dos_main` is the entry point called by `postw90.x`.
    - Requires MLWFs and real-space Hamiltonian matrix elements from a prior `wannier90.x` run.
    - Interpolates band energies (and velocities for adaptive smearing) onto a dense k-mesh specified by `dos_kmesh`.
    - Outputs the DOS to `seedname-dos.dat`. If `pw90_boltzwann%calc_also_dos` is true from the BoltzWann module, DOS is written to `seedname_boltzdos.dat` instead (or additionally).
```
