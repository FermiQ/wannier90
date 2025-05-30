## Overview
The `w90_sitesym` module implements routines to incorporate site symmetry into the Wannier function localization procedure, as described by R. Sakuma in Phys Rev B 87, 235109 (2013). This allows for the construction of symmetry-adapted Wannier functions (SAWFs). The module handles reading symmetry operation data, symmetrizing various matrices (U-matrices from disentanglement and wannierisation, Z-matrix, gradients), and applying symmetry operations to relate quantities at different k-points.

## Key Components

### Module `w90_sitesym`
- **Description:** The main module containing all subroutines and data related to site symmetry operations.

### Subroutine `sitesym_read`
- **Description:** Reads symmetry-related information from a file named `seedname.dmn`. This file contains information about irreducible k-points, symmetry operations, and the D-matrices (representation matrices) for both Bloch bands and Wannier functions.
- **Parameters:**
    - `sitesym` (type(sitesym_type), intent(inout)): The derived type to be populated with symmetry data.
    - `num_bands`, `num_kpts`, `num_wann`: Dimensions for consistency checks.
    - `seedname` (character(len=50), intent(in)): Base name for files.
    - `error`, `comm`.
- **Returns:** Populates the `sitesym` derived type.

### Subroutine `sitesym_dealloc`
- **Description:** Deallocates all allocatable arrays within the `sitesym` derived type.
- **Parameters:** `sitesym`, `error`, `comm`.

### Symmetrization Subroutines
- **`sitesym_symmetrize_u_matrix(sitesym, umat, num_bands, ndim, num_kpts, num_wann, ...)`:** Symmetrizes a U-matrix (either `U_matrix_opt` from disentanglement or `U_matrix` from wannierisation). It enforces the condition $U(R\mathbf{k}) = d(R,\mathbf{k}) U(\mathbf{k}) D^\dagger(R,\mathbf{k})$, where $d$ and $D$ are representation matrices for Bloch states and Wannier functions, respectively.
    - `ndim`: Leading dimension of `umat` (can be `num_bands` or `num_wann`).
    - `lwindow_in` (optional): Logical mask for states within the disentanglement window.
