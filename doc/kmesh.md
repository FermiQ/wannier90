## Overview
The `w90_kmesh` module is responsible for analyzing the regular k-point mesh used in an electronic structure calculation and determining the necessary information for constructing Wannier functions. This primarily involves identifying neighboring k-points and the "b-vectors" that connect them. These b-vectors are crucial for calculating the overlap matrices $M_{mn}^{(\mathbf{k,b})} = \langle u_{m\mathbf{k}} | u_{n\mathbf{k+b}} \rangle$ which are fundamental to the Wannier interpolation scheme. The module determines shells of nearest-neighbor k-points, their weights, and ensures that a completeness relation (Eq. B1 of PRB 56, 12847 (1997)) is satisfied for the chosen b-vectors. It also handles writing this k-mesh information to the `seedname.nnkp` file, which can be used by external codes or for restarting calculations.

## Key Components

### Module `w90_kmesh`
- **Description:** The main module containing all subroutines and data related to k-mesh analysis and b-vector determination.

### Subroutine `kmesh_get`
- **Description:** This is the main routine for calculating the b-vectors and their properties. It identifies shells of neighboring k-points, determines their distances and multiplicities, and selects a set of shells (either automatically or based on user input) to satisfy the completeness relation (B1 condition). It populates the `kmesh_info` derived type with this information.
- **Parameters:**
    - `kmesh_input`: Input parameters controlling shell search and selection.
    - `kmesh_info`: Output structure containing detailed k-mesh information (b-vectors, weights, neighbor lists).
    - `print_output`: Controls printing of output.
    - `kpt_latt`: Fractional coordinates of k-points.
    - `real_lattice`: Real-space lattice vectors.
    - `num_kpts`: Total number of k-points.
    - `gamma_only`: Logical, true if only the Gamma point is considered (special handling).
    - `stdout`: Output unit for printing.
    - `timer`: Timing object.
    - `error`: Error handling object.
    - `comm`: MPI communicator.
- **Returns:** Populates `kmesh_info` with `nnlist`, `nncell`, `bk`, `bka`, `wb`, `nntot`, `nnh`.

### Subroutine `kmesh_write`
- **Description:** Writes the k-mesh neighborhood information and other relevant data (lattice vectors, k-points, projections, excluded bands) to the `seedname.nnkp` file. This file is essential for the ab initio code interface when Wannier90 is used to generate the $M_{mn}$ and $A_{mn}$ matrices.
- **Parameters:**
    - `exclude_bands`: List of bands to exclude.
    - `kmesh_info`: K-mesh information to be written.
    - `proj_input`: Projection information.
    - `print_output`: Controls printing.
    - `kpt_latt`: Fractional coordinates of k-points.
    - `real_lattice`: Real-space lattice vectors.
    - `num_kpts`, `num_proj`: Number of k-points and projections.
    - `calc_only_A`: Logical, if true only $A_{mn}$ matrices are needed.
    - `spinors`: Logical, true if dealing with spinors.
    - `seedname`: Base name for files.
    - `timer`: Timing object.
- **Returns:** Creates the `seedname.nnkp` file.

### Subroutine `kmesh_dealloc`
- **Description:** Deallocates all allocatable arrays within the `kmesh_info` structure to free memory.
- **Parameters:**
    - `kmesh_info`: The k-mesh data structure.
    - `error`: Error handling object.
    - `comm`: MPI communicator.
- **Returns:** Frees memory associated with `kmesh_info`.

### Subroutine `kmesh_supercell_sort`
- **Description:** Sorts the reciprocal lattice cells in a supercell search region by their distance from the origin. This optimization significantly speeds up the search for k-point neighbors.
- **Parameters:**
    - `print_output`: Controls printing.
    - `recip_lattice`: Reciprocal lattice vectors.
    - `lmn`: Output array of sorted cell indices (l,m,n).
    - `timer`: Timing object.
- **Returns:** Populates `lmn` with cell indices sorted by distance.

