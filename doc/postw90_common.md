## Overview
The `w90_postw90_common` module provides a set of shared utility routines and data distribution mechanisms specifically for the `postw90.x` executable. These routines are essential for setting up Wannier interpolation calculations for various physical properties. Key functionalities include Fourier transforming operators from real space to k-space, calculating k-mesh spacing, determining electronic occupations, and distributing parameters and data read by the root MPI process to all other processes. It also includes the Wigner-Seitz cell construction routine used by `postw90.x`.

## Key Components

### Module `w90_postw90_common`
- **Description:** Contains common utility routines for `postw90.x`.

### Fourier Transform Subroutines
- **`pw90common_fourier_R_to_k(ws_region, wannier_data, ...)`:** Performs a Fourier transform of a generic operator `OO_R` from real space (R-space) to k-space: $O(\mathbf{k}) = \sum_{\mathbf{R}} e^{i\mathbf{k}\cdot\mathbf{R}} O(\mathbf{R})$. It can optionally use Wigner-Seitz distance logic via `ws_distance` to ensure the correct $\mathbf{R}$ vectors (shortest ones) are used.
- **`pw90common_fourier_R_to_k_new(ws_region, wannier_data, ..., OO, OO_dx, OO_dy, OO_dz)`:** A newer version that can simultaneously compute $O(\mathbf{k})$ and its first k-derivatives (related to $iR_\alpha O(\mathbf{R})$).
- **`pw90common_fourier_R_to_k_new_second_d(kpt, OO_R, ..., OO, OO_da, OO_dadb)`:** Computes $O(\mathbf{k})$, its first derivative terms $iR_\alpha O(\mathbf{R})$, and second derivative terms $-R_\alpha R_\beta O(\mathbf{R})$.
- **`pw90common_fourier_R_to_k_new_second_d_TB_conv(...)`:** Similar to the above but includes Wannier center shifts in the phase factor (TB convention): $e^{i\mathbf{k}\cdot(\mathbf{R} + \tau_j - \tau_i)}$.
- **`pw90common_fourier_R_to_k_vec(ws_region, wannier_data, ..., OO_true, OO_pseudo)`:** Fourier transforms a vector operator. Can compute both true vector and pseudovector (axial vector) transformations.
- **`pw90common_fourier_R_to_k_vec_dadb(...)` and `pw90common_fourier_R_to_k_vec_dadb_TB_conv(...)`:** Similar to `_new_second_d` but for vector operators, including Wannier center shifts in the TB convention version.

### Initialization and Data Distribution Subroutines
- **`pw90common_wanint_setup(...)`:** Sets up data structures needed for Wannier interpolation, primarily by calling `wignerseitz` to determine the Wigner-Seitz supercell vectors (`wigner_seitz%irvec`, `wigner_seitz%ndegen`, etc.). If `effective_model` is true, `nrpts` is read from `seedname_HH_R.dat`.
- **`pw90common_wanint_get_kpoint_file(kpoint_dist, error, comm)`:** Reads a list of k-points from an external file named `kpoint.dat` if `wanint_kpoint_file` is true. This allows for calculations on arbitrary k-points.
- **`pw90common_wanint_w90_wannier90_readwrite_dist(...)`:** Distributes a large set of parameters (read by `w90_postw90_readwrite_read` on the root node) to all MPI processes. This is a specific distribution routine for parameters defined in `w90_wannier90_types` and `w90_types`.
- **`pw90common_wanint_data_dist(...)`:** Distributes data read from the checkpoint file (like $U$ matrices and Wannier centers) and computes `v_matrix = u_matrix_opt * u_matrix`.

### Other Utilities
- **`pw90common_get_occ(ef, eig, occ, num_wann)`:** Calculates the electronic occupation numbers `occ` for a set of eigenvalues `eig` given a Fermi energy `ef`, assuming a step function (T=0K).
- **`kmesh_spacing_singleinteger(num_points, recip_lattice)` / `kmesh_spacing_mesh(mesh, recip_lattice)` (Interface `pw90common_kmesh_spacing`):** Calculates a characteristic k-mesh spacing $\Delta k$, typically the largest spacing along the primitive reciprocal lattice vector directions. Used for adaptive smearing width determination.
- **`wignerseitz(...)`:** Determines the Wigner-Seitz supercell lattice vectors $\mathbf{R}$ and their degeneracies. This is a copy/adaptation of the `hamiltonian_wigner_seitz` routine.

## Important Variables/Constants
This module primarily provides subroutines. It operates on data structures defined in `w90_types`, `w90_wannier90_types`, and `w90_postw90_types`.

## Usage Examples
These routines are mainly called internally by `postw90.x` and its various computational modules (e.g., `w90_berry`, `w90_dos`).

**Setting up for interpolation:**
```fortran
! In postw90.F90 (on root node initially, then data is broadcast)
call pw90common_wanint_setup(num_wann, verbose, real_lattice, mp_grid, &
                             effective_model, ws_vec, stdout, seedname, timer, error, comm)
! ...
! After reading checkpoint data on root:
call pw90common_wanint_data_dist(num_wann, num_kpts, num_bands, u_matrix_opt, u_matrix, &
                                 dis_window, wann_data, scissors_shift, v_matrix, &
                                 num_valence_bands, have_disentangled, error, comm)
```

**Fourier transforming an operator:**
```fortran
! In a module like w90_berry.F90
call pw90common_fourier_R_to_k_new(ws_region, wannier_data, ws_distance, wigner_seitz, &
                                   HH_R, kpt, real_lattice, mp_grid, num_wann, error, comm, &
                                   OO=H_k, OO_dx=delH_k_dx, ...)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_i`, `twopi`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication (`comms_bcast`, `comms_reduce`, etc.).
    - `w90_io`: For `io_file_unit`, timing functions.
    - `w90_types`, `w90_wannier90_types`, `w90_postw90_types`: For derived data type definitions.
    - `w90_utility`: For `utility_inv3`, `utility_metric`, coordinate transformations.
    - `w90_ws_distance`: For `ws_translate_dist` if Wigner-Seitz shifted distances are used.

- **External Libraries:** None directly.

- **Interactions:**
    - This module is central to `postw90.x`, providing foundational routines for setting up interpolation calculations.
    - The Fourier transform routines are key to converting real-space Wannier Hamiltonians and other operators into their k-space representations needed for band structure interpolation, Berry phase calculations, etc.
    - The data distribution routines are essential for initializing parallel `postw90.x` runs.
    - The Wigner-Seitz cell construction defines the real-space lattice vectors $\mathbf{R}$ over which sums are performed in Fourier transforms.
```