- **`sitesym_symmetrize_gradient(sitesym, grad, imode, num_kpts, num_wann)`:** Symmetrizes the gradient of the spread functional.
    - `imode=1`: Full symmetrization $\frac{1}{N_G N_{R'}} \sum_{R,R'} D^\dagger(R',k) d^\dagger(R,R'k) G(R R'k) d(R,R'k) D(R',k)$. (This seems to describe a more complex operation than what's implemented, which is closer to $G(k) \leftarrow \frac{1}{N_{R'}} \sum_{R'} D^\dagger(R',k) G(R'k) D(R',k)$ where $R'k=k$).
- **`sitesym_symmetrize_rotation(sitesym, urot, num_kpts, num_wann, ...)`:** Symmetrizes a rotation matrix `urot` (likely related to wannierisation steps) using the relation $U_{rot}(R\mathbf{k}) = D(R,\mathbf{k}) U_{rot}(\mathbf{k}) D^\dagger(R,\mathbf{k})$.
- **`sitesym_symmetrize_zmatrix(sitesym, czmat, num_bands, num_kpts, lwindow_in)`:** Symmetrizes the Z-matrix used in disentanglement: $Z(\mathbf{k}) \leftarrow \frac{1}{N_G N_{R'}} \sum_{R,R'} d^\dagger(R',k) Z(R R'k) d(R',k)$ (Again, the implementation seems to be $Z(k) \leftarrow \frac{1}{N_{R'}} \sum_{R'} d^\dagger(R',k) Z(R'k) d(R',k)$ where $R'k=k$).

### Subroutine `sitesym_dis_extract_symmetry`
- **Description:** A specialized routine for the disentanglement step when site symmetry is active. It likely modifies the way the optimal subspace is found or how the Z-matrix eigenvectors are treated to be compatible with symmetry. It involves an iterative refinement of `umat` (presumably `U_matrix_opt`).
- **Parameters:** `sitesym`, `lambda` (output Lagrange multipliers?), `umat` (input/output), `zmat` (input Z-matrix for the current k-point `ik`), `ik`, `n` (number of states in window at `ik`), `num_bands`, `num_wann`, `stdout`, `error`, `comm`.

### Subroutine `sitesym_replace_d_matrix_band`
- **Description:** Replaces `sitesym%d_matrix_band` with `sitesym%d_matrix_wann` after the disentanglement phase, effectively changing the basis for symmetry operations from the Bloch bands to the `num_wann` selected Wannier-like states.
- **Parameters:** `sitesym`, `num_wann`.

### Subroutine `sitesym_slim_d_matrix_band` (Not currently called)
- **Description:** Intended to reduce the dimension of `sitesym%d_matrix_band` if only states within a certain window (`lwindow_in`) are considered. Appears to be unused in the current version.

### Internal Subroutines
- **`symmetrize_ukirr(num_wann, num_bands, ir, ndim, umat, sitesym, stdout, error, comm, n)`:** Symmetrizes $U(\mathbf{k}_{ir})$ for an irreducible k-point $\mathbf{k}_{ir}$ by averaging over symmetry operations $R'$ that leave $\mathbf{k}_{ir}$ invariant ($R'\mathbf{k}_{ir} = \mathbf{k}_{ir} + \mathbf{G}$): $U \leftarrow \frac{1}{N_{R'}} \sum_{R'} d^\dagger(R',\mathbf{k}_{ir}) U D(R',\mathbf{k}_{ir})$. It then orthonormalizes the result.
- **`orthogonalize_u(ndim, m, u, n, error, comm)`:** Orthonormalizes the columns of a matrix `u` (of size `n x m`, with `n <= ndim`) using Singular Value Decomposition (SVD).

## Important Variables/Constants

- **`sitesym_type` (Derived Type from `w90_wannier90_types`):**
    - `nsymmetry` (integer): Number of symmetry operations.
    - `nkptirr` (integer): Number of irreducible k-points.
    - `ik2ir(num_kpts)` (integer): Maps a k-point index to its corresponding irreducible k-point index.
    - `ir2ik(nkptirr)` (integer): Maps an irreducible k-point index to its actual k-point index in the full list.
    - `kptsym(nsymmetry, nkptirr)` (integer): For each irreducible k-point `ir` and symmetry operation `isym`, stores the index of the k-point $R_{isym}\mathbf{k}_{ir}$.
    - `d_matrix_band(num_bands, num_bands, nsymmetry, nkptirr)` (complex): Representation matrices $d_{\mu\nu}(R,\mathbf{k})$ for symmetry operations acting on Bloch bands.
    - `d_matrix_wann(num_wann, num_wann, nsymmetry, nkptirr)` (complex): Representation matrices $D_{mn}(R,\mathbf{k})$ for symmetry operations acting on Wannier functions.
    - `symmetrize_eps` (real(kind=dp)): Tolerance for convergence in `symmetrize_ukirr`.

## Usage Examples
The use of this module is typically enabled by setting `site_symmetry = .true.` in the `seedname.win` input file.

1.  **Reading Symmetry Data:** Early in the calculation, `sitesym_read` is called if `lsitesymmetry` is true.
    ```fortran
    ! In wannier_prog.F90 or postw90_common.F90
    if (lsitesymmetry) then
      call sitesym_read(sitesym, num_bands, num_kpts, num_wann, seedname, error, comm)
      ! ... error check ...
    end if
    ```

2.  **During Disentanglement:**
    - `sitesym_symmetrize_u_matrix` is called to symmetrize `U_matrix_opt`.
    - `sitesym_dis_extract_symmetry` is called instead of the standard `dis_extract` path if symmetry is active.
    - `sitesym_replace_d_matrix_band` is called after disentanglement.

3.  **During Wannierisation:**
    - `sitesym_symmetrize_gradient` is called to symmetrize the gradient of $\Omega$.
    - `sitesym_symmetrize_rotation` is called to symmetrize the rotation matrices $U_{mn}^{(\mathbf{k})}$ obtained from the steepest descent or CG step.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `cmplx_1`, `cmplx_0`.
    - `w90_comms`: For `w90comm_type`.
    - `w90_error`: For error handling (`w90_error_type`, `set_error_fatal`, `set_error_unconv`, `set_error_alloc`, `set_error_dealloc`, `set_error_file`).
    - `w90_utility`: For `utility_zgemm` (matrix multiplication).
    - `w90_wannier90_types`: Defines `sitesym_type`.
    - `w90_io`: For `io_file_unit`.

- **External Libraries:**
    - **LAPACK/BLAS:** `zgemm` (complex matrix-matrix multiplication) is used extensively. `zgesvd` (complex SVD) is used in `orthogonalize_u`. `ZHPGVX` (complex Hermitian generalized eigenvalue problem) is used in `sitesym_dis_extract_symmetry`.

- **Interactions:**
    - Reads symmetry information from `seedname.dmn`. This file needs to be prepared by the user or a pre-processing tool that understands the crystal symmetry and how it acts on the Bloch states.
    - Modifies the behavior of the disentanglement (`w90_disentangle`) and wannierisation (`w90_wannierise`) procedures by ensuring that the relevant matrices ($U_{opt}$, $U$, gradients) conform to the specified site symmetries.
    - Aims to produce Wannier functions that transform according to irreducible representations of the site symmetry group of the Wannier centers.
```
