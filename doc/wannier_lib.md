## Overview
The `wannier_lib.F90` file provides a library interface to the core Wannier90 functionality. It allows external codes to call Wannier90 routines to perform wannierisation and related calculations without running the standalone `wannier90.x` executable. This is achieved primarily through two main subroutines: `wannier_setup` for initialization and parameter setup, and `wannier_run` for executing the main calculation steps. The file also defines two modules, `w90_libv1_types` and `w90_wannier90_libv1_types`, which appear to be legacy or specific containers for global variables used by the library mode.

**Note:** The comments in the code state that the library mode currently works ONLY in serial and that these library routines are considered obsolete.

## Key Components

### Module `w90_libv1_types`
- **Description:** A module that defines and saves global variables mirroring many of the core data structures found in `w90_types`. This seems to be a way to maintain state for the library mode.
- **Key Saved Variables:** `exclude_bands`, `num_bands`, `num_kpts`, `num_wann`, `optimisation`, `atoms` (type `atom_data_type`), `dis_window` (type `dis_manifold_type`), `kmesh_info`, `kmesh_data`, `verbose` (type `print_output_type`), `input_proj`, `system` (type `w90_system_type`), `wann_data`, `ws_region`, `fermi_energy_list`, `kpt_latt`, `eigval`, `u_matrix_opt`, `u_matrix`, `mp_grid`, `num_proj`, `real_lattice`, `spec_points`. Also contains a local error printing routine `prterr`.

### Module `w90_wannier90_libv1_types`
- **Description:** Similar to `w90_libv1_types`, this module defines and saves global variables, but these mirror data structures found in `w90_wannier90_types`.
- **Key Saved Variables:** `w90_calcs` (type `w90_calculation_type`), `out_files`, `rs_region` (type `real_space_ham_type`), `plot` (type `wvfn_read_type`), `band_plot`, `wann_plot`, `dis_data` (type `dis_control_type`), `dis_spheres`, `wannierise` (type `wann_control_type`), `wann_omega`, `lsitesymmetry`, `symmetrize_eps`, `fermi_surface_data`, `tran`, `select_proj`, `eig_found`, `a_matrix`, various `m_matrix` versions, `omega_invariant`.

### Subroutine `wannier_setup`
- **Description:** Initializes the Wannier90 library mode. It takes essential system parameters (lattice, k-points, atomic structure, etc.) from the calling program, reads additional parameters from the `seedname.win` input file (similar to `wannier90.x`), sets up the k-mesh, and then returns derived k-mesh information and projection details back to the calling program.
- **Parameters (Intent(in)):** `seed__name`, `mp_grid_loc`, `num_kpts_loc`, `real_lattice_loc`, `recip_lattice_loc`, `kpt_latt_loc`, `num_bands_tot`, `num_atoms_loc`, `atom_symbols_loc`, `atoms_cart_loc`, `gamma_only_loc`, `spinors_loc`.
- **Parameters (Intent(out)):** `nntot_loc`, `nnlist_loc`, `nncell_loc`, `num_bands_loc`, `num_wann_loc`, `proj_site_loc`, `proj_l_loc`, `proj_m_loc`, `proj_radial_loc`, `proj_z_loc`, `proj_x_loc`, `proj_zona_loc`, `exclude_bands_loc`, and optional `proj_s_loc`, `proj_s_qaxis_loc`.
- **Functionality:**
    1. Initializes MPI if not already done (expects to run on a single process).
    2. Opens `seedname.wout` for logging.
    3. Copies input parameters into global module variables.
    4. Calls `w90_wannier90_readwrite_read` to parse `seedname.win`.
    5. Calls `w90_wannier90_readwrite_write` to print parameters.
    6. Calls `kmesh_get` to determine k-point neighbor information.
    7. Copies the derived k-mesh and projection data back to the output arguments.
    8. If `postproc_setup` is true, calls `kmesh_write`.
    9. Performs deallocations.

### Subroutine `wannier_run`
- **Description:** Executes the main Wannier90 calculation (disentanglement, wannierisation, plotting, transport) using parameters previously set up by `wannier_setup` and matrices provided by the calling program.
- **Parameters (Intent(in)):** All parameters that were input to `wannier_setup`, plus `M_matrix_loc` (overlaps) and `A_matrix_loc` (projections), `eigenvalues_loc`.
- **Parameters (Intent(out)):** `U_matrix_loc` (final U matrix), and optional `U_matrix_opt_loc`, `lwindow_loc`, `wann_centres_loc`, `wann_spreads_loc`, `spread_loc`.
- **Functionality:**
    1. Initializes MPI (similar to `wannier_setup`).
    2. Re-reads parameters from `seedname.win` (this seems redundant if `wannier_setup` was called correctly and global variables are used, but follows the structure of `wannier_prog.F90`).
    3. Sets up k-mesh via `kmesh_get`.
    4. Allocates and populates overlap/projection matrices (`a_matrix`, `m_matrix`, etc.) from the input `_loc` arguments.
    5. If disentanglement is needed, calls `dis_main`. Otherwise, calls `overlap_project`.
    6. Calls `wann_main` (or `wann_main_gamma`) for the wannierisation procedure.
    7. Writes checkpoint file.
    8. If requested, calls `plot_main` for plotting.
    9. If requested, calls `tran_main` for transport.
    10. Copies results (U matrices, centers, spreads) to output arguments.
    11. Performs deallocations.