### Subroutine `kmesh_get_bvectors`
- **Description:** For a given k-point and a specific shell distance, this routine finds all b-vectors (vectors connecting the k-point to its neighbors in that shell).
- **Parameters:**
    - `kmesh_input`: Input parameters (e.g., tolerance).
    - `print_output`: Controls printing.
    - `bvector`: Output array of b-vectors.
    - `kpt_cart`: Cartesian coordinates of k-points.
    - `recip_lattice`: Reciprocal lattice vectors.
    - `shell_dist`: The distance defining the current shell.
    - `lmn`: Sorted list of supercells to search in.
    - `kpt`: Index of the reference k-point.
    - `multi`: Multiplicity of the shell (number of b-vectors to find).
    - `num_kpts`: Total number of k-points.
    - `timer`, `error`, `comm`.
- **Returns:** Populates `bvector` with the b-vectors for the specified shell.

### Subroutine `kmesh_shell_automatic`
- **Description:** Automatically determines the shells of k-point neighbors and their weights (`bweight`) required to satisfy the B1 completeness condition. It iteratively adds shells, rejecting linearly dependent ones, until B1 is met.
- **Parameters:** Similar to `kmesh_get`, plus `bweight` (output weights) and `dnn` (shell distances).
- **Returns:** Populates `kmesh_input%shell_list`, `kmesh_input%num_shells`, and `bweight`.

### Subroutine `kmesh_shell_fixed`
- **Description:** Calculates the weights (`bweight`) for a user-specified list of shells to satisfy the B1 condition. The shells are provided in `kmesh_input%shell_list`.
- **Parameters:** Similar to `kmesh_shell_automatic`.
- **Returns:** Populates `bweight`.

### Subroutine `kmesh_shell_from_file` (developer tool)
- **Description:** Reads a user-defined set of b-vectors from a `seedname.kshell` file and calculates the weights. This is a developer option not intended for general use.
- **Parameters:** Similar to `kmesh_shell_fixed`, plus `bvec_inp` (b-vectors from file) and `seedname`.
- **Returns:** Populates `bweight` and related `kmesh_input` parameters.

### Function `internal_maxloc`
- **Description:** A reproducible version of `maxloc` to ensure consistent ordering of b-vectors, especially when dealing with degenerate shells.
- **Parameters:** `dist` (array of distances).
- **Returns:** The index of the maximum value in `dist`, consistently choosing the smallest index in case of ties.

## Important Variables/Constants

