## Overview
The `w90_overlap` module is responsible for managing the overlap matrices $M_{mn}^{(\mathbf{k,b})} = \langle u_{m\mathbf{k}} | u_{n\mathbf{k+b}} \rangle$ and the projection matrices $A_{mn}^{(\mathbf{k})} = \langle \psi_{m\mathbf{k}} | g_n \rangle$. These matrices are fundamental inputs for the Wannier90 code, typically provided by an ab initio electronic structure code. This module handles the allocation of memory for these matrices, reading them from files (`seedname.mmn` for $M$ and `seedname.amn` for $A$), and performing initial transformations if necessary (e.g., when no disentanglement is performed, or for Gamma-point only calculations).

## Key Components

### Module `w90_overlap`
- **Description:** The main module for handling overlap and projection matrices.

### Subroutine `overlap_allocate`
- **Description:** Allocates memory for the various matrices related to overlaps and projections. The actual dimensions depend on whether disentanglement is active (`num_bands > num_wann`). It prepares arrays for `a_matrix`, `m_matrix`, `m_matrix_local`, `m_matrix_orig`, `m_matrix_orig_local`, `u_matrix`, and `u_matrix_opt`.
- **Parameters:** Pointers to all the matrices to be allocated, dimensions (`nntot`, `num_bands`, `num_kpts`, `num_wann`), `timing_level`, `timer`, `error`, `comm`.
- **Returns:** Allocates the specified matrices.

### Subroutine `overlap_read`
- **Description:** Reads the $M_{mn}^{(\mathbf{k,b})}$ matrices from `seedname.mmn` and, if `use_bloch_phases` is false, the $A_{mn}^{(\mathbf{k})}$ matrices from `seedname.amn`. It performs checks on the dimensions read from the files against the expected dimensions. For MPI runs, the root node reads the files and then scatters the data to other nodes (`m_matrix_orig_local` or `m_matrix_local`). If `use_bloch_phases` is true, `u_matrix` is initialized to identity. It also calls `overlap_project` or `overlap_project_gamma` if no disentanglement and no Bloch phases are used.
- **Parameters:** Similar to `overlap_allocate`, plus `kmesh_info`, `select_projection`, `sitesym`, logical flags (`cp_pp`, `gamma_only`, `lsitesymmetry`, `use_bloch_phases`), `seedname`, `stdout`.
- **Returns:** Populates the allocated matrices with data from files or initializes them.

### Subroutine `overlap_project`
- **Description:** This routine is called when disentanglement is *not* performed (`num_bands == num_wann`) and `use_bloch_phases` is false. It takes the initial projections $A_{mn}^{(\mathbf{k})}$ (stored in `u_matrix` at this stage) and constructs an initial guess for the unitary matrices $U_{mn}^{(\mathbf{k})}$ using a Lowdin transformation ($U = S^{-1/2}A$, where $S=A A^\dagger$). The overlap matrices $M_{mn}^{(\mathbf{k,b})}$ are then rotated into this new basis: $M' = U^\dagger M U$.
- **Parameters:** `sitesym`, `m_matrix`, `m_matrix_local`, `u_matrix` (input $A$, output $U$), `nnlist`, `nntot`, `num_bands`, `num_kpts`, `num_wann`, `timing_level`, `lsitesymmetry`, `stdout`, `timer`, `error`, `comm`.
- **Returns:** Modifies `u_matrix` to be the Lowdin-transformed unitary matrices and updates `m_matrix` and `m_matrix_local`.

### Subroutine `overlap_project_gamma`
- **Description:** A specialized version of `overlap_project` for Gamma-point only calculations. It performs a similar Lowdin transformation but assumes real arithmetic for the initial projection matrices and ensures the resulting $U$ matrix is real.
- **Parameters:** `m_matrix`, `u_matrix`, `nntot`, `num_wann`, `timing_level`, `stdout`, `timer`, `error`, `comm`.
- **Returns:** Modifies `u_matrix` and `m_matrix`.

### Subroutine `overlap_rotate`
- **Description:** This routine is specifically for post-processing Car-Parrinello (CPMD) calculations, particularly when `cp_pp` is true. It reads a transformation matrix $\lambda$ from `lambda.dat` (presumably eigenvectors from diagonalizing an overlap or Hamiltonian matrix in CPMD) and rotates both `m_matrix_orig` and `a_matrix` into this basis of Kohn-Sham eigenstates.
- **Parameters:** `a_matrix`, `m_matrix_orig`, `nntot`, `num_bands`, `timing_level`, `timer`, `error`, `comm`.
- **Returns:** Modifies `a_matrix` and `m_matrix_orig`.

### Subroutine `overlap_dealloc`
- **Description:** Deallocates all matrices that were allocated by `overlap_allocate`.
- **Parameters:** Pointers to all potentially allocated matrices, `error`, `comm`.
- **Returns:** Frees memory.

## Important Variables/Constants

