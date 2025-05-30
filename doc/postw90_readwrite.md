## Overview
The `w90_postw90_readwrite` module is responsible for handling input and output operations specifically tailored for the `postw90.x` executable. It extends the general I/O capabilities of `w90_readwrite` by parsing parameters unique to post-processing tasks like Berry curvature calculations, Boltzmann transport, DOS, k-space path/slice interpolations, and gyrotropic effects. It also includes routines for writing summaries of these parameters and estimating memory usage for `postw90.x` calculations.

## Key Components

### Module `w90_postw90_readwrite`
- **Description:** The main module for `postw90.x`-specific I/O.

### Type `pw90_extra_io_type`
- **Description:** A local derived type used for temporary storage of certain parameters read from the input file before they are assigned to their respective module-level types. This primarily includes frequency ranges for Kubo and gyrotropic calculations, global smearing parameters, and global k-mesh settings.
- **Fields:**
    - `gyrotropic_freq_min`, `gyrotropic_freq_max`, `gyrotropic_freq_step` (real(kind=dp))
    - `kubo_freq_min`, `kubo_freq_max`, `kubo_freq_step` (real(kind=dp))
    - `smear` (type `pw90_smearing_type`): Global smearing settings.
    - `global_kmesh` (type `kmesh_spacing_type`): Global k-mesh settings.
    - `global_kmesh_set` (logical): True if a global k-mesh is defined.
    - `boltz_2d_dir` (character(len=4)): Specifies 2D direction for Boltzmann transport.

### Subroutine `w90_postw90_readwrite_read`
- **Description:** The main input reading routine for `postw90.x`. It calls various specialized `w90_readwrite_read_X` (from the base `w90_readwrite` module) and `w90_wannier90_readwrite_read_X` (from `w90_wannier90_readwrite` for `wannier90.x` specific types that `postw90.x` might also need, though this seems to be a slight misnomer in this context, it should be `w90_postw90_readwrite_read_X` for postw90 specific types) routines to parse the `seedname.win` file. It populates numerous data structures controlling DOS, k-path/slice plots, Berry phase calculations, Boltzmann transport, gyrotropic effects, and general interpolation.
- **Parameters:** An extensive list of `intent(inout)` arguments corresponding to nearly all configurable parameters and data structures for `postw90.x` (e.g., `ws_region`, `w90_system`, `print_output`, `kmesh_input`, `dis_manifold`, `pw90_calculation`, `pw90_berry`, `pw90_boltzwann`, etc.).
- **Returns:** Populates all the output arguments with values read from the input file or with default values. Sets up error status if issues are encountered.

### Subroutine `w90_postw90_readwrite_write`
- **Description:** Writes a detailed summary of the parsed input parameters specific to `postw90.x` tasks to the main output file (`stdout`). This provides a comprehensive record of the settings used for the post-processing calculation.
- **Parameters:** Similar to `w90_postw90_readwrite_read`, but parameters are `intent(in)`.
- **Returns:** Prints formatted information to `stdout`.

### Subroutine `w90_postw90_readwrite_mem_estimate`
- **Description:** Estimates the memory usage for `postw90.x` calculations, focusing on the BoltzWann part if enabled.
- **Parameters:** `mem_param` (memory from base parameters), `mem_bw` (output for BoltzWann memory), `dis_manifold`, `do_boltzwann`, `pw90_boltzwann`, `spin_decomp`, `num_wann`, `stdout`.
- **Returns:** Prints memory estimates for BoltzWann.

