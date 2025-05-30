## Overview
The `w90_hamiltonian` module is responsible for calculating and managing the Hamiltonian of the system in the basis of the Wannier functions (WFs). This involves transforming the Hamiltonian from the Bloch basis (obtained from an ab initio calculation) to the WF basis. It also handles the real-space representation of the Hamiltonian, $H_{mn}(\mathbf{R})$, which describes the interaction between WF $m$ at the origin and WF $n$ in cell $\mathbf{R}$. This module provides routines to set up necessary structures, compute $H(\mathbf{R})$, and write it to output files in various formats.

## Key Components

### Module `w90_hamiltonian`
- **Description:** The main module containing all subroutines and data related to the Wannier Hamiltonian.

### Subroutine `hamiltonian_setup`
- **Description:** Allocates arrays and sets up data structures required for Hamiltonian calculations. This includes determining the Wigner-Seitz grid points (`irvec`, `ndegen`, `nrpts`) and allocating space for the real-space Hamiltonian `ham_r` and k-space Hamiltonian `ham_k`.
- **Parameters:**
    - `ham_logical`: Logical flags for Hamiltonian setup and status.
    - `print_output`: Controls printing of output.
    - `ws_region`: Parameters for Wigner-Seitz cell construction.
    - `w90_calculation`: General calculation parameters.
    - `ham_k`, `ham_r`: k-space and real-space Hamiltonian arrays.
    - `real_lattice`: Real-space lattice vectors.
    - `wannier_centres_translated`: WF centres translated to the home cell.
    - `irvec`, `mp_grid`, `ndegen`, `num_kpts`, `num_wann`, `nrpts`, `rpt_origin`: K-mesh, Wigner-Seitz grid, and dimensions.
    - `bands_plot_mode`, `transport_mode`: Strings indicating calculation modes.
    - `stdout`: Output unit.
    - `timer`: Timing object.
    - `error`: Error handling object.
    - `comm`: MPI communicator.
- **Returns:** Initializes Hamiltonian related arrays and sets `ham_logical%ham_have_setup` to true.

### Subroutine `hamiltonian_dealloc`
- **Description:** Deallocates all allocatable arrays associated with the Hamiltonian module to free memory.
- **Parameters:**
    - `ham_logical`: Logical flags for Hamiltonian setup and status.
    - `ham_k`, `ham_r`: k-space and real-space Hamiltonian arrays.
    - `wannier_centres_translated`: Translated WF centres.
    - `irvec`, `ndegen`: Wigner-Seitz grid arrays.
    - `error`: Error handling object.
    - `comm`: MPI communicator.
- **Returns:** Resets logical flags in `ham_logical`.

### Subroutine `hamiltonian_get_hr`
- **Description:** Calculates the Hamiltonian in the Wannier function basis in real space, $H_{mn}(\mathbf{R})$. It first transforms the Bloch eigenvalues into the optimal subspace (if disentanglement is used), then rotates this into the basis of smooth Bloch states (Wannier gauge), and finally Fourier transforms to real space. It can optionally translate Wannier centres to the home unit cell.
- **Parameters:** Many, including input Bloch eigenvalues (`eigval`), rotation matrices (`u_matrix`, `u_matrix_opt`), k-point information (`kpt_latt`), Wannier centres, and output arrays for `ham_r` and `ham_k`.
- **Returns:** Populates `ham_r` with the real-space Hamiltonian matrix elements. Sets `ham_logical%have_ham_r` to true.

### Subroutine `internal_translate_centres` (contained in `hamiltonian_get_hr`)
- **Description:** Translates the Wannier function centres to be within a specified region, typically the home unit cell or a region centered around the average atomic position.
- **Parameters:** `atom_data`, `real_space_ham` (controls translation), `real_lattice`, `wannier_centres` (input), `wannier_centres_translated` (output), `shift_vec` (output shifts), `iprint`, `num_wann`, `error`.
- **Returns:** Populates `wannier_centres_translated` and `shift_vec`.

### Subroutine `hamiltonian_write_hr`
- **Description:** Writes the real-space Hamiltonian matrix elements $H_{mn}(\mathbf{R})$ to a formatted text file named `seedname_hr.dat`.
- **Parameters:** `ham_logical`, `ham_r`, `irvec`, `ndegen`, `nrpts`, `num_wann`, `timing_level`, `seedname`, `timer`, `error`, `comm`.
- **Returns:** Creates `seedname_hr.dat`. Sets `ham_logical%hr_written` to true.

### Subroutine `hamiltonian_wigner_seitz`
- **Description:** Determines the set of real-space lattice vectors $\mathbf{R}$ that fall within the Wigner-Seitz cell of the superlattice defined by `mp_grid`. This is crucial for defining the range of $H_{mn}(\mathbf{R})$.
- **Parameters:** `ws_region` (controls search size and tolerance), `print_output`, `real_lattice`, `irvec` (output R-vectors), `mp_grid`, `ndegen` (output degeneracy of R-vectors), `nrpts` (output number of R-vectors), `rpt_origin` (index of $\mathbf{R}=0$), `stdout`, `timer`, `error`, `count_pts` (logical: if true, only counts points, else stores them), `comm`.
- **Returns:** Populates `irvec`, `ndegen`, `nrpts`, `rpt_origin`.

### Subroutine `hamiltonian_write_rmn`
- **Description:** Writes the matrix elements of the position operator in the Wannier basis, $\langle W_m(\mathbf{0}) | \mathbf{r} | W_n(\mathbf{R}) \rangle$, to a formatted text file named `seedname_r.dat`. These are calculated using the Berry connection matrices (`m_matrix`).
- **Parameters:** `kmesh_info`, `m_matrix`, `kpt_latt`, `irvec`, `nrpts`, `num_kpts`, `num_wann`, `seedname`, `error`, `comm`.
- **Returns:** Creates `seedname_r.dat`.

