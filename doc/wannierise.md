## Overview
The `w90_wannierise` module contains the core routines for the iterative minimization of the spread functional to obtain Maximally Localised Wannier Functions (MLWFs). This is the heart of the wannierisation procedure. It implements the algorithm described by Marzari and Vanderbilt (PRB 56, 12847 (1997)) and Souza, Marzari, and Vanderbilt (PRB 65, 035109 (2001)). The module provides two main routines: `wann_main` for the general case and `wann_main_gamma` for Gamma-point only calculations, which can sometimes allow for simplifications.

## Key Components

### Module `w90_wannierise`
- **Description:** The main module for the MLWF minimization algorithm.

### Type `localisation_vars_type`
- **Description:** A derived type to store the different components of the total spread functional.
- **Fields:**
    - `om_i` (real(kind=dp)): Gauge-invariant part of the spread, $\Omega_I$.
    - `om_d` (real(kind=dp)): Diagonal part of the gauge-dependent spread, $\Omega_D$.
    - `om_od` (real(kind=dp)): Off-diagonal part of the gauge-dependent spread, $\Omega_{OD}$.
    - `om_tot` (real(kind=dp)): Total spread, $\Omega = \Omega_I + \Omega_D + \Omega_{OD}$.
    - `om_iod` (real(kind=dp)): Combined $\Omega_I + \Omega_{OD}$ term, used in selective localization.
    - `om_nu` (real(kind=dp)): Lagrange multiplier term due to constrained centers.

### Subroutine `wann_main`
- **Description:** The main driver for the wannierisation procedure. It iteratively refines the unitary matrices $U_{mn}^{(\mathbf{k})}$ to minimize the total spread $\Omega$. This involves calculating the gradient of the spread, determining a search direction (e.g., steepest descent or conjugate gradients), performing a line search (or using a fixed step), and updating the $U$ matrices. It also handles guiding centers and constrained localization if requested.
- **Parameters:** An extensive list, including `wann_control` (for iteration parameters), `omega` (output spreads), `m_matrix` (overlaps), `u_matrix` (input initial guess, output final $U$), `kmesh_info`, `num_wann`, `num_kpts`, `wannier_data` (output centers and spreads), and various other system and control parameters.
- **Returns:** Populates `wannier_data` with final centers and spreads, `omega` with final spread components, and `u_matrix` with the converged unitary transformation.

### Subroutine `wann_main_gamma`
- **Description:** A specialized version of `wann_main` for Gamma-point only calculations. It may employ optimizations specific to this case, such as working with real matrices if applicable.
- **Parameters & Returns:** Similar to `wann_main`.

### Internal Subroutines
- **`internal_test_convergence` / `internal_test_convergence_gamma`:** Checks if the wannierisation has converged by comparing the change in spread over a window of iterations (`wann_control%conv_window`) against a tolerance (`wann_control%conv_tol`). Handles logic for adding noise if convergence stalls.
- **`internal_random_noise`:** Adds random noise to the search direction to help escape local minima if convergence stalls.
- **`precond_search_direction`:** Calculates a preconditioned search direction for the conjugate gradients algorithm.
- **`internal_search_direction`:** Calculates the search direction for spread minimization, typically using a conjugate gradients (Fletcher-Reeves) or steepest descent method.
- **`internal_optimal_step`:** Performs a parabolic line search to find the optimal step length along the current search direction.
- **`internal_new_u_and_m` / `internal_new_u_and_m_gamma`:** Updates the unitary matrices `u_matrix` (and `u_matrix_loc` for MPI) and subsequently the overlap matrices `m_matrix` (and `m_matrix_loc`) using the determined step and search direction. It involves matrix exponentiation of the (anti-Hermitian) gradient step.
- **`wann_phases`:** Determines the branch cut (integer $m_{\mathbf{k}\mathbf{b}}^{(n)}$ in $M_{nn}^{(\mathbf{k,b})} = |M_{nn}^{(\mathbf{k,b})}| e^{i \phi_{\mathbf{k}\mathbf{b}}^{(n)}} e^{-i \mathbf{b} \cdot \mathbf{r}_n^{(0)}} e^{i 2\pi m_{\mathbf{k}\mathbf{b}}^{(n)}}$) for the logarithm of the diagonal elements of $M_{nn}^{(\mathbf{k,b})}$, often using guiding centers to ensure smooth phases. Populates `csheet` (phase factors $e^{i 2\pi m}$) and `sheet` (the $2\pi m$ values).
- **`wann_omega` / `wann_omega_gamma`:** Calculates the different components of the spread functional ($\Omega_I, \Omega_D, \Omega_{OD}, \Omega_{TOT}$) and the Wannier function centers (`rave`) and quadratic spreads (`r2ave`, `rave2`).
- **`wann_domega`:** Calculates the gradient of the spread functional with respect to rotations of the $U$ matrices. This gradient (`cdodq_loc`) is crucial for the minimization algorithms.
- **`wann_spread_copy`:** Utility to copy spread components from one `localisation_vars_type` to another.
- **`wann_calc_projection`:** Calculates and writes the projection of each Wannier function onto the original Bloch bands within the outer window (if disentanglement was used).
- **`wann_write_xyz`:** Writes the final Wannier function centers to a `.xyz` file.
- **`wann_write_vdw_data`:** Writes data (centers, spreads, occupations) for Van der Waals C6 coefficient post-processing.
- **`wann_check_unitarity`:** Checks the unitarity of the final $U$ matrices.
- **`wann_write_r2mn`:** Writes matrix elements $\langle \mathbf{r}^2 \rangle_{mn}$ to `seedname.r2mn`.
- **`wann_svd_omega_i`:** Performs a singular value decomposition analysis of $M_{mn}^{(\mathbf{k,b})}$ to provide alternative expressions for $\Omega_I$.

