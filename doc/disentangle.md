## Overview
The `w90_disentangle` module is a core component of Wannier90 responsible for extracting an optimal, smooth, and connected subspace of `num_wann` Bloch-like states from a larger set of `num_bands` Bloch states, particularly when `num_bands > num_wann` (entangled bands). This process is crucial for constructing well-localized Wannier functions. The disentanglement procedure aims to minimize a gauge-invariant spread functional, $\Omega_I$, which involves an iterative refinement of the unitary matrices `U_matrix_opt` that define the selected subspace at each k-point. The module handles energy windows (outer and inner/frozen) to guide the selection of states.

## Key Components

### Module `w90_disentangle`
- **Description:** Contains all subroutines and logic for the disentanglement procedure.

### Subroutine `dis_main`
- **Description:** The main driver routine for the disentanglement process. It orchestrates the definition of energy windows, projection of initial trial orbitals, iterative optimization of the subspace to minimize $\Omega_I$, and finalization of the `U_matrix_opt` and `M_matrix`.
- **Parameters:**
    - `dis_control`: Control parameters for disentanglement (iterations, tolerance, mixing).
    - `dis_spheres`: Parameters for k-space spheres where disentanglement might be restricted.
    - `dis_manifold`: Information about the selected manifold (windows, dimensions).
    - `kmesh_info`: K-mesh information (neighbors, weights).
    - `kpt_latt`: K-point coordinates.
    - `sitesym`: Symmetry information.
    - `print_output`: Controls printing.
    - `a_matrix`: Projections $\langle \psi_m | g_n \rangle$.
    - `m_matrix`, `m_matrix_local`, `m_matrix_orig`, `m_matrix_orig_local`: Overlap matrices $M_{mn}^{(\mathbf{k,b})}$. `_orig` are initial full-dimension overlaps, `_local` are for current MPI task.
    - `u_matrix`: Unitary matrices for wannierisation (initialized here if `optimisation <= 0`).
    - `u_matrix_opt`: Unitary matrices defining the optimal `num_wann` dimensional subspace from the original `num_bands`. This is the primary output of the disentanglement.
    - `eigval`: Bloch eigenvalues.
    - `real_lattice`: Real-space lattice vectors.
    - `omega_invariant`: Output value of the minimized $\Omega_I$.
    - `num_bands`, `num_kpts`, `num_wann`: Dimensions.
    - `optimisation`: Optimization level.
    - `gamma_only`: Logical for Gamma-point only calculation.
    - `lsitesymmetry`: Logical for using site symmetry.
    - `stdout`, `timer`, `error`, `comm`.
- **Returns:** Populates `u_matrix_opt`, `omega_invariant`, and modifies `m_matrix` (or `m_matrix_local`) to be in the optimal subspace.

### Subroutine `dis_windows`
- **Description:** Defines the outer and inner (frozen) energy windows for disentanglement. It determines which Bloch states fall within these windows at each k-point.
- **Parameters:** `dis_spheres`, `dis_manifold` (output: `ndimwin`, `lwindow`), `eigval_opt` (input original eigenvalues, output slimmed and shifted eigenvalues within window), `kpt_latt`, `recip_lattice`, `indxfroz`, `indxnfroz`, `ndimfroz`, `nfirstwin` (output window indices), `iprint`, `num_bands`, `num_kpts`, `num_wann`, `timing_level`, `lfrozen`, `linner` (output logicals), `on_root`, `stdout`, `timer`, `error`, `comm`.
- **Returns:** Modifies `dis_manifold` and `eigval_opt`, sets up window index arrays and logical flags.

### Subroutine `dis_project`
- **Description:** Constructs the initial projection of the trial localized orbitals (often Gaussians, represented by `a_matrix`) onto the Bloch states within the outer window. It then orthogonalizes/unitarizes this projection using Singular Value Decomposition (SVD) to form an initial `u_matrix_opt`.
- **Parameters:** `a_matrix` (input, gets modified), `u_matrix_opt` (output), `ndimwin`, `nfirstwin`, `num_bands`, `num_kpts`, `num_wann`, `timing_level`, `on_root`, `timer`, `error`, `stdout`, `comm`.
- **Returns:** Initializes `u_matrix_opt`.

### Subroutine `dis_proj_froz`
- **Description:** Modifies the `u_matrix_opt` obtained from `dis_project` to explicitly include states from a frozen (inner) window. It computes the leading eigenvectors of $Q_{froz} P_s Q_{froz}$, where $P_s$ projects onto the subspace of projected Gaussians and $Q_{froz} = 1 - P_{froz}$ projects away from the frozen states.
- **Parameters:** `u_matrix_opt` (input/output), `indxfroz`, `ndimfroz`, `ndimwin`, `iprint`, `num_bands`, `num_kpts`, `num_wann`, `timing_level`, `lfrozen`, `on_root`, `timer`, `error`, `stdout`, `comm`.
- **Returns:** Modifies `u_matrix_opt` to incorporate frozen states correctly.

### Subroutine `dis_extract`
- **Description:** Performs the iterative minimization of the spread functional $\Omega_I$ to find the optimal `num_wann`-dimensional subspace at each k-point. It computes the "Z-matrix" (Eq. 21 of Marzari & Vanderbilt, PRB 56, 12847 (1997)), diagonalizes it, and uses its eigenvectors to update `u_matrix_opt`. This is the main iterative loop.
- **Parameters:** Similar to `dis_main`, including `dis_control` for iteration parameters, `m_matrix_orig_local` (slimmed overlaps), and outputs `omega_invariant`.
- **Returns:** Refines `u_matrix_opt` to minimize $\Omega_I$, calculates `omega_invariant`, and updates `eigval_opt` to be eigenvalues of the Hamiltonian within the final optimal subspace.