### Subroutine `hamiltonian_write_tb`
- **Description:** Writes information for constructing a tight-binding model based on Wannier functions to `seedname_tb.dat`. This includes lattice vectors, $H_{mn}(\mathbf{R})$, and $\langle W_m(\mathbf{0}) | \mathbf{r} | W_n(\mathbf{R}) \rangle$.
- **Parameters:** `ham_logical`, `kmesh_info`, `ham_r`, `m_matrix`, `kpt_latt`, `real_lattice`, `irvec`, `ndegen`, `nrpts`, `num_kpts`, `num_wann`, `timing_level`, `seedname`, `timer`, `error`, `comm`.
- **Returns:** Creates `seedname_tb.dat`. Sets `ham_logical%tb_written` to true.

## Important Variables/Constants

- **`ham_r (num_wann, num_wann, nrpts)` (complex(kind=dp), allocatable):** Stores the real-space Hamiltonian matrix elements $H_{mn}(\mathbf{R})$.
- **`ham_k (num_wann, num_wann, num_kpts)` (complex(kind=dp), allocatable):** Stores the k-space Hamiltonian matrix elements $H_{mn}(\mathbf{k})$ in the Wannier gauge.
- **`irvec (3, nrpts)` (integer, allocatable):** Stores the components of the Wigner-Seitz lattice vectors $\mathbf{R}$ in the basis of the lattice vectors.
- **`ndegen (nrpts)` (integer, allocatable):** The weight (degeneracy) of each $\mathbf{R}$ vector, $1/N_R$, where $N_R$ is the number of equivalent points.
- **`nrpts` (integer):** The total number of Wigner-Seitz lattice vectors $\mathbf{R}$.
- **`rpt_origin` (integer):** The index in `irvec` corresponding to the $\mathbf{R}=0$ vector.
- **`ham_logical` (type(ham_logical_type)):** A derived type containing logical flags indicating the status of various Hamiltonian calculations and operations (e.g., `ham_have_setup`, `have_ham_r`, `hr_written`).
- **`wannier_centres_translated (3, num_wann)` (real(kind=dp), allocatable):** Cartesian coordinates of Wannier function centres after being translated to the home cell.

## Usage Examples
The subroutines in this module are typically called sequentially by the main `wannier` program:
1.  `hamiltonian_setup` is called to initialize and allocate arrays.
2.  `hamiltonian_get_hr` is called to compute the real-space Hamiltonian $H_{mn}(\mathbf{R})$ using Bloch states, eigenvalues, and rotation matrices (U, U_opt) obtained from previous steps (overlap, disentanglement, wannierisation).
3.  If specific output files are requested (controlled by input parameters in `seedname.win`), then `hamiltonian_write_hr`, `hamiltonian_write_rmn`, and/or `hamiltonian_write_tb` are called to write the computed matrices to disk.
4.  Finally, `hamiltonian_dealloc` is called at the end of the program to free memory.

Conceptual usage within the larger Wannier90 workflow:
```fortran
! In wannier_prog.F90 or similar driving routine:

call hamiltonian_setup(...) ! Allocate and prepare
! ... (calculations for U_matrix, eigval, etc.) ...
call hamiltonian_get_hr(...) ! Compute H(R)
if (wannier_plot_hr) call hamiltonian_write_hr(...) ! Write H(R) if requested
! ... (other plotting or transport calculations) ...
call hamiltonian_dealloc(...) ! Clean up
```

## Dependencies and Interactions

- **Internal Dependencies:** (Modules used via `USE` statements)
    - `w90_constants`: Provides physical constants (`twopi`, `cmplx_i`) and data precision (`dp`).
    - `w90_types`: Defines basic derived data types (`print_output_type`, `ws_region_type`, `timer_list_type`, `atom_data_type`, `dis_manifold_type`, `kmesh_info_type`, `w90comm_type`).
    - `w90_error`: Used for error handling and reporting (`set_error_alloc`, `set_error_dealloc`, `set_error_fatal`, `set_error_file`).
    - `w90_io`: For file I/O operations (`io_file_unit`, `io_date`) and timing (`io_stopwatch_start`, `io_stopwatch_stop`).
    - `w90_utility`: Provides utility functions like matrix inversion (`utility_inverse_mat`), coordinate transformations (`utility_cart_to_frac`, `utility_frac_to_cart`), and metric tensor calculation (`utility_metric`).
    - `w90_wannier90_types`: Defines specific derived types for Wannier90 (`w90_calculation_type`, `real_space_ham_type`, `ham_logical_type`).

- **External Libraries:**
    - None directly called from this module, but the data it processes (eigenvalues, Bloch states) originates from ab initio codes that often link to BLAS/LAPACK for numerical linear algebra.

- **Interactions:**
    - Receives data (eigenvalues `eigval`, rotation matrices `u_matrix`, `u_matrix_opt`, k-point data `kpt_latt`, `kmesh_info`, overlap matrices `m_matrix` for position operator) from other parts of the Wannier90 code (e.g., `w90_wannierise`, `w90_overlap`, `w90_disentangle`).
    - Provides the real-space Hamiltonian `ham_r` and related quantities to other modules for further calculations, such as plotting routines (`w90_plot`) or transport calculations (`w90_transport`).
    - Writes output files (`seedname_hr.dat`, `seedname_r.dat`, `seedname_tb.dat`) that can be used by users or other post-processing tools for analysis or to build tight-binding models.
    - The `hamiltonian_wigner_seitz` subroutine is crucial for defining the spatial extent of real-space interactions.
    - The `internal_translate_centres` subroutine helps in ensuring that Wannier functions are consistently located for analysis and plotting.
```
