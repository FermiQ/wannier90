## Overview
The `w90_wannier90_readwrite` module provides subroutines specifically for reading and writing data structures and parameters used by the main `wannier90.x` executable. It builds upon the general input/output routines in `w90_readwrite` by handling more complex data types and program-specific logic. This includes parsing detailed settings for various calculation stages (disentanglement, wannierisation, plotting, transport), managing memory allocation for these settings, distributing parameters in MPI runs, and writing checkpoint files specific to `wannier90.x`.

## Key Components

### Module `w90_wannier90_readwrite`
- **Description:** The main module for `wannier90.x`-specific I/O operations.

### Type `w90_extra_io_type`
- **Description:** A local derived type used to temporarily store certain parameters read from the input file before they are transferred to their final locations in other modules/types.
- **Fields:**
    - `one_dim_axis` (character(len=20)): Specifies the axis for 1D systems or 1D-like processing.
    - `ccentres_frac(:, :)` (real(kind=dp), allocatable): Stores constrained Wannier function centers in fractional coordinates.

### Subroutine `w90_wannier90_readwrite_read`
- **Description:** This is a master subroutine that orchestrates the reading of all input parameters from the `seedname.win` file for a `wannier90.x` run. It calls numerous more specialized `w90_readwrite_read_X` and `w90_wannier90_readwrite_read_X` routines to parse different sections of the input file and populate the corresponding derived types. It handles setting default values and performs some initial consistency checks. This routine is intended to be called only by the root MPI process.
- **Parameters:** A very extensive list of `intent(inout)` arguments corresponding to nearly all configurable parameters and data structures in Wannier90 (e.g., `atom_data`, `band_plot`, `dis_control`, `kmesh_input`, `wann_control`, `w90_system`, `tran`, `print_output`, `num_bands`, `num_wann`, etc.).
- **Returns:** Populates all the output arguments with values read from the input file or with default values. Sets up error status if issues are encountered.

### Subroutine `w90_wannier90_readwrite_dist`
- **Description:** Distributes the parameters read by `w90_wannier90_readwrite_read` (on the root node) to all other MPI processes. This ensures all processes have a consistent set of parameters. It uses `comms_bcast` for each parameter.
- **Parameters:** Similar to `w90_wannier90_readwrite_read`, but parameters are `intent(inout)` as they are received by non-root nodes.
- **Returns:** Populates parameters on non-root nodes.

### Subroutine `w90_wannier90_readwrite_write`
- **Description:** Writes a summary of the parsed input parameters and system information to the main output file (`stdout`). This provides a record of the settings used for the calculation.
- **Parameters:** Similar to `w90_wannier90_readwrite_read`, but parameters are `intent(in)`.
- **Returns:** Prints formatted information to `stdout`.

### Subroutine `w90_wannier90_readwrite_write_chkpt`
- **Description:** Writes the current state of a `wannier90.x` calculation to a binary checkpoint file (`seedname.chk`). This includes $U$ matrices, $M$ matrices (if computed), Wannier function centers and spreads, disentanglement information, and key dimensional parameters.
- **Parameters:** `chkpt` (string indicating stage like 'postdis', 'postwann'), `exclude_bands`, `wannier_data`, `kmesh_info`, `kpt_latt`, `num_kpts`, `dis_manifold`, `num_bands`, `num_wann`, `u_matrix`, `u_matrix_opt`, `m_matrix`, `mp_grid`, `real_lattice`, `omega_invariant`, `have_disentangled`, `stdout`, `seedname`.
- **Returns:** Creates or overwrites `seedname.chk`.

### Subroutine `w90_wannier90_readwrite_memory_estimate`
- **Description:** Estimates the maximum RAM that will be allocated during different phases of the calculation (disentanglement, wannierisation, plotting) based on the input parameters. It prints this estimate to `stdout`.
- **Parameters:** Various data structures and dimension parameters needed to calculate memory usage.
- **Returns:** Prints memory estimates.

### Subroutine `w90_wannier90_readwrite_w90_dealloc`
- **Description:** Deallocates memory for various data structures specific to `wannier90.x` that were allocated during the read phase or calculations. It calls the general `w90_readwrite_dealloc` and then deallocates additional specific arrays.
- **Parameters:** `atom_data`, `band_plot`, `dis_spheres`, etc. (similar to `w90_wannier90_readwrite_read`).

### Specialized Reading Subroutines (e.g., `w90_wannier90_readwrite_read_sym`, `w90_wannier90_readwrite_read_w90_calcs`)
- **Description:** These are numerous internal helper subroutines called by `w90_wannier90_readwrite_read` to handle specific keywords or blocks in the input file. For example, `w90_wannier90_readwrite_read_sym` reads site symmetry parameters, and `w90_wannier90_readwrite_read_w90_calcs` reads flags for different calculation types (transport, plotting).

## Important Variables/Constants
This module primarily orchestrates the population of data types defined elsewhere (e.g., in `w90_types` and `w90_wannier90_types`). The `w90_extra_io_type` is a local type used for temporary storage during parsing.

## Usage Examples
These routines are central to the initialization phase of `wannier90.x`.

**Typical call sequence in `wannier_prog.F90` (on root node):**
```fortran
! In wannier_prog.F90, at the beginning of the program on the root node:
call w90_wannier90_readwrite_read(atom_data, band_plot, ..., seedname, stdout, error, comm)
if (allocated(error)) call prterr(error, stdout, stderr, comm) ! Handle error

call w90_wannier90_readwrite_write(atom_data, band_plot, ..., stdout) ! Print summary

call w90_wannier90_readwrite_memory_estimate(atom_data, ..., stdout) ! Print memory estimate

! Then, distribute parameters to other nodes:
call w90_wannier90_readwrite_dist(atom_data, band_plot, ..., error, comm)
if (allocated(error)) call prterr(error, stdout, stderr, comm)
```

**Writing a checkpoint file:**
```fortran
! In wannier_prog.F90, after a significant calculation stage (e.g., disentanglement)
if (on_root) then
  call w90_wannier90_readwrite_write_chkpt('postdis', exclude_bands, wannier_data, ...)
end if
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: Uses `dp`.
    - `w90_types`: Uses many derived types defined therein (e.g., `atom_data_type`, `kmesh_input_type`).
    - `w90_error`: For error handling.
    - `w90_readwrite`: This module relies heavily on the generic keyword and block reading routines from `w90_readwrite` (e.g., `w90_readwrite_in_file`, `w90_readwrite_get_keyword`). It also uses `w90_readwrite_dealloc` and specific `w90_readwrite_read_X` routines.
    - `w90_wannier90_types`: Uses derived types specific to `wannier90.x` (e.g., `w90_calculation_type`, `wann_control_type`).
    - `w90_utility`: For `utility_recip_lattice`, `utility_inverse_mat`, etc.
    - `w90_comms`: For `comms_bcast` in the `_dist` routine.
    - `w90_io`: For `post_proc_flag`.

- **External Libraries:** None directly.

- **Interactions:**
    - This module is the primary interface for `wannier90.x` to read its input parameters from `seedname.win`.
    - It populates almost all major data structures that control the behavior and hold the data for a Wannier90 calculation.
    - It writes and reads checkpoint files (`seedname.chk`) which allow for restarting calculations or transferring data.
    - Its `_dist` routine is essential for initializing MPI runs correctly.
```
