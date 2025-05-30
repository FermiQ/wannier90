## Overview
The `w90_spin` module, part of `postw90.x`, provides routines for calculating spin-related properties using the Wannier-interpolated Hamiltonian and spin operators. Specifically, it can compute the total spin magnetic moment of the system and the k-dependent expectation values of spin components for each band. These functionalities are essential when `spinors = .true.` is used in the `wannier90.x` calculation.

## Key Components

### Module `w90_spin`
- **Description:** The main module for spin-related calculations in `postw90.x`.

### Subroutine `spin_get_moment`
- **Description:** Calculates the total spin magnetic moment per unit cell. It iterates over a k-mesh (either a full BZ grid or a user-supplied list from `kpoint.dat`), calls `spin_get_moment_k` for each k-point to get the local spin contribution, and then sums (integrates) these contributions, weighted appropriately, to obtain the total spin moment. The result is printed to standard output in Bohr magnetons, including x, y, z components and polar/azimuthal angles.
- **Parameters:**
    - `pw90_spin` (type `pw90_spin_mod_type`): Contains spin calculation parameters (e.g., k-mesh for spin, quantization axis).
    - `fermi_energy_list(:)` (real(dp)): Fermi energy (expects a single value).
    - `HH_R(:, :, :)` (complex(dp)): Real-space Hamiltonian matrix elements.
    - `SS_R(:, :, :, :)` (complex(dp)): Real-space spin operator matrix elements $\langle \mathbf{0}n | \sigma_i | \mathbf{R}m \rangle$.
    - Other parameters include system information, Wannier data, k-mesh details, `u_matrix`, `v_matrix`, etc.
- **Returns:** Prints the calculated spin magnetic moment components to `stdout`.

### Subroutine `spin_get_nk`
- **Description:** Computes the expectation value of the spin projection along a specified quantization axis for each Wannier-interpolated band $n$ at a given k-point: $\langle \psi_{n\mathbf{k}}^{(H)} | \mathbf{S} \cdot \mathbf{\hat{n}} | \psi_{n\mathbf{k}}^{(H)} \rangle$. Here, $\mathbf{S}$ is the vector of Pauli matrices, and $\mathbf{\hat{n}}$ is the unit vector defining the quantization axis (from `pw90_spin%axis_polar` and `pw90_spin%axis_azimuth`). $\psi_{n\mathbf{k}}^{(H)}$ are the eigenstates of the interpolated Wannier Hamiltonian $H_W(\mathbf{k})$.
- **Parameters:**
    - `pw90_spin`: For quantization axis.
    - `HH_R`, `SS_R`: Real-space Hamiltonian and spin matrices.
    - `kpt(3)`: The k-point at which to calculate.
    - `spn_nk(num_wann)` (real(dp), out): Output array of spin expectation values for each band.
    - Other system and Wannier parameters.
- **Returns:** Populates `spn_nk`.

### Private Helper Subroutine `spin_get_moment_k`
- **Description:** Calculates the contribution to the total spin magnetic moment from a single k-point. It first computes the expectation values of the three Pauli matrices $\langle \sigma_x \rangle, \langle \sigma_y \rangle, \langle \sigma_z \rangle$ in the basis of the interpolated Bloch states $\psi_{n\mathbf{k}}^{(H)}$ by calling `spin_get_S`. Then, it sums these expectation values over the occupied states (determined by `ef` and `eig`) to get the k-point contribution to the spin moment.
- **Parameters:** `kpt`, `ef` (Fermi energy), `spn_k(3)` (output k-contribution to spin moment), `HH_R`, `SS_R`, and other system parameters.
- **Returns:** Populates `spn_k`.