### Subroutine `dis_extract_gamma`
- **Description:** A specialized version of `dis_extract` for Gamma-point only calculations, where certain simplifications or different numerical approaches (e.g., using real arithmetic for Z-matrix diagonalization) can be applied.
- **Parameters & Returns:** Similar to `dis_extract`.

### Internal Subroutines (e.g., `internal_check_orthonorm`, `internal_slim_m`, `internal_find_u`, `internal_zmatrix`, `internal_test_convergence`)
- **Description:** These are helper routines called by the main `dis_` subroutines to perform specific tasks like checking orthonormality of `U_matrix_opt`, reducing the dimension of overlap matrices to the window size, finding an initial guess for the `u_matrix` (for wannierisation), computing the Z-matrix, and checking convergence of $\Omega_I$.

## Important Variables/Constants

- **`u_matrix_opt (num_bands, num_wann, num_kpts)` (complex(kind=dp)):** The primary output. This matrix defines the transformation from the original `num_bands` Bloch states (within the outer window) to the `num_wann` states forming the optimally connected subspace.
- **`dis_manifold%ndimwin(num_kpts)` (integer):** Number of Bloch states within the outer energy window at each k-point.
- **`dis_manifold%win_min`, `dis_manifold%win_max` (real(kind=dp)):** Energy boundaries of the outer window.
- **`dis_manifold%froz_min`, `dis_manifold%froz_max` (real(kind=dp)):** Energy boundaries of the inner (frozen) window.
- **`lfrozen(num_bands, num_kpts)` (logical):** True if a state (indexed within the outer window) is part of the frozen window.
- **`linner` (logical):** True if a frozen window is active.
- **`omega_invariant` (real(kind=dp)):** The value of the gauge-invariant part of the spread functional, $\Omega_I$, after minimization.
- **`czmat_in / rzmat_in (num_bands, num_bands, num_kpts)` (complex/real):** The Z-matrix (or its real version for Gamma-point) used in the iterative minimization.
- **`dis_control%num_iter` (integer):** Maximum number of iterations for $\Omega_I$ minimization.
- **`dis_control%conv_tol` (real(kind=dp)):** Convergence tolerance for $\Omega_I$.
- **`dis_control%mix_ratio` (real(kind=dp)):** Mixing ratio for updating the Z-matrix between iterations.

## Usage Examples
The `dis_main` subroutine is called from the main Wannier90 program (`wannier_prog.F90`) when `num_bands > num_wann`, indicating that disentanglement is required.

```fortran
! In wannier_prog.F90, after overlaps are read/calculated:
if (disentanglement) then ! disentanglement is (num_bands > num_wann)
  call dis_main(dis_control, dis_spheres, dis_manifold, kmesh_info, kpt_latt, sitesym, &
                print_output, a_matrix, m_matrix, m_matrix_local, m_matrix_orig, &
                m_matrix_orig_local, u_matrix, u_matrix_opt, eigval, real_lattice, &
                omega_invariant, num_bands, num_kpts, num_wann, optimisation, gamma_only, &
                lsitesymmetry, stdout, timer, error, comm)
  if (allocated(error)) call prterr(error, stdout, stderr, comm)
  have_disentangled = .true.
end if
```
The user controls the disentanglement process through parameters in the `seedname.win` input file, such as `dis_win_min`, `dis_win_max`, `dis_froz_min`, `dis_froz_max`, `dis_num_iter`, `dis_conv_tol`.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_comms`: For MPI communication (broadcast, gather, allreduce).
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_1`, and various `eps*` tolerance values.
    - `w90_io`: For timing (`io_stopwatch_start`, `io_stopwatch_stop`) and file I/O (though less prominent in `disentangle` itself).
    - `w90_error`: For error handling and reporting.
    - `w90_types`: Defines derived types like `dis_manifold_type`, `kmesh_info_type`, `print_output_type`, `timer_list_type`.
    - `w90_wannier90_types`: Defines `dis_control_type`, `dis_spheres_type`, `sitesym_type`.
    - `w90_sitesym`: If symmetry is enabled (`lsitesymmetry = .true.`), routines from this module are called to symmetrize `U_matrix_opt` and the Z-matrix.
    - `w90_utility`: For `utility_recip_lattice_base`.

- **External Libraries:**
    - **LAPACK/BLAS:** Heavily used for linear algebra operations:
        - `zgesvd` (complex SVD) in `dis_project` and `internal_find_u`.
        - `dgesvd` (real SVD) in `internal_find_u_gamma`.
        - `zhpevx` (complex Hermitian packed eigenvalue problem solver) in `dis_proj_froz` and `dis_extract`.
        - `dspevx` (real symmetric packed eigenvalue problem solver) in `dis_extract_gamma`.
        - `zgemm` (complex matrix-matrix multiplication) and `dgemm` (real matrix-matrix multiplication) are used in several places.

- **Interactions:**
    - Receives initial Bloch eigenvalues (`eigval`), overlap matrices (`m_matrix_orig`, `m_matrix_orig_local`), and projection matrices (`a_matrix`) from earlier stages of the Wannier90 calculation (typically from `w90_overlap` or read from files).
    - Receives k-mesh information (`kmesh_info`) from `w90_kmesh`.
    - The primary output, `u_matrix_opt`, defines the selected subspace and is crucial for the subsequent wannierisation step (`w90_wannierise`) where the spread $\Omega$ is minimized.
    - The modified `m_matrix` (transformed into the optimal subspace) is also passed to `w90_wannierise`.
    - The `eigval_opt` array is updated to contain the eigenvalues of the Hamiltonian projected onto the final optimal subspace, which can be used for plotting disentangled band structures.
```
