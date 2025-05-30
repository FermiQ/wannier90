## Overview
The `w90_berry` module in the `postw90` part of Wannier90 is dedicated to calculating various physical properties related to Berry phases and Berry curvatures in k-space. These properties are computed using the Maximally Localised Wannier Functions (MLWFs) obtained from a `wannier90.x` run. The module can calculate:
-   Anomalous Hall Conductivity (AHC)
-   Orbital Magnetization (morb)
-   Complex Optical Conductivity (Kubo-Greenwood formula) and Joint Density of States (JDOS)
-   Nonlinear Shift Current (sc)
-   Spin Hall Conductivity (SHC)
-   $\mathbf{k} \cdot \mathbf{p}$ expansion coefficients

It relies on matrix elements of position ($\mathbf{r}$) and Hamiltonian ($H$) in the Wannier basis, which are Fourier transformed to obtain k-space quantities.

## Key Components

### Module `w90_berry`
- **Description:** The main module for Berry phase related property calculations.

### Subroutine `berry_main`
- **Description:** This is the primary driver routine for all calculations within the `w90_berry` module. It determines which properties to calculate based on the `pw90_berry%task` input string. It then calls the appropriate `get_R_oper` routines (from `w90_get_oper`) to construct the necessary real-space matrix elements (e.g., $\langle \mathbf{0}n | \mathbf{r} | \mathbf{R}m \rangle$, $\langle \mathbf{0}n | H | \mathbf{R}m \rangle$) if they are not already available. Subsequently, it iterates over a k-mesh (either a regular Monkhorst-Pack grid or a user-supplied list from `kpoint.dat`) and calls specific `berry_get_*_klist` routines to compute the k-dependent integrands. Finally, it sums/integrates these contributions and writes the results to output files.
- **Parameters:** A large number of arguments, including:
    - `pw90_berry` (type `pw90_berry_mod_type`): Controls which Berry properties to calculate, k-mesh for interpolation, energy smearing, etc.
    - `dis_manifold`, `kmesh_info`, `kpoint_dist`, `pw90_band_deriv_degen`, `pw90_oper_read`, `pw90_spin`, `physics`, `ws_region`, `pw90_spin_hall`, `wannier_data`, `ws_distance`, `wigner_seitz`, `print_output`: Various data structures holding system, Wannier, and control information.
    - `AA_R`, `BB_R`, `CC_R`, `HH_R`, `SH_R`, `SHR_R`, `SR_R`, `SS_R`, `SAA_R`, `SBB_R`: Allocatable arrays for real-space matrix elements of position, Hamiltonian, and spin operators.
    - `u_matrix`, `v_matrix`: Unitary matrices from wannierisation (if effective model).
    - `eigval`: Bloch eigenvalues.
    - `real_lattice`, `scissors_shift`, `mp_grid`, `fermi_n`, `num_wann`, `num_kpts`, `num_bands`, `num_valence_bands`: System and dimension parameters.
    - `effective_model`, `have_disentangled`, `spin_decomp`: Logical flags.
    - `seedname`, `stdout`, `timer`, `error`, `comm`.
- **Returns:** Generates various output files (e.g., `seedname-ahc.dat`, `seedname-morb.dat`, `seedname-kubo_S_xx.dat`, `seedname-shc.dat`) depending on the tasks selected.

### Subroutine `berry_get_imf_klist`
- **Description:** Calculates the k-point resolved contribution to the anomalous Hall conductivity (AHC) and orbital magnetization. Specifically, it computes the integrand related to $-2\text{Im}[f(\mathbf{k})]$ (where $f$ is related to Berry connection and Hamiltonian matrix elements) for a list of Fermi energies.
- **Parameters:** Includes `fermi_energy_list`, `kpt` (current k-point), `AA_R`, `HH_R`, `u_matrix`, `v_matrix`, `eigval`, output `imf_k_list`.
- **Returns:** Populates `imf_k_list` with the k-point contribution to the AHC integrand.

### Subroutine `berry_get_imfgh_klist`
- **Description:** Calculates k-point resolved contributions for orbital magnetization, which involves three terms: $-2\text{Im}[f(\mathbf{k})]$, $-2\text{Im}[g(\mathbf{k})]$, and $-2\text{Im}[h(\mathbf{k})]$ as defined in CTVR06 and LVTS12.
- **Parameters:** Similar to `berry_get_imf_klist`, but also involves `BB_R` and `CC_R` matrix elements, and outputs `img_k_list` and `imh_k_list` in addition to `imf_k_list`.

### Subroutine `berry_get_kubo_k`
- **Description:** Computes the k-point contribution to the complex optical conductivity tensor $\sigma_{\alpha\beta}(\omega)$ using the Kubo-Greenwood formula. It calculates both Hermitian and anti-Hermitian parts. It also computes the joint density of states (JDOS). If `spin_decomp` is true, it provides spin-decomposed contributions.
- **Parameters:** Includes `pw90_berry` (for frequency list, smearing), `AA_R`, `HH_R`, `SS_R` (if spin decomposed), `kpt`, output arrays `kubo_H_k`, `kubo_AH_k`, `jdos_k` (and their `_spn` counterparts).

### Subroutine `berry_get_sc_klist`
- **Description:** Calculates the k-point contribution to the nonlinear shift current tensor $\sigma_{abc}(\omega)$.
- **Parameters:** Includes `kmesh_info`, `AA_R`, `HH_R`, `kpt`, output `sc_k_list`.