### Private Helper Subroutine `spin_get_S`
- **Description:** Computes the expectation values of the three Pauli matrices, $S_x, S_y, S_z$, in the basis of the interpolated Bloch states $\psi_{n\mathbf{k}}^{(H)}$: $S_{\alpha,nm}(\mathbf{k}) = \langle \psi_{n\mathbf{k}}^{(H)} | \sigma_\alpha | \psi_{m\mathbf{k}}^{(H)} \rangle$. It first Fourier transforms the real-space spin matrices $SS_R(:,:,:,is)$ to k-space $SS_k(:,:,is)$, then rotates them into the Hamiltonian eigenbasis using the matrix `UU` (eigenvectors of $H_W(\mathbf{k})$). The diagonal elements $S_{\alpha,nn}(\mathbf{k})$ are the desired expectation values.
- **Parameters:** `kpt`, `S(num_wann, 3)` (output matrix of $\langle \sigma_\alpha \rangle_n$), `HH_R`, `SS_R`, and other system parameters.
- **Returns:** Populates `S`.

## Important Variables/Constants

- **`pw90_spin_mod_type` (from `postw90_types.F90`):**
    - `axis_polar` (real(dp)): Polar angle $\theta$ for spin quantization axis.
    - `axis_azimuth` (real(dp)): Azimuthal angle $\phi$ for spin quantization axis.
    - `kmesh` (type `kmesh_spacing_type`): K-mesh for spin moment integration.
- **`SS_R (num_wann, num_wann, nrpts, 3)` (complex(dp)):** Real-space matrix elements of the Pauli spin operators $\sigma_x, \sigma_y, \sigma_z$ in the Wannier basis.
- **`spn_k(3)` (real(dp)):** Contribution to the spin moment from a single k-point.
- **`spn_all(3)` (real(dp)):** Total spin moment integrated over the BZ.
- **`spn_nk(num_wann)` (real(dp)):** Expectation value of $\mathbf{S} \cdot \mathbf{\hat{n}}$ for each band at a given k-point.

## Usage Examples
This module is invoked by `postw90.x` when `spin_moment = .true.` or when `kpath_bands_colour = 'spin'` or `kslice_fermi_lines_colour = 'spin'` is set in the `seedname.win` input file.

**Example `seedname.win` section for total spin moment:**
```
spin_moment = .true.
spin_axis_polar = 0.0    ! In degrees, angle from z-axis
spin_axis_azimuth = 0.0  ! In degrees, angle from x-axis in xy-plane
spin_kmesh = 20 20 20    ! K-mesh for BZ integration
fermi_energy = 0.0       ! Fermi energy in eV
spinors = .true.         ! Must be true in wannier90.x run
```

**Example for spin-colored bands:**
```
kpath = .true.
kpath_task = 'bands'
kpath_bands_colour = 'spin'
spin_axis_polar = 0.0
spin_axis_azimuth = 0.0
spinors = .true.
begin kpoint_path
  ...
end kpoint_path
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `pi`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication (`comms_reduce`, `mpirank`, `mpisize`).
    - `w90_io`: For file I/O (`io_file_unit`) and timing.
    - `w90_types`: For shared data structures (`print_output_type`, `wannier_data_type`, etc.).
    - `w90_postw90_types`: For `postw90`-specific types (`pw90_spin_mod_type`, etc.).
    - `w90_utility`: For `utility_diagonalize`, `utility_rotate_diag`.
    - `w90_get_oper`: To obtain real-space Hamiltonian `HH_R` and spin matrices `SS_R`.
    - `w90_postw90_common`: For Fourier transform routines (`pw90common_fourier_R_to_k`) and occupation calculation (`pw90common_get_occ`).
    - `w90_wan_ham`: (Indirectly) For routines that provide `eigval` and `UU` (eigenvectors of $H_W(\mathbf{k})$).

- **External Libraries:**
    - **LAPACK/BLAS:** Used implicitly for matrix diagonalizations (`utility_diagonalize` calls `ZHPEVX`).

- **Interactions:**
    - `spin_get_moment` and `spin_get_nk` are the main public interfaces.
    - Requires prior computation of MLWFs, $H(\mathbf{R})$, and $\sigma_i(\mathbf{R})$ (from `wannier90.x` and `w90_get_oper`).
    - Spin matrix elements are read from `seedname.spn`.
    - The calculated spin moment is printed to `stdout`. The `spn_nk` values are used by `w90_kpath` or `w90_kslice` for color-coding plots.
```
