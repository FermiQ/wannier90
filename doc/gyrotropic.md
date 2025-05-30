## Overview
The `w90_gyrotropic` module, part of `postw90.x`, is dedicated to calculating various gyrotropic effects in materials. These effects describe the response of a material to electromagnetic fields that involves a change in the polarization state of light or other phenomena related to spatial dispersion and time-reversal symmetry breaking. The calculations are based on the theoretical framework and Wannier function interpolation methods described in S.S. Tsirkin, P. Aguado Puente, I. Souza, Phys. Rev. B 97, 035158 (2018) (TAS17). The module computes tensors like the D-tensor, K-tensor, C-tensor, and quantities related to natural optical activity (NOA).

## Key Components

### Module `w90_gyrotropic`
- **Description:** The main module for calculating gyrotropic effects.

### Subroutine `gyrotropic_main`
- **Description:** This is the primary driver routine for all calculations within the `w90_gyrotropic` module. It determines which specific gyrotropic tensors/effects to calculate based on the `pw90_gyrotropic%task` input string. It calls routines from `w90_get_oper` to obtain necessary real-space matrix elements (Hamiltonian $H(\mathbf{R})$, position $\mathbf{r}(\mathbf{R})$, spin $\mathbf{s}(\mathbf{R})$) and then calls `gyrotropic_get_k_list` to compute the k-space integrands. Finally, it sums/integrates these contributions and writes the results to formatted output files.
- **Parameters:**
    - `pw90_gyrotropic` (type `pw90_gyrotropic_type`): Contains all gyrotropic-specific input parameters (k-mesh, energy smearing, frequency range, task selection, etc.).
    - Other parameters are similar to those in `w90_berry_main`, providing system information, Wannier function data, matrix elements (e.g., `HH_R`, `AA_R`, `SS_R`), `eigval`, `real_lattice`, etc.
- **Returns:** Generates various output files (e.g., `seedname-gyrotropic-K_orb.dat`, `seedname-gyrotropic-D.dat`) depending on the tasks selected.

### Subroutine `gyrotropic_get_k_list`
- **Description:** Calculates the k-point resolved contributions to the various gyrotropic tensors. This involves computing or using pre-computed k-space matrix elements of position, Hamiltonian, and their derivatives, as well as spin operators if `eval_spn` is true. The specific integrands depend on the tensor being calculated (K, C, D, $D(\omega)$, NOA).
- **Parameters:** A large list including `pw90_gyrotropic`, system parameters, real-space matrix elements (`HH_R`, `AA_R`, `BB_R`, `CC_R`, `SS_R`), current `kpt`, `kweight`, and output arrays for the k-resolved contributions to gyrotropic tensors (e.g., `gyro_K_orb`, `gyro_D`).
- **Returns:** Populates the output arrays with k-point contributions to the requested gyrotropic tensors.

### Subroutine `gyrotropic_get_curv_w_k`
- **Description:** Calculates the band-resolved frequency-dependent Berry curvature $\tilde{\Omega}(\omega)$, which is a component for the $D(\omega)$ tensor (Eq. 12 of TAS17).
- **Parameters:** `eig` (eigenvalues at current k), `AA` (Berry connection $\mathbf{A}_{nm}(\mathbf{k})$), `curv_w_k` (output), `pw90_gyrotropic` (for frequency list).
- **Returns:** Populates `curv_w_k(num_wann, n_freq, 3)`.

### Subroutine `gyrotropic_get_NOA_Bnl_orb` and `gyrotropic_get_NOA_Bnl_spin`
- **Description:** These routines calculate intermediate matrix elements $B^{\text{orb}}_{nl,ac}$ and $B^{\text{spin}}_{nl,ac}$ which are needed for computing the natural optical activity tensor $\gamma_{abc}$ (Eq. C12, C14, C15 of TAS17). These involve sums over intermediate states and matrix elements of position, Hamiltonian derivatives, and spin operators.
- **Parameters:** `eig`, `del_eig`, `AA`, `S_h` (spin matrices in H-gauge), occupation lists, `Bnl` (output).