## Important Variables/Constants

- **`u_matrix (num_wann, num_wann, num_kpts)` (complex(kind=dp)):** The unitary matrices that rotate the Bloch states (or the optimal subspace from disentanglement) into the Wannier gauge. This is the primary variable being optimized.
- **`m_matrix (num_wann, num_wann, nntot, num_kpts)` (complex(kind=dp)):** Overlap matrices $M_{mn}^{(\mathbf{k,b})}$ in the current basis defined by `u_matrix`.
- **`wann_control_type` (Derived Type):** Contains control parameters for the minimization:
    - `num_iter`: Max iterations.
    - `conv_tol`: Convergence tolerance for spread.
    - `num_cg_steps`: Conjugate gradient steps before reset.
    - `guiding_centres%enable`: Whether to use guiding centers.
    - `fixed_step`, `trial_step`: Step length controls.
- **`wann_omega_type` (Derived Type):** Stores the spread components.
    - `total`, `invariant`, `tilde`.
- **`wannier_data_type` (Derived Type):** Stores final results.
    - `centres`, `spreads`.
- **`cdodq_loc (num_wann, num_wann, num_kpts_local)` (complex(kind=dp)):** Gradient of the spread functional for the local k-points.
- **`csheet (num_wann, nntot, num_kpts)` (complex(kind=dp)):** Phase factors $e^{i 2\pi m_{\mathbf{k}\mathbf{b}}^{(n)}}$ for branch choice.
- **`rave (3, num_wann)` (real(kind=dp)):** Wannier function centers.
- **`r2ave (num_wann)` (real(kind=dp)):** $\langle r^2 \rangle_n$.
- **`rave2 (num_wann)` (real(kind=dp)):** $\langle \mathbf{r} \rangle_n^2$.

## Usage Examples
The `wann_main` (or `wann_main_gamma`) subroutine is called by the main program (`wannier_prog.F90`) after the initial setup (parameter reading, k-mesh, overlaps, and potentially disentanglement) is complete.

```fortran
! In wannier_prog.F90
if (gamma_only) then
  call wann_main_gamma(atoms, dis_window, ..., seedname, stdout, timer, error, comm)
else
  call wann_main(atoms, dis_window, ..., seedname, stdout, timer, error, comm)
end if
if (allocated(error)) call prterr(error, stdout, stderr, comm)
```
The behavior is controlled by parameters in `seedname.win` that are parsed into the `wann_control_type` variable.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_1`, `cmplx_i`, `twopi`, `eps*`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication (`comms_allreduce`, `comms_gatherv`, `comms_bcast`, `comms_scatterv`).
    - `w90_io`: For timing and file I/O.
    - `w90_types`: Defines shared data structures like `kmesh_info_type`, `wannier_data_type`.
    - `w90_wannier90_types`: Defines `wann_control_type`, `wann_omega_type`, etc.
    - `w90_wannier90_readwrite`: For writing checkpoint files.
    - `w90_utility`: For matrix operations (`utility_zgemm`), coordinate transformations.
    - `w90_sitesym`: If `lsitesymmetry` is true, uses `sitesym_symmetrize_gradient`.
    - `w90_hamiltonian`: If `output_file%write_hr_diag` is true, it calls Hamiltonian routines.

- **External Libraries:**
    - **LAPACK/BLAS:** `zheev` (complex Hermitian eigenvalue solver) and `zgees` (Schur decomposition) are used in `internal_new_u_and_m` for matrix exponentiation. `zgesvd` (SVD) is used in `wann_svd_omega_i`. `zdotc` is used for complex dot products.

- **Interactions:**
    - Receives initial $U$ matrices (from `w90_overlap` or `w90_disentangle`) and $M$ matrices (from `w90_overlap`, possibly modified by `w90_disentangle`).
    - Iteratively updates $U$ and $M$ to minimize the spread.
    - Outputs final Wannier function centers and spreads (`wannier_data`).
    - Can call routines from `w90_hamiltonian` and `w90_plot` for specific outputs.
    - Writes checkpoint files via `w90_wannier90_readwrite_write_chkpt`.
```