### Subroutine `berry_get_shc_klist`
- **Description:** Computes the k-point contribution to the spin Hall conductivity. It can use different formalisms (Qiao or Ryoo) depending on `pw90_spin_hall%method`.
- **Parameters:** Includes `pw90_spin_hall`, `AA_R`, `HH_R`, `SH_R`, `SHR_R`, `SR_R`, `SS_R`, `SAA_R`, `SBB_R`, `kpt`, and outputs `shc_k_fermi` (for Fermi energy scan) or `shc_k_freq` (for frequency scan).

### Subroutine `berry_get_kdotp`
- **Description:** Extracts $\mathbf{k} \cdot \mathbf{p}$ expansion coefficients up to second order around a specified k-point (usually Gamma) using quasi-degenerate perturbation theory adapted for Wannier functions.
- **Parameters:** `pw90_berry` (for `kdotp_kpoint`, `kdotp_bands`), `HH_R`, `kpt` (expansion center), output `kdotp`.

### Subroutine `berry_print_progress`
- **Description:** A utility to print the progress of k-point loop calculations to standard output.

## Important Variables/Constants

- **`pw90_berry_mod_type` (from `postw90_types.F90`):**
    - `task` (character): String specifying tasks (e.g., 'ahc', 'morb', 'kubo', 'sc', 'shc', 'kdotp').
    - `kmesh%mesh(3)` (integer): Dimensions of the interpolation k-mesh.
    - `kubo_nfreq` (integer): Number of frequency points for Kubo/SC/SHC calculations.
    - `kubo_freq_list(:)` (complex(dp)): List of frequencies $\hbar\omega + i\eta_{smr}$.
    - `kubo_smearing%type_index` (integer): Type of smearing for Kubo/SC/SHC.
    - `curv_adpt_kmesh` (integer): Factor for adaptive k-mesh refinement for AHC/SHC.
    - `curv_adpt_kmesh_thresh` (real(dp)): Threshold for triggering adaptive refinement.
- **`pw90_spin_hall_type` (from `postw90_types.F90`):**
    - `method` (character): Method for SHC ('qiao' or 'ryoo').
    - `alpha`, `beta`, `gamma` (integer): Cartesian component indices for $\sigma_{\alpha\beta}^{\gamma}$.
    - `freq_scan` (logical): If true, scan SHC vs. frequency; else vs. Fermi energy.
- Matrix elements in real space (e.g., `HH_R`, `AA_R`, `SS_R`) which are Fourier transformed.
- Output arrays for k-resolved and integrated quantities (e.g., `imf_k_list`, `ahc_list`, `kubo_H`, `sc_list`, `shc_fermi`).
- `alpha_A`, `beta_A` (integer parameters): Define pairs for antisymmetric tensor components (e.g., (y,z) for x).
- `alpha_S`, `beta_S` (integer parameters): Define pairs for symmetric tensor components (e.g., (x,y) for xy).

## Usage Examples
This module is invoked by `postw90.x` when `berry = .true.` is set in the `seedname.win` input file. Specific tasks and their parameters are also set in the input file.

**Example `seedname.win` section for AHC:**
```
berry = .true.
berry_task = 'ahc'
berry_kmesh = 20 20 20 ! Interpolation k-mesh
fermi_energy = 0.0 ! Or use fermi_energy_min/max/step for a scan
```

**Example for Kubo optical conductivity:**
```
berry = .true.
berry_task = 'kubo'
berry_kmesh = 30 30 30
kubo_freq_min = 0.0
kubo_freq_max = 5.0
kubo_freq_step = 0.01
kubo_smr_fixed_en_width = 0.1 ! Smearing width in eV
fermi_energy = 0.0
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_i`, `pi`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication (`comms_reduce`, `mpirank`, `mpisize`).
    - `w90_io`: For file I/O and timing.
    - `w90_types`: For shared data structures (`print_output_type`, `wannier_data_type`, etc.).
    - `w90_postw90_types`: For `postw90`-specific types like `pw90_berry_mod_type`.
    - `w90_utility`: For `utility_recip_lattice_base`, matrix operations, smearing functions.
    - `w90_get_oper`: Crucial for obtaining the real-space matrix elements of Hamiltonian and position operators (`get_HH_R`, `get_AA_R`, etc.).
    - `w90_postw90_common`: For Fourier transform routines (`pw90common_fourier_R_to_k_vec`, etc.) and occupation calculation (`pw90common_get_occ`).
    - `w90_spin`: For `spin_get_nk` if `spin_decomp` is true for Kubo.
    - `w90_wan_ham`: For routines like `wham_get_eig_deleig` to get k-derivatives of eigenvalues and matrix elements in the Hamiltonian eigenbasis.

- **External Libraries:**
    - **LAPACK/BLAS:** Used extensively within the utility and Hamiltonian manipulation routines for matrix diagonalizations, multiplications, etc.

- **Interactions:**
    - `berry_main` is the entry point called by `postw90.x`.
    - Requires prior computation of MLWFs and their real-space Hamiltonian and position matrix elements by `wannier90.x` (stored in `seedname.chk`, `seedname_hr.dat`, `seedname_r.dat`). The `w90_get_oper` module reads/constructs these.
    - Outputs various data files (e.g., `seedname-ahc.dat`, `seedname-kubo_S_xx.dat`) containing the calculated physical properties.
    - The k-mesh for interpolation is independent of the k-mesh used in the initial `wannier90.x` run and is specified by `berry_kmesh` or related parameters.
```
