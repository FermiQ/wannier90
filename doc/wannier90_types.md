## Overview
The `w90_wannier90_types` module defines derived data types that are specific to the main `wannier90.x` executable. These types are not typically used by the post-processing tool `postw90.x`. They encapsulate parameters and data structures related to various aspects of the Wannier function calculation, such as control over different calculation phases (disentanglement, wannierisation), plotting options, real-space Hamiltonian settings, transport calculation parameters, and site symmetry information. This module complements `w90_types` which defines types shared by both `wannier90.x` and `postw90.x`.

## Key Components

### Module `w90_wannier90_types`
- **Description:** The central module for defining `wannier90.x`-specific derived data types.

### Type `w90_calculation_type`
- **Description:** Contains logical flags to control the overall execution path and which parts of the calculation are performed.
- **Fields:**
    - `postproc_setup` (logical): If true, run in pre-processing mode (e.g., to write `.nnkp` file).
    - `restart` (character(len=20)): Specifies the restart mode (e.g., 'default', 'wannierise', 'plot').
    - `bands_plot` (logical): If true, calculate and plot interpolated band structure.
    - `wannier_plot` (logical): If true, plot Wannier functions.
    - `fermi_surface_plot` (logical): If true, calculate and plot Fermi surface.
    - `transport` (logical): If true, perform transport calculations.

### Type `output_file_type`
- **Description:** Contains logical flags that control which output files are written.
- **Fields:** (all logical)
    - `write_hr`: Write real-space Hamiltonian $H(\mathbf{R})$.
    - `write_r2mn`: Write matrix elements of $r^2$.
    - `write_proj`: Write projections.
    - `write_hr_diag`: Write diagonal elements of $H(\mathbf{R})$.
    - `write_vdw_data`: Write data for Van der Waals C6 coefficient calculations.
    - `write_u_matrices`: Write $U$ and $U_{opt}$ matrices.
    - `write_bvec`: Write $b$-vectors.
    - `write_rmn`: Write matrix elements of $\mathbf{r}$.
    - `write_tb`: Write tight-binding model parameters.
    - `write_xyz`: Write Wannier function centers in XYZ format.

### Type `real_space_ham_type`
- **Description:** Parameters controlling the real-space Hamiltonian $H(\mathbf{R})$.
- **Fields:**
    - `hr_cutoff` (real(kind=dp)): Value below which $H(\mathbf{R})$ elements are zeroed.
    - `dist_cutoff` (real(kind=dp)): Distance beyond which $H(\mathbf{R})$ elements are zeroed.
    - `dist_cutoff_mode` (character(len=20)): Mode for applying `dist_cutoff` ('one_dim', 'two_dim', 'three_dim').
    - `dist_cutoff_hc` (real(kind=dp)): Separate distance cutoff for the conductor part in LCR transport.
    - `one_dim_dir` (integer): Specifies the transport direction for 1D systems.
    - `system_dim` (integer): Dimensionality of the system (1, 2, or 3).
    - `translate_home_cell` (logical): If true, translate Wannier functions to the home unit cell.
    - `translation_centre_frac(3)` (real(kind=dp)): Fractional coordinates of the center for WF translation.
    - `automatic_translation` (logical): If true, automatically determine translation center.

### Type `band_plot_type`
- **Description:** Parameters for band structure plotting.
- **Fields:**
    - `mode` (character(len=20)): Mode for band interpolation (e.g., 's-k', 'cut').
    - `format` (character(len=20)): Output format (e.g., 'gnuplot', 'xmgrace').
    - `project(:)` (integer, allocatable): List of Wannier function indices to project bands onto.

### Type `wannier_plot_type`
- **Description:** Parameters for plotting Wannier functions.
- **Fields:**
    - `list(:)` (integer, allocatable): List of Wannier function indices to plot.
    - `supercell(3)` (integer): Supercell dimensions for plotting.
    - `radius` (real(kind=dp)): Radius for cube file generation.
    - `scale` (real(kind=dp)): Scale factor for cube file atom display.
    - `format` (character(len=20)): Output format ('xcrysden', 'cube').
    - `mode` (character(len=20)): Plotting mode ('crystal', 'molecule').
    - `spinor_mode` (character(len=20)): For spinor WFs ('total', 'up', 'down').
    - `spinor_phase` (logical): If true, include phase for spinor components.

