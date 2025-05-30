## Overview
The `w90_plot` module in Wannier90 is responsible for generating various kinds of output data that can be used for plotting and visualization. This includes interpolated band structures along specified k-point paths, Fermi surfaces, and the Wannier functions themselves in formats suitable for visualization software like XCrysDen or for cube file processing. It also provides routines to write out the $U$ matrices (both from disentanglement and wannierisation) and $b$-vectors.

## Key Components

### Module `w90_plot`
- **Description:** The main module containing all plotting-related subroutines.

### Subroutine `plot_main`
- **Description:** This is the primary driver routine for all plotting functionalities. Based on flags set in the input file (e.g., `bands_plot`, `fermi_surface_plot`, `wannier_plot`, `write_hr`, `write_u_matrices`), it calls the appropriate specialized subroutines to generate the requested data. It handles the setup of the Hamiltonian in the Wannier basis if needed for band interpolation or Fermi surface plotting.
- **Parameters:** Takes a large number of arguments, including data structures for atomic positions (`atom_data`), band plotting parameters (`band_plot`), disentanglement manifold (`dis_manifold`), Fermi energies (`fermi_energy_list`), k-mesh info (`kmesh_info`), k-point path (`kpoint_path`), Wannier function data (`wannier_data`), Hamiltonian matrices (`ham_k`, `ham_r`), $U$ matrices (`u_matrix`, `u_matrix_opt`), eigenvalues (`eigval`), lattice vectors (`real_lattice`), and various control flags and dimensions.
- **Returns:** Generates various output files depending on the input flags.

### Subroutine `plot_interpolate_bands`
- **Description:** Calculates and writes the interpolated band structure along a specified k-point path. It uses the real-space Hamiltonian $H(\mathbf{R})$ to construct the Hamiltonian $H(\mathbf{k})$ at arbitrary k-points along the path, diagonalizes it, and writes the eigenvalues to output files. It can also project the bands onto selected Wannier functions. Output can be formatted for Gnuplot (`_band.dat`, `_band.gnu`) and/or Xmgrace (`_band.agr`). It also generates a `_band.kpt` file listing the k-points along the path and `_band.labelinfo.dat` with information about high-symmetry points.
- **Parameters:** Includes `mp_grid`, `real_lattice`, `band_plot` (controls format, projection), `kpoint_path`, `real_space_ham` (controls $H(R)$ cutoff), `num_wann`, `wannier_data`, `ham_r`, `irvec`, `ndegen`, `nrpts`, `wannier_centres_translated`, `seedname`.
- **Contains:**
    - `plot_interpolate_gnuplot`: Formats band structure data for Gnuplot.
    - `plot_interpolate_xmgrace`: Formats band structure data for Xmgrace.
    - `plot_cut_hr`: Applies cutoffs to the real-space Hamiltonian $H(\mathbf{R})$ based on distance or magnitude, which can be useful for plotting bands from truncated Hamiltonians.

### Subroutine `plot_fermi_surface`
- **Description:** Generates data for plotting the Fermi surface in the Xcrysden `.bxsf` format. It calculates eigenvalues on a 3D grid of k-points using the interpolated Hamiltonian and writes them out, along with the Fermi energy, so that Fermi surface visualization tools can render the surface.
- **Parameters:** `fermi_energy_list`, `recip_lattice`, `fermi_surface_plot` (controls grid density), `num_wann`, `ham_r`, `irvec`, `ndegen`, `nrpts`, `seedname`.

### Subroutine `plot_wannier`
- **Description:** Generates data for plotting the Wannier functions in real space. It reads the Bloch wavefunctions (UNK files), transforms them to Wannier functions using the $U$ matrices (and $U_{opt}$ if disentangled), and writes them out, typically in XCrysDen `.xsf` format or Gaussian `.cube` format.
- **Parameters:** `wannier_plot` (controls format, supercell size, which WFs to plot), `wvfn_read` (controls UNK file format), `wannier_data`, `u_matrix_opt`, `dis_manifold`, `real_lattice`, `atom_data`, `kpt_latt`, `u_matrix`, `num_kpts`, `num_bands`, `num_wann`, `have_disentangled`, `spinors`, `bohr`, `seedname`.
- **Contains:**
    - `internal_cube_format`: Writes Wannier functions in Gaussian cube format.
    - `internal_xsf_format`: Writes Wannier functions in XCrysDen XSF format.

### Subroutine `plot_u_matrices`
- **Description:** Writes the $U$ matrices to formatted text files. If disentanglement was performed, `u_matrix_opt` (from disentanglement) is written to `seedname_u_dis.mat`, and `u_matrix` (from wannierisation) is written to `seedname_u.mat`.
- **Parameters:** `u_matrix_opt`, `u_matrix`, `kpt_latt`, `have_disentangled`, `num_wann`, `num_kpts`, `num_bands`, `seedname`.

