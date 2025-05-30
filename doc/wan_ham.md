## Overview
The `w90_wan_ham` module, part of `postw90.x`, provides routines to work with the Hamiltonian in the Wannier basis. Its primary functions are to compute quantities derived from the k-space Wannier Hamiltonian $H_W(\mathbf{k})$, which is obtained by Fourier transforming the real-space Hamiltonian $H(\mathbf{R})$ (itself constructed by `w90_get_oper` or read directly if `effective_model = .true.`). Key calculations include obtaining eigenvalues $E_{n\mathbf{k}}$ and their k-derivatives (band velocities $\mathbf{v}_{n\mathbf{k}} = \frac{1}{\hbar} \nabla_{\mathbf{k}} E_{n\mathbf{k}}$), as well as matrix elements of k-derivatives of $H_W(\mathbf{k})$ which are essential for Berry phase and other transport-related property calculations.

## Key Components

### Module `w90_wan_ham`
- **Description:** Contains subroutines for operations on the Wannier Hamiltonian, primarily for interpolation and derivative calculations.

### Subroutine `wham_get_eig_deleig`
- **Description:** A core routine that, for a given k-point, calculates the eigenvalues $E_{n\mathbf{k}}$ of the interpolated Wannier Hamiltonian $H_W(\mathbf{k})$, the matrix of eigenvectors $U_\mathbf{k}$ that diagonalizes $H_W(\mathbf{k})$ (i.e., $U_\mathbf{k}^\dagger H_W(\mathbf{k}) U_\mathbf{k} = \text{diag}(E_{n\mathbf{k}})$), and the k-derivatives of the eigenvalues $\nabla_{\mathbf{k}} E_{n\mathbf{k}}$. The derivatives are computed using the Hellmann-Feynman theorem or degenerate perturbation theory if `pw90_band_deriv_degen%use_degen_pert` is true.
- **Parameters:**
    - `eig(num_wann)` (out): Interpolated eigenvalues.
    - `del_eig(num_wann, 3)` (out): $\nabla_{\mathbf{k}} E_{n\mathbf{k}}$.
    - `HH(:, :)` (out): $H_W(\mathbf{k})$.
    - `delHH(:, :, :)` (out): $\nabla_{\mathbf{k}} H_W(\mathbf{k})$.
    - `UU(:, :)` (out): Eigenvectors $U_\mathbf{k}$.
    - `HH_R`: Real-space Hamiltonian (input for Fourier transform).
    - `kpt`: The k-point for calculation.
    - Other system and control parameters.
- **Returns:** Populates `eig`, `del_eig`, `HH`, `delHH`, and `UU`.

### Subroutine `wham_get_eig_deleig_TB_conv`
- **Description:** A version of `wham_get_eig_deleig` that uses the "tight-binding" phase convention for Fourier transforms (includes Wannier center shifts in the phase factor). It takes pre-computed `delHH` and `UU` as input rather than recomputing them from `HH_R`.
- **Parameters:** `delHH` (in), `UU` (in), `eig` (in), `del_eig` (out).

### Subroutine `wham_get_eig_UU_HH_AA_sc`
- **Description:** A wrapper routine that first calls `get_HH_R` and `get_AA_R` (from `w90_get_oper`) to ensure $H(\mathbf{R})$ and $\mathbf{A}(\mathbf{R})$ (position matrix elements) are available. Then, it Fourier transforms them to get $H_W(\mathbf{k})$ (`HH`), $\nabla_{\mathbf{k}} H_W(\mathbf{k})$ (`HH_da`), and $\nabla_{\mathbf{k}} \nabla_{\mathbf{k}} H_W(\mathbf{k})$ (`HH_dadb`). Finally, it diagonalizes $H_W(\mathbf{k})$ to get `eig` and `UU`. This is used in shift current calculations.
- **Parameters:** Similar to `wham_get_eig_deleig`, plus `AA_R` and outputs `HH_da`, `HH_dadb`.

### Subroutine `wham_get_eig_UU_HH_AA_sc_TB_conv`
- **Description:** Similar to `wham_get_eig_UU_HH_AA_sc` but uses the "tight-binding" phase convention for Fourier transforms.

### Subroutine `wham_get_D_h_a` (older version, likely for specific components)
- **Description:** Computes $D^H_\alpha = U^\dagger (\nabla_{k_\alpha} U)$, where $U$ are eigenvectors of $H_W(\mathbf{k})$. Uses Eq. (24) of WYSV06 (PRB 74, 195118 (2006)). This version seems to be for a single Cartesian component $\alpha$.
- **Parameters:** `delHH_a` ($\nabla_{k_\alpha} H_W(\mathbf{k})$), `UU`, `eig`, `ef` (Fermi energy for occupation), `D_h_a` (output).

### Subroutine `wham_get_D_h`
- **Description:** Computes the matrix $D^H_{\alpha,nm} = \frac{\langle u_{n\mathbf{k}}^{(H)} | \nabla_{k_\alpha} H_W(\mathbf{k}) | u_{m\mathbf{k}}^{(H)} \rangle}{E_{m\mathbf{k}} - E_{n\mathbf{k}}}$ for $n \neq m$, and $0$ for $n=m$. $u^{(H)}$ are eigenstates of $H_W(\mathbf{k})$.
- **Parameters:** `delHH` ($\nabla_{\mathbf{k}} H_W(\mathbf{k})$), `D_h` (output $D^H_\alpha$), `UU`, `eig`.