### Type `wvfn_read_type`
- **Description:** Parameters for reading wavefunction files (UNK files).
- **Fields:**
    - `formatted` (logical): If true, UNK files are formatted; otherwise binary.
    - `spin_channel` (integer): Spin channel to read (1 for up, 2 for down).

### Type `dis_control_type`
- **Description:** Control parameters for the disentanglement procedure.
- **Fields:**
    - `num_iter` (integer): Maximum number of disentanglement iterations.
    - `mix_ratio` (real(kind=dp)): Mixing ratio for updating the Z-matrix.
    - `conv_tol` (real(kind=dp)): Convergence tolerance for $\Omega_I$.
    - `conv_window` (integer): Number of iterations over which `conv_tol` must be met.

### Type `dis_spheres_type`
- **Description:** Parameters for defining k-space spheres used in selective disentanglement.
- **Fields:**
    - `first_wann` (integer): Index of the first Wannier function to consider for sphere-based disentanglement.
    - `num` (integer): Number of k-space spheres.
    - `spheres(:, :)` (real(kind=dp), allocatable): Array storing sphere centers (3 components) and radii (1 component).

### Type `wann_slwf_type` (Selective Localisation / Constrained Centres)
- **Description:** Parameters for selective localization and constraining Wannier function centers.
- **Fields:**
    - `slwf_num` (integer): Number of objective Wannier functions (others excluded from spread).
    - `selective_loc` (logical): If true, perform selective localization.
    - `constrain` (logical): If true, constrain WF centers.
    - `centres(:, :)` (real(kind=dp), allocatable): Cartesian coordinates of constrained centers.
    - `lambda` (real(kind=dp)): Lagrange multiplier for center constraints.

### Type `guiding_centres_type`
- **Description:** Parameters for using guiding centers to control phases during wannierisation.
- **Fields:**
    - `enable` (logical): If true, use guiding centers.
    - `num_guide_cycles` (integer): Iterations between applying guiding center logic.
    - `num_no_guide_iter` (integer): Initial iterations before starting guiding centers.
    - `centres(:, :)` (real(kind=dp), allocatable): Coordinates of guiding centers (often from projections).

### Type `wann_control_type`
- **Description:** Control parameters for the main wannierisation (spread minimization) procedure.
- **Fields:**
    - `num_dump_cycles` (integer): Frequency to write checkpoint files.
    - `num_print_cycles` (integer): Frequency to print iteration information.
    - `num_iter` (integer): Maximum number of wannierisation iterations.
    - `num_cg_steps` (integer): Number of conjugate gradient steps before reset.
    - `conv_tol` (real(kind=dp)): Convergence tolerance for the spread.
    - `conv_window` (integer): Number of iterations for convergence check.
    - `guiding_centres` (type(guiding_centres_type)): Nested type for guiding center parameters.
    - `fixed_step` (real(kind=dp)): Fixed step length for minimization (if > 0).
    - `trial_step` (real(kind=dp)): Initial trial step for line search.
    - `precond` (logical): If true, use preconditioning.
    - `lfixstep` (logical): Derived flag, true if `fixed_step > 0`.
    - `conv_noise_amp` (real(kind=dp)): Amplitude for adding noise if convergence stalls.
    - `conv_noise_num` (integer): Number of iterations to apply noise.
    - `constrain` (type(wann_slwf_type)): Nested type for selective localisation/constrained centres.

### Type `wann_omega_type`
- **Description:** Stores the total spread and its decomposition.
- **Fields:**
    - `invariant` (real(kind=dp)): Gauge-invariant part of the spread ($\Omega_I$).
    - `total` (real(kind=dp)): Total spread ($\Omega$).
    - `tilde` (real(kind=dp)): Gauge-dependent part of the spread ($\tilde{\Omega} = \Omega_D + \Omega_{OD}$).

### Type `fermi_surface_plot_type`
- **Description:** Parameters for Fermi surface plotting.
- **Fields:**
    - `num_points` (integer): Number of points along each direction of the k-grid for Fermi surface.
    - `plot_format` (character(len=20)): Output format (e.g., 'xcrysden').