## Important Variables/Constants
The key variables are those stored in the `w90_libv1_types` and `w90_wannier90_libv1_types` modules. These modules essentially make most of the Wannier90 internal state global for the duration of the library call.

- **Seedname:** `seed__name` (input to library calls), `seedname` (internal).
- **Input data from calling code:** `mp_grid_loc`, `num_kpts_loc`, `real_lattice_loc`, `kpt_latt_loc`, `num_bands_tot`, `atoms_cart_loc`, `M_matrix_loc`, `A_matrix_loc`, `eigenvalues_loc`, etc.
- **Output data to calling code:** `nntot_loc`, `nnlist_loc`, `nncell_loc` (from `wannier_setup`); `U_matrix_loc`, `wann_centres_loc`, `wann_spreads_loc` (from `wannier_run`).

## Usage Examples
The library is intended to be called from another Fortran program. A typical workflow would be:

1.  **Calling Program:** Prepare all necessary input data (lattice, k-points, atom positions, $M$ and $A$ matrices, eigenvalues).
2.  **Call `wannier_setup`:**
    ```fortran
    call wannier_setup('myseed', mp_grid, num_kpts, real_lattice, recip_lattice, &
                       kpt_latt, num_bands_total, num_atoms, atom_symbols, atoms_cart, &
                       is_gamma_only, is_spinors, nntot, nnlist, nncell, &
                       num_bands_wannier, num_wann_out, proj_site, proj_l, proj_m, &
                       proj_radial, proj_z, proj_x, proj_zona, exclude_bands_indices, &
                       proj_s, proj_s_qaxis)
    ! The calling program now has k-mesh info (nntot, nnlist, nncell) and
    ! details about projections needed (proj_*)
    ```
3.  **Calling Program:** Uses the information from `wannier_setup` (e.g., `nntot`) if needed. Prepares $M$, $A$, and eigenvalue arrays according to Wannier90's requirements.
4.  **Call `wannier_run`:**
    ```fortran
    call wannier_run('myseed', mp_grid, num_kpts, real_lattice, recip_lattice, &
                     kpt_latt, num_bands_wannier, num_wann_out, nntot, num_atoms, &
                     atom_symbols, atoms_cart, is_gamma_only, M_matrix_for_wannier, &
                     A_matrix_for_wannier, eigenvalues_for_wannier, U_matrix_final, &
                     U_matrix_opt_final, lwindow_final, wann_centres_final, &
                     wann_spreads_final, spread_final)
    ! The calling program now has the results of the wannierisation.
    ```
A test example `test_library.F90` is mentioned in the source code comments.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `w90_physical_constants_type`.
    - `w90_types`: For most of the derived data type definitions.
    - `w90_wannier90_types`: For `wannier90.x`-specific derived data types.
    - `w90_io`: For file I/O, timing, version string.
    - `w90_readwrite`: For parsing `seedname.win` and some shared I/O tasks.
    - `w90_wannier90_readwrite`: For `wannier90.x`-specific parsing and checkpointing.
    - `w90_kmesh`: For `kmesh_get`, `kmesh_write`, `kmesh_dealloc`.
    - `w90_overlap`: For `overlap_allocate`, `overlap_project`, `overlap_dealloc`.
    - `w90_disentangle`: For `dis_main`.
    - `w90_wannierise`: For `wann_main`, `wann_main_gamma`.
    - `w90_hamiltonian`: For `hamiltonian_setup`, `hamiltonian_get_hr`, `hamiltonian_dealloc`.
    - `w90_plot`: For `plot_main`.
    - `w90_transport`: For `tran_main`.
    - `w90_sitesym`: For site symmetry operations (if `lsitesymmetry` is true).
    - `w90_comms`: For MPI related types and basic MPI status calls (though the library is stated to be serial).
    - `w90_error`: For error handling types and routines.

- **External Libraries:**
    - **MPI:** The code includes MPI headers and makes MPI initialization checks, but the library is noted as serial-only. This suggests it might be compiled with MPI support but expects to run on a single process when used as a library.

- **Interactions:**
    - This module acts as a wrapper around the main computational machinery of Wannier90.
    - It reads configuration from `seedname.win` like the standalone executable.
    - It takes core physical data (lattice, k-points, matrices $M$ and $A$, eigenvalues) directly from the calling program's memory instead of from intermediate files like `.mmn`, `.amn`, `.eig`.
    - It returns key results (U-matrices, Wannier centers, spreads) directly to the calling program.
    - It still produces standard Wannier90 output files like `seedname.wout`, `seedname.chk`, and plotting outputs if requested.
```