### Subroutine `wham_get_D_h_P_value`
- **Description:** Similar to `wham_get_D_h`, but uses a different prescription for the energy denominator: $(E_m - E_n) / ((E_m - E_n)^2 + \eta^2)$, where $\eta$ is `pw90_berry%sc_eta`. This is from Blount's work, often used in shift current.
- **Parameters:** `pw90_berry`, `delHH`, `D_h`, `UU`, `eig`.

### Subroutine `wham_get_JJp_JJm_list`
- **Description:** Computes matrices $J^{+}_{a,nm} = i \frac{\langle u_{n\mathbf{k}}^{(H)} | \nabla_{k_a} H_W(\mathbf{k}) | u_{m\mathbf{k}}^{(H)} \rangle}{E_{m\mathbf{k}} - E_{n\mathbf{k}}}$ and $J^{-}_{a,nm} = i \frac{\langle u_{n\mathbf{k}}^{(H)} | \nabla_{k_a} H_W(\mathbf{k}) | u_{m\mathbf{k}}^{(H)} \rangle}{E_{n\mathbf{k}} - E_{m\mathbf{k}}}$ for occupied $m$ and unoccupied $n$. These are used in orbital magnetization calculations.
- **Parameters:** `delHH` (for a specific Cartesian direction $a$), `UU`, `eig`, `JJp_list` (output), `JJm_list` (output), `fermi_energy_list`, optional `occ` (occupations).

### Subroutine `wham_get_occ_mat_list`
- **Description:** Computes the occupation matrix $f_{nm}(\mathbf{k}) = \sum_l U_{nl} f(E_{l\mathbf{k}}) U^*_{ml}$ and $g_{nm}(\mathbf{k}) = \delta_{nm} - f_{nm}(\mathbf{k})$ for a list of Fermi energies. $f(E_{l\mathbf{k}})$ is the Fermi-Dirac occupation of eigenstate $l$.
- **Parameters:** `fermi_energy_list`, `f_list` (output), `g_list` (output), `UU`, `eig` (optional, if `occ` not given), `occ` (optional, if `eig` not given).

## Important Variables/Constants
This module primarily provides computational routines. Key inputs are the real-space Hamiltonian `HH_R` and the k-point `kpt`. Outputs are k-space quantities like `eig` (eigenvalues), `del_eig` (velocities), `HH` ($H_W(\mathbf{k})$), `delHH` ($\nabla H_W(\mathbf{k})$), `UU` (eigenvectors of $H_W(\mathbf{k})$), and derived matrices like `D_h`, `JJp_list`, `f_list`.

## Usage Examples
This module is used by other `postw90.x` modules that require interpolated band structures or their derivatives.

**Example: Getting energies and velocities at a k-point (conceptual, from `w90_dos`)**
```fortran
! In a routine within w90_dos or w90_berry
complex(kind=dp), allocatable :: HH(:,:), delHH(:,:,:), UU(:,:)
real(kind=dp), allocatable :: eig_k(:), deleig_k(:,:)
! ... allocate arrays ...
! ... ensure HH_R is available via get_HH_R ...

call wham_get_eig_deleig(dis_manifold, kpt_latt, pw90_band_deriv_degen, ws_region, &
                         print_output, wannier_data, ws_distance, wigner_seitz, delHH, HH, &
                         HH_R, u_matrix, UU, v_matrix, deleig_k, eig_k, eigval, kpt_current, &
                         real_lattice, scissors_shift, mp_grid, num_bands, num_kpts, &
                         num_wann, num_valence_bands, effective_model, &
                         have_disentangled, seedname, stdout, timer, error, comm)
! Now eig_k contains eigenvalues and deleig_k contains dE/dk at kpt_current
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_i`.
    - `w90_error`: For error handling.
    - `w90_comms`: For `w90comm_type`.
    - `w90_types`: For shared data structures.
    - `w90_postw90_types`: For `postw90`-specific types.
    - `w90_utility`: For `utility_diagonalize`, `utility_rotate`, `utility_rotate_diag`, `utility_zgemm_new`.
    - `w90_get_oper`: Routines like `wham_get_eig_UU_HH_AA_sc` call `get_HH_R` and `get_AA_R`.
    - `w90_postw90_common`: For Fourier transform routines (`pw90common_fourier_R_to_k_new`, etc.) and `pw90common_get_occ`.

- **External Libraries:**
    - **LAPACK/BLAS:** Used for matrix diagonalizations (e.g., `ZHPEVX` via `utility_diagonalize`) and matrix multiplications.

- **Interactions:**
    - This module is a workhorse for many `postw90.x` calculations.
    - It takes the real-space Hamiltonian `HH_R` (from `w90_get_oper`) and a k-point as input.
    - It provides k-space quantities ($E_{n\mathbf{k}}$, $\nabla_{\mathbf{k}} E_{n\mathbf{k}}$, $H_W(\mathbf{k})$, $\nabla_{\mathbf{k}} H_W(\mathbf{k})$, $U_\mathbf{k}$, etc.) to modules like `w90_berry`, `w90_dos`, `w90_boltzwann`, `w90_kpath`, `w90_kslice`, and `w90_gyrotropic`.
```