### Subroutine `plot_bvec`
- **Description:** Writes the $b$-vectors ($\mathbf{b_k}$) and their weights ($w_b$) to a file named `seedname.bvec`. This file can be used by other codes, such as EPW, for calculating electron-phonon coupling or transport properties.
- **Parameters:** `kmesh_info`, `num_kpts`, `seedname`, `error`, `comm`.

## Important Variables/Constants
This module primarily uses data passed as arguments, which are defined and described in other modules (e.g., `w90_wannier90_types`, `w90_hamiltonian`, `w90_kmesh`). Key input flags that control its behavior (read from `seedname.win`) include:
- `bands_plot` (logical): If true, call `plot_interpolate_bands`.
- `bands_plot_format` (character): 'gnuplot', 'xmgrace', or both.
- `fermi_surface_plot` (logical): If true, call `plot_fermi_surface`.
- `wannier_plot` (logical): If true, call `plot_wannier`.
- `wannier_plot_list` (integer array): List of Wannier functions to plot.
- `wannier_plot_supercell` (integer array(3)): Supercell dimensions for plotting WFs.
- `write_hr` (logical): If true, write $H(\mathbf{R})$ via `hamiltonian_write_hr`.
- `write_rmn` (logical): If true, write $\langle 0n | \mathbf{r} | Rm \rangle$ via `hamiltonian_write_rmn`.
- `write_tb` (logical): If true, write tight-binding model file via `hamiltonian_write_tb`.
- `write_u_matrices` (logical): If true, call `plot_u_matrices`.
- `write_bvec` (logical): If true, call `plot_bvec`.

## Usage Examples
The `plot_main` subroutine is typically called once near the end of a Wannier90 calculation, after the Wannier functions have been obtained (or if only post-processing of $H(\mathbf{R})$ is requested).

```fortran
! In wannier_prog.F90, after wannierisation or if restarting for plotting:
if (w90_calculation%bands_plot .or. w90_calculation%fermi_surface_plot .or. &
    output_file%write_hr .or. w90_calculation%wannier_plot .or. &
    output_file%write_u_matrices .or. output_file%write_tb .or. &
    output_file%write_bvec) then

  call plot_main(atom_data, band_plot, dis_manifold, fermi_energy_list, fermi_surface_plot, &
                 ham_logical, kmesh_info, kpt_latt, output_file, wvfn_read, real_space_ham, &
                 kpoint_path, print_output, wannier_data, wannier_plot, ws_region, &
                 w90_calculation, ham_k, ham_r, m_matrix, u_matrix, u_matrix_opt, eigval, &
                 real_lattice, wannier_centres_translated, physics%bohr, irvec, mp_grid, ndegen, &
                 shift_vec, nrpts, num_bands, num_kpts, num_wann, rpt_origin, &
                 transport%mode, have_disentangled, lsitesymmetry, w90_system%spinors, &
                 seedname, stdout, timer, error, comm)
  if (allocated(error)) call prterr(error, stdout, stderr, comm)
end if
```
The user specifies which plots to generate via flags in the `seedname.win` input file.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_comms`: For MPI communication (primarily `comms_reduce` in `plot_wannier`).
    - `w90_constants`: For `dp`, `cmplx_0`, `twopi`, `eps6`.
    - `w90_hamiltonian`: Uses `hamiltonian_get_hr`, `hamiltonian_write_hr`, `hamiltonian_setup`, `hamiltonian_write_rmn`, `hamiltonian_write_tb` if these outputs or band interpolation are requested.
    - `w90_io`: For timing, file unit management, and date functions.
    - `w90_types`: Defines various data structures passed as arguments.
    - `w90_utility`: For `utility_recip_lattice_base`, `utility_metric`.
    - `w90_wannier90_types`: Defines various data structures.
    - `w90_ws_distance`: For `ws_translate_dist`, `ws_write_vec` if distance-based cutoffs or outputs are used.
    - `w90_error`: For error handling.

- **External Libraries:**
    - **LAPACK/BLAS:** `ZHPEVX` (complex Hermitian packed eigenvalue problem solver) is used in `plot_interpolate_bands` and `plot_fermi_surface` to diagonalize the interpolated Hamiltonian at each k-point. `zaxpy` (complex vector addition) is used in `plot_wannier`.

- **Interactions:**
    - Relies on data computed in other modules:
        - `w90_hamiltonian` for $H(\mathbf{R})$ (`ham_r`) if band structure or Fermi surface is plotted.
        - `w90_wannierise` for the final $U$ matrices (`u_matrix`) and Wannier function centres (`wannier_data%centres`).
        - `w90_disentangle` for `u_matrix_opt` if disentanglement was performed.
        - `w90_kmesh` for `kmesh_info` (used in `plot_bvec` and when calling `hamiltonian_write_rmn/tb`).
    - Reads Bloch wavefunction data from `UNK` files if `wannier_plot` is true. These files are generated by the ab initio code.
    - Produces various output files (`_band.dat`, `_band.gnu`, `_band.agr`, `_band.kpt`, `.bxsf`, `_u.mat`, `_u_dis.mat`, `.xsf`, `.cube`, `.bvec`) intended for visualization or further analysis by the user or other programs.
```