- **`a_matrix (num_bands, num_wann, num_kpts)` (complex(kind=dp), allocatable):** Stores the projection matrices $A_{mn}^{(\mathbf{k})}$. If disentanglement is off, this is not used directly for $A_{mn}$ but `u_matrix` might hold initial projections.
- **`m_matrix (num_wann, num_wann, nntot, num_kpts)` (complex(kind=dp), allocatable):** Stores the overlap matrices $M_{mn}^{(\mathbf{k,b})}$ in the (potentially optimized) `num_wann` dimensional basis.
- **`m_matrix_local (num_wann, num_wann, nntot, num_kpts_local)` (complex(kind=dp), allocatable):** The portion of `m_matrix` relevant to the current MPI process.
- **`m_matrix_orig (num_bands, num_bands, nntot, num_kpts)` (complex(kind=dp), allocatable):** Stores the original overlap matrices $M_{mn}^{(\mathbf{k,b})}$ in the full `num_bands` dimensional basis, read from `seedname.mmn`. Used when disentanglement is active.
- **`m_matrix_orig_local (num_bands, num_bands, nntot, num_kpts_local)` (complex(kind=dp), allocatable):** The portion of `m_matrix_orig` relevant to the current MPI process.
- **`u_matrix (num_wann, num_wann, num_kpts)` (complex(kind=dp), allocatable):** Stores the unitary rotation matrices $U_{mn}^{(\mathbf{k})}$ that transform from the initial Bloch states (or the optimal subspace from disentanglement) to the Wannier gauge. If `use_bloch_phases` is false and no disentanglement, it's initialized from $A_{mn}$.
- **`u_matrix_opt (num_bands, num_wann, num_kpts)` (complex(kind=dp), allocatable):** Stores the unitary matrices defining the optimal `num_wann` dimensional subspace from the original `num_bands` Bloch states. Used during disentanglement.

## Usage Examples
The routines in this module are called sequentially in the main Wannier90 program flow.

1.  **Memory Allocation:** `overlap_allocate` is called first to prepare space for the matrices.
    ```fortran
    ! In wannier_prog.F90
    call overlap_allocate(a_matrix, m_matrix, m_matrix_local, m_matrix_orig, &
                          m_matrix_orig_local, u_matrix, u_matrix_opt, kmesh_info%nntot, &
                          num_bands, num_kpts, num_wann, print_output%timing_level, &
                          timer, error, comm)
    ```

2.  **Reading Data:** `overlap_read` is then called to read data from `.mmn` and `.amn` files.
    ```fortran
    ! In wannier_prog.F90
    call overlap_read(kmesh_info, select_projection, sitesym, a_matrix, m_matrix, &
                      m_matrix_local, m_matrix_orig, m_matrix_orig_local, u_matrix, &
                      u_matrix_opt, num_bands, num_kpts, num_proj, num_wann, &
                      print_output%timing_level, cp_pp, gamma_only, lsitesymmetry, &
                      use_bloch_phases, seedname, stdout, timer, error, comm)
    ```
    If `use_bloch_phases` is `.false.` and `disentanglement` is `.false.`, `overlap_read` internally calls `overlap_project` (or `overlap_project_gamma`).

3.  **Deallocation:** At the end of the calculation, `overlap_dealloc` is called.
    ```fortran
    ! In wannier_prog.F90
    call overlap_dealloc(a_matrix, m_matrix, m_matrix_local, m_matrix_orig, &
                         m_matrix_orig_local, u_matrix, u_matrix_opt, error, comm)
    ```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_1`, `eps5`.
    - `w90_comms`: For MPI communication (scatter/gather data via `comms_scatterv`, `comms_bcast`).
    - `w90_io`: For file operations (`io_file_unit`) and timing (`io_stopwatch_start`, `io_stopwatch_stop`).
    - `w90_types`: Defines `kmesh_info_type`, `timer_list_type`.
    - `w90_wannier90_types`: Defines `select_projection_type`, `sitesym_type`.
    - `w90_error`: For error handling (`set_error_alloc`, `set_error_file`, `set_error_dealloc`, `set_error_fatal`).
    - `w90_utility`: For `utility_zgemm` (matrix multiplication).
    - `w90_sitesym`: If symmetry is enabled, `overlap_project` calls `sitesym_symmetrize_u_matrix`.

- **External Libraries:**
    - **LAPACK/BLAS:**
        - `zgesvd` (complex SVD) is used in `overlap_project`.
        - `dgesvd` (real SVD) is used in `overlap_project_gamma`.
        - `dspev` (real symmetric packed eigenvalue problem solver) is used in `overlap_rotate`.
        - `matmul` (Fortran intrinsic, often optimized with BLAS) is used in `overlap_rotate`.
        - `zgemm` (via `utility_zgemm`) and `dgemm` are used for matrix multiplications.

- **Interactions:**
    - Reads input files `seedname.mmn` and `seedname.amn` which are typically generated by an ab initio code during a `-pp` (preprocess) run of Wannier90, or directly by the interface to the ab initio code.
    - Provides the `m_matrix_orig` (or `m_matrix` if no disentanglement) and `a_matrix` to the `w90_disentangle` module if disentanglement is performed.
    - Provides the `m_matrix` and `u_matrix` (initial guess for Wannier gauge transformation) to the `w90_wannierise` module for the main spread minimization.
    - The `kmesh_info` (from `w90_kmesh`) is used to understand the structure of the `.mmn` file (number of neighbors `nntot`, neighbor lists `nnlist`, `nncell`).
```