### Type `transport_type`
- **Description:** Parameters for quantum transport calculations.
- **Fields:** (Many fields, including those listed in `src/transport.F90` header)
    - `easy_fix` (logical): Flag for a simple parity fix in LCR.
    - `mode` (character(len=20)): 'bulk' or 'lcr'.
    - `win_min`, `win_max`, `energy_step`: Energy window for transport.
    - `num_bb`, `num_ll`, `num_rr`, `num_cc`, `num_lc`, `num_cr`: WF counts for LCR.
    - `num_bandc`: Bandwidth for conductor Hamiltonian.
    - `write_ht`, `read_ht`: Flags for reading/writing Hamiltonian blocks.
    - `use_same_lead` (logical): If true, left and right leads are identical in LCR.
    - `num_cell_ll`, `num_cell_rr`: Number of unit cells in lead principal layers.
    - `group_threshold` (real(kind=dp)): Threshold for grouping WFs in LCR.

### Type `select_projection_type`
- **Description:** Parameters for selecting a subset of initial projections.
- **Fields:**
    - `lselproj` (logical): If true, select a subset of projections.
    - `proj2wann_map(:)` (integer, allocatable): Map from original projection index to Wannier function index.

### Type `sitesym_type`
- **Description:** Parameters related to site symmetry operations.
- **Fields:**
    - `nkptirr` (integer): Number of irreducible k-points.
    - `nsymmetry` (integer): Number of symmetry operations.
    - `kptsym(:, :)` (integer, allocatable): Map from (isym, ik_irr) to ik_sym.
    - `ir2ik(:)` (integer, allocatable): Map from irreducible k-point index to full k-point index.
    - `ik2ir(:)` (integer, allocatable): Map from full k-point index to irreducible k-point index.
    - `symmetrize_eps` (real(kind=dp)): Tolerance for symmetry condition checks.
    - `d_matrix_band(:, :, :, :)` (complex(kind=dp), allocatable): Representation matrices for Bloch bands.
    - `d_matrix_wann(:, :, :, :)` (complex(kind=dp), allocatable): Representation matrices for Wannier functions.

### Type `ham_logical_type`
- **Description:** Logical flags related to the status and computation of the Hamiltonian.
- **Fields:** (all logical)
    - `ham_have_setup`: True if Hamiltonian setup is done.
    - `have_ham_k`: True if $H(\mathbf{k})$ is computed.
    - `have_ham_r`: True if $H(\mathbf{R})$ is computed.
    - `have_translated`: True if Wannier centers have been translated.
    - `hr_written`: True if $H(\mathbf{R})$ has been written to file.
    - `tb_written`: True if tight-binding file has been written.
    - `use_translation`: True if WFs should be translated for $H(\mathbf{R})$ calculation.

## Important Variables/Constants
- **`dp` (integer, parameter, from `w90_constants`):** Defines the kind for double precision real numbers.
- **`maxlen` (integer, parameter, from `w90_constants`):** Default maximum length for character strings.

## Usage Examples
These types are primarily used within `wannier90.x` to structure the vast number of parameters and data arrays. They are populated by reading the `seedname.win` input file (via `w90_wannier90_readwrite` and `w90_readwrite` modules) and then passed as arguments to various computational and output routines.

```fortran
! Example of declaring a variable of wann_control_type
use w90_wannier90_types, only: wann_control_type
type(wann_control_type) :: wannierisation_params

! Accessing a nested field
wannierisation_params%guiding_centres%enable = .true.
wannierisation_params%num_iter = 200
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: Uses `dp` for precision and `maxlen` for string lengths.

- **External Libraries:** None.

- **Interactions:**
    - This module is central to `wannier90.x`, providing the data structures that hold most of its configuration and runtime data.
    - It is closely linked with `w90_types` (for shared types) and `w90_wannier90_readwrite` (which populates instances of these types from the input file).
    - Many other modules in `wannier90.x` (e.g., `w90_wannierise`, `w90_disentangle`, `w90_plot`, `w90_transport`) take arguments of these types to control their behavior and access necessary data.
```