### Specialized Reading Subroutines
This module contains numerous subroutines named `w90_wannier90_readwrite_read_X` (which should ideally be `w90_postw90_readwrite_read_X` for clarity as they handle `postw90.x` specific settings) to parse distinct blocks or keywords from the input file. Examples:
- `w90_wannier90_readwrite_read_pw90_calcs`: Reads flags for which `postw90.x` calculations to perform.
- `w90_wannier90_readwrite_read_effective_model`: Reads `effective_model` flag.
- `w90_wannier90_readwrite_read_oper`: Reads formatting flags for `.spn` and `.uHu` files.
- `w90_wannier90_readwrite_read_kslice`: Reads parameters for k-space slice plots.
- `w90_wannier90_readwrite_read_smearing`: Reads global smearing parameters into `pw90_extra_io%smear`.
- `w90_wannier90_readwrite_read_scissors_shift`: Reads `scissors_shift`.
- `w90_wannier90_readwrite_read_pw90spin`: Reads parameters for spin moment calculations.
- `w90_wannier90_readwrite_read_gyrotropic`: Reads parameters for gyrotropic effects.
- `w90_wannier90_readwrite_read_berry`: Reads parameters for Berry module tasks.
- `w90_wannier90_readwrite_read_spin_hall`: Reads parameters for spin Hall conductivity.
- `w90_wannier90_readwrite_read_pw90ham`: Reads parameters for k.p Hamiltonian calculations.
- `w90_wannier90_readwrite_read_pw90_kpath`: Reads parameters for k-path plotting.
- `w90_wannier90_readwrite_read_dos`: Reads parameters for DOS calculations.
- `w90_wannier90_readwrite_read_geninterp`: Reads parameters for generic k-point interpolation.
- `w90_wannier90_readwrite_read_boltzwann`: Reads parameters for BoltzWann calculations.
- `w90_wannier90_readwrite_read_energy_range`: Sets up energy/frequency ranges for various modules.
- `w90_wannier90_readwrite_read_global_kmesh`: Reads global k-mesh settings.
- `w90_wannier90_readwrite_read_local_kmesh`: Reads module-specific k-mesh settings, falling back to global if not set.

### Subroutine `get_module_kmesh`
- **Description:** A helper routine to get k-mesh settings (either from module-specific keywords like `dos_kmesh` or from global `kmesh`/`kmesh_spacing`) and convert spacing to mesh points if needed.

### Subroutine `parameters_gyro_write_task`
- **Description:** A small helper to format output lines for gyrotropic task flags.

## Important Variables/Constants
- **`pw90_extra_io_type`:** As described above, for temporary storage of global settings.
- The module primarily populates instances of derived types defined in `w90_postw90_types.F90` by reading from the `.win` file.

## Usage Examples
This module is called by the main `postw90.F90` program.

**Core parameter reading in `postw90.F90` (on root node):**
```fortran
! In postw90.F90
call w90_postw90_readwrite_read(ws_region, system, exclude_bands, verbose, wann_data, &
                                kmesh_data, kpt_latt, num_kpts, dis_window, fermi_energy_list, &
                                atoms, num_bands, num_wann, eigval, mp_grid, real_lattice, &
                                spec_points, pw90_calcs, postw90_oper, scissors_shift, &
                                effective_model, pw90_spin, pw90_ham, kpath, kslice, dos_data, &
                                berry, spin_hall, gyrotropic, geninterp, boltz, eig_found, &
                                write_data_pw90, gamma_only, physics%bohr, optimisation, stdout, &
                                seedname, error, comm)
! ... error check ...
! Then print summary
call w90_postw90_readwrite_write(verbose, system, fermi_energy_list, atoms, num_wann, &
                                 real_lattice, spec_points, pw90_calcs, postw90_oper, &
                                 scissors_shift, pw90_spin, kpath, kslice, dos_data, berry, &
                                 gyrotropic, geninterp, boltz, write_data_pw90, optimisation, stdout)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `maxlen`.
    - `w90_types`: For base data structures like `print_output_type`, `atom_data_type`.
    - `w90_readwrite`: Crucially uses the generic keyword/block reading routines from this base module.
    - `w90_postw90_types`: Defines all the `pw90_*_type` structures that this module populates.
    - `w90_error`: For error handling.
    - `w90_utility`: For `utility_recip_lattice`.
    - `w90_comms`: For `w90comm_type`.

- **External Libraries:** None.

- **Interactions:**
    - This module is the primary parser for `postw90.x` input from `seedname.win`.
    - It populates the various `pw90_*_type` data structures which then control the execution flow and parameters for different post-processing tasks (DOS, Berry, BoltzWann, etc.).
    - It interacts with `w90_readwrite` for the underlying file reading and keyword extraction logic.
```