- **`kmesh_info_type` (Derived Type):** A structure that holds all the key information about the k-mesh neighbors. Its components include:
    - **`nnlist (num_kpts, nntot)` (integer):** For each k-point `nkp`, `nnlist(nkp, nn)` is the index of its `nn`-th neighbor.
    - **`nncell (3, num_kpts, nntot)` (integer):** For each k-point `nkp`, `nncell(:, nkp, nn)` gives the G-vector (in reciprocal lattice units) that translates the `nn`-th neighbor (which is in the 1st BZ) to the actual $\mathbf{k+b}$ vector.
    - **`bk (3, nntot, num_kpts)` (real(kind=dp)):** The b-vectors $\mathbf{b_k}$ for each k-point `nkp` to its `nntot` neighbors. $\mathbf{b_k} = (\mathbf{k'} + \mathbf{G}) - \mathbf{k}$.
    - **`bka (3, nnh)` (real(kind=dp)):** The unique b-vector directions (nnh = nntot/2, not considering inversion symmetry).
    - **`wb (nntot)` (real(kind=dp)):** Weights associated with each b-vector, used in the B1 completeness relation.
    - **`nntot` (integer):** Total number of neighbors for each k-point (sum of multiplicities of chosen shells).
    - **`nnh` (integer):** Number of unique b-vector directions (usually `nntot/2`).
    - `neigh(num_kpts, nnh)` (integer): Index mapping to `bk` for each of the `nnh` unique directions.
    - `wbtot` (real(kind=dp)): Sum of weights.
    - `explicit_nnkpts` (logical): True if nnkp info was read from file.

- **`kmesh_input_type` (Derived Type):** Holds input parameters related to k-mesh generation:
    - **`search_shells` (integer):** Number of shells to search for neighbors.
    - **`num_shells` (integer):** Number of shells to *use* for constructing b-vectors (can be 0 for automatic).
    - **`shell_list (max_shells)` (integer):** Indices of shells to use if `num_shells > 0`.
    - **`tol` (real(kind=dp)):** Tolerance for comparing distances and satisfying B1.
    - `skip_B1_tests` (logical): If true, skips the B1 completeness check.

- **`nsupcell` (integer, parameter, default=5):** Size of the supercell (in units of reciprocal cell vectors) in which to search for k-point neighbors.
- **`max_shells` (integer, from `w90_types`):** Maximum number of shells that can be considered.
- **`num_nnmax` (integer, from `w90_types`):** Maximum number of nearest neighbors a k-point can have.

## Usage Examples
The `w90_kmesh` module is primarily used internally by Wannier90.
1.  During initialization, if `wannier_setup` is true (i.e., not reading all info from checkpoint files), `kmesh_get` is called.
    ```fortran
    ! In wannier_prog.F90 or similar
    if (.not. kmesh_info%explicit_nnkpts) call kmesh_get(kmesh_input, kmesh_info, print_output, &
                                                         kpt_latt, real_lattice, &
                                                         num_kpts, gamma_only, stdout, &
                                                         timer, error, comm)
    ```
2.  If `postproc_setup` is true (often for generating data for the ab initio code to compute overlaps), `kmesh_write` is called to create the `seedname.nnkp` file.
    ```fortran
    ! In wannier_prog.F90 or similar
    if (w90_calculation%postproc_setup) then
      if (on_root) call kmesh_write(exclude_bands, kmesh_info, input_proj, print_output, kpt_latt, &
                                    real_lattice, num_kpts, num_proj, calc_only_A, &
                                    w90_system%spinors, seedname, timer)
      ! ... stop program ...
    end if
    ```
3.  At the end of the program, `kmesh_dealloc` is called to free allocated memory.

The `seedname.nnkp` file generated by `kmesh_write` is a critical interface component. Ab initio codes read this file to understand which k-point overlaps ($M_{mn}^{(\mathbf{k,b})}$) and projections ($A_{mn}^{(\mathbf{k})}$) need to be computed and written out for Wannier90.

## Dependencies and Interactions

- **Internal Dependencies:** (Modules used via `USE` statements)
    - `w90_constants`: Provides data precision (`dp`) and mathematical constants.
    - `w90_types`: Defines core derived data types (`kmesh_info_type`, `kmesh_input_type`, `print_output_type`, `timer_list_type`, `proj_input_type`, `max_shells`, `num_nnmax`).
    - `w90_error`: Handles error reporting and management (e.g., `set_error_alloc`, `set_error_fatal`).
    - `w90_comms`: Provides the `w90comm_type` for MPI communication.
    - `w90_io`: For file I/O operations (`io_file_unit`, `io_date`) and timing (`io_stopwatch_start`, `io_stopwatch_stop`).
    - `w90_utility`: Provides utility functions like reciprocal lattice calculation (`utility_recip_lattice`, `utility_recip_lattice_base`), coordinate transformation (`utility_frac_to_cart`), and vector comparison (`utility_compar`).

- **External Libraries:**
    - **LAPACK/BLAS:** The `dgesvd` routine (from LAPACK) is used in `kmesh_shell_automatic` and `kmesh_shell_fixed` for singular value decomposition, which helps in solving the system of equations for shell weights.

- **Interactions:**
    - Receives k-point coordinates (`kpt_latt`) and real-space lattice vectors (`real_lattice`) typically read from input files or provided by the main program.
    - Receives user-defined parameters for shell selection via `kmesh_input_type` (e.g., `num_shells`, `shell_list`, `search_shells`, `kmesh_tol`).
    - Populates `kmesh_info_type` which is then used by other modules, notably `w90_overlap` (to know which overlaps $M_{mn}^{(\mathbf{k,b})}$ to compute or read) and `w90_wannierise`.
    - `kmesh_write` produces `seedname.nnkp`, which is a key file read by ab initio codes when interfacing with Wannier90 for the `-pp` (preprocess) run.
    - The module ensures the k-mesh information is consistent and satisfies mathematical requirements (B1 condition) for a valid Wannier interpolation.
```