### Subroutine `gyrotropic_outprint_tensor` and `gyrotropic_outprint_tensor_w`
- **Description:** Helper routines for writing the calculated gyrotropic tensors to output files in a formatted way. `gyrotropic_outprint_tensor_w` handles the actual writing for a given frequency `omega` (or $\omega=0$ for static tensors). It can output symmetric and antisymmetric parts of $3 \times 3$ tensors or 1D array data.
- **Parameters:** `stdout`, `seedname`, `pw90_gyrotropic`, `fermi_energy_list`, `f_out_name` (base for output filename), `arrEf` (static 3x3 tensor vs $E_F$), `arrEf1D` (static 1D array vs $E_F$), `arrEfW` (frequency-dependent 3x3 tensor vs $E_F$), `units`, `comment`, `symmetrize`.

## Important Variables/Constants

- **`pw90_gyrotropic_type` (from `postw90_types.F90`):**
    - `task` (character): String specifying tasks (e.g., '-k', '-c', '-d0', '-dw', '-spin', '-noa', 'all').
    - `kmesh%mesh(3)` (integer): Dimensions of the interpolation k-mesh.
    - `smearing%type_index`, `smearing%fixed_width`: Smearing for energy delta functions.
    - `freq_list(:)` (complex(dp)): List of frequencies $\hbar\omega + i\eta_{smr}$ for frequency-dependent properties.
    - `nfreq` (integer): Number of frequency points.
    - `eigval_max` (real(dp)): Energy cutoff for including bands in sums.
    - `degen_thresh` (real(dp)): Threshold to consider bands degenerate.
    - `band_list(:)`: List of bands to include in calculations.
    - `box_corner(3)`, `box_b1(3)`, `box_b2(3)`, `box_b3(3)`: Define a specific region in k-space for integration (e.g., around a Weyl point).
- **Output arrays:** `gyro_K_orb`, `gyro_K_spn`, `gyro_D`, `gyro_Dw`, `gyro_C`, `gyro_NOA_orb`, `gyro_NOA_spn`, `gyro_DOS`.
- **`alpha_A`, `beta_A` (integer parameters):** Indices for constructing pseudovectors from antisymmetric tensors.

## Usage Examples
This module is invoked by `postw90.x` when `gyrotropic = .true.` is set in the `seedname.win` input file. Specific tasks and parameters are also set in the input file.

**Example `seedname.win` section:**
```
gyrotropic = .true.
gyrotropic_task = '-k -d0 -dw -noa'  ! Calculate K, D(0), D(w), and NOA
gyrotropic_kmesh = 20 20 20
fermi_energy = 0.0
gyrotropic_freq_min = 0.0
gyrotropic_freq_max = 5.0
gyrotropic_freq_step = 0.05
gyrotropic_smr_fixed_en_width = 0.1 ! Smearing for delta functions in eV
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_i`, `twopi`, `pw90_physical_constants_type`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication.
    - `w90_io`: For file I/O and timing.
    - `w90_types`: For shared data structures.
    - `w90_postw90_types`: For `postw90`-specific types.
    - `w90_utility`: For matrix operations, smearing functions.
    - `w90_berry`: Uses `berry_get_imf_klist` and `berry_get_imfgh_klist` as components for some gyrotropic tensor calculations.
    - `w90_get_oper`: To obtain real-space matrix elements (`HH_R`, `AA_R`, `BB_R`, `CC_R`, `SS_R`).
    - `w90_postw90_common`: For Fourier transforms and k-mesh spacing.
    - `w90_wan_ham`: For band energies and k-derivatives (`wham_get_eig_deleig`, `wham_get_D_h`).
    - `w90_spin`: For spin matrix elements (`spin_get_S`).

- **External Libraries:**
    - **LAPACK/BLAS:** Used extensively for matrix operations within utility and Hamiltonian modules.

- **Interactions:**
    - `gyrotropic_main` is the entry point called by `postw90.x`.
    - Requires MLWFs and various real-space matrix elements from a `wannier90.x` run.
    - Outputs data files (e.g., `seedname-gyrotropic-K_orb.dat`) containing the calculated gyrotropic tensors as a function of Fermi energy and/or frequency.
```
