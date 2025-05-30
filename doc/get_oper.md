## Overview
The `w90_get_oper` module, part of `postw90.x`, is responsible for calculating the real-space matrix elements of various operators in the Wannier function basis. These operators include the Hamiltonian ($H$), position ($\mathbf{r}$), and spin ($\sigma_i$). The module reads k-space matrix elements (e.g., overlaps $M_{mn}^{(\mathbf{k,b})}$, Bloch Hamiltonian eigenvalues $E_{n\mathbf{k}}$, spin matrices $\langle u_{n\mathbf{k}} | \sigma_i | u_{m\mathbf{k}} \rangle$) provided by an ab initio code (via interface files like `seedname.mmn`, `seedname.eig`, `seedname.spn`) and transforms them into the Wannier basis in real space. These real-space matrix elements are then used by other `postw90.x` modules (like `w90_berry`) to compute various physical properties.

## Key Components

### Module `w90_get_oper`
- **Description:** The main module for obtaining real-space operator matrix elements in the Wannier basis.

### Subroutine `get_HH_R`
- **Description:** Computes the real-space Hamiltonian matrix elements $\langle \mathbf{0}n | H | \mathbf{R}m \rangle$ in eV. If `effective_model` is true, it reads these directly from `seedname_HH_R.dat`. Otherwise, it Fourier transforms the k-space Hamiltonian (constructed from eigenvalues `eigval` and rotation matrices `v_matrix` which diagonalize the Wannier Hamiltonian at each k-point) to real space. It can also apply a scissor operator correction.
- **Parameters:** `dis_manifold`, `kpt_latt`, `print_output`, `wigner_seitz`, `HH_R` (output), `u_matrix`, `v_matrix`, `eigval`, `real_lattice`, `scissors_shift`, dimensions, `effective_model`, `have_disentangled`, `seedname`, `stdout`, `timer`, `error`, `comm`.
- **Returns:** Populates `HH_R`.

### Subroutine `get_AA_R`
- **Description:** Computes the real-space position operator matrix elements $\langle \mathbf{0}n | r_\alpha | \mathbf{R}m \rangle$. If `effective_model` is true, it reads these from `seedname_AA_R.dat`. Otherwise, it calculates them by Fourier transforming the Berry connection matrices $A_{\alpha,nm}^{(\mathbf{k})} = i \langle u_{n\mathbf{k}} | \nabla_{k_\alpha} u_{m\mathbf{k}} \rangle$. The k-space Berry connections are constructed from the overlap matrices $M_{nm}^{(\mathbf{k,b})}$ read from `seedname.mmn`.
- **Parameters:** `pw90_berry`, `dis_manifold`, `kmesh_info`, `kpt_latt`, `print_output`, `AA_R` (output), `HH_R` (used if `effective_model` and `pw90_berry%transl_inv` for some consistency checks), `v_matrix`, `eigval`, `irvec`, `nrpts`, dimensions, `effective_model`, `have_disentangled`, `seedname`, `stdout`, `timer`, `error`, `comm`.
- **Returns:** Populates `AA_R`.

### Subroutine `get_BB_R`
- **Description:** Computes $\langle \mathbf{0}n | H (\mathbf{r}-\mathbf{R}) | \mathbf{R}m \rangle$ (related to orbital magnetization) by Fourier transforming $B_{\alpha,nm}^{(\mathbf{k})} = i \langle u_{n\mathbf{k}} | H | \nabla_{k_\alpha} u_{m\mathbf{k}} \rangle$.
- **Parameters:** Similar to `get_AA_R`, outputs `BB_R`.

### Subroutine `get_CC_R`
- **Description:** Computes $\langle \mathbf{0}n | r_\alpha H (\mathbf{r}-\mathbf{R})_\beta | \mathbf{R}m \rangle$ (related to orbital magnetization) by Fourier transforming $C_{\alpha\beta,nm}^{(\mathbf{k})} = \langle \nabla_{k_\alpha} u_{n\mathbf{k}} | H | \nabla_{k_\beta} u_{m\mathbf{k}} \rangle$. Reads k-space matrix elements from `seedname.uHu`.
- **Parameters:** Similar to `get_AA_R`, plus `pw90_oper_read` (for `uHu_formatted` flag), outputs `CC_R`.

### Subroutine `get_FF_R` (Not directly called by `berry_main` in the provided `berry.F90`)
- **Description:** Computes $\langle \mathbf{0}n | r_\alpha (\mathbf{r}-\mathbf{R})_\beta | \mathbf{R}m \rangle$ by Fourier transforming $F_{\alpha\beta,nm}^{(\mathbf{k})} = \langle \nabla_{k_\alpha} u_{n\mathbf{k}} | \nabla_{k_\beta} u_{m\mathbf{k}} \rangle$. Reads k-space matrix elements from `seedname.uIu`.
- **Parameters:** Similar to `get_AA_R`, outputs `FF_R`.

### Subroutine `get_SS_R`
- **Description:** Computes the real-space spin operator matrix elements $\langle \mathbf{0}n | \sigma_i | \mathbf{R}m \rangle$ (where $\sigma_i$ are Pauli matrices). It reads k-space spin matrix elements $\langle \psi_{n\mathbf{k}} | \sigma_i | \psi_{m\mathbf{k}} \rangle$ from `seedname.spn`, transforms them to the Wannier gauge, and then Fourier transforms to real space.
- **Parameters:** Similar to `get_AA_R`, plus `pw90_oper_read` (for `spn_formatted` flag), outputs `SS_R`.

### Subroutine `get_SHC_R` (Spin Hall Conductivity related)
- **Description:** Computes various real-space matrix elements needed for Spin Hall Conductivity calculations, specifically $\langle \mathbf{0}n | \sigma_i H | \mathbf{R}m \rangle$ (`SH_R`), $\langle \mathbf{0}n | \sigma_i (\mathbf{r}-\mathbf{R})_\alpha | \mathbf{R}m \rangle$ (`SR_R`), and $\langle \mathbf{0}n | \sigma_i H (\mathbf{r}-\mathbf{R})_\alpha | \mathbf{R}m \rangle$ (`SHR_R`). It combines data from `.spn` and `.mmn` files.
- **Parameters:** Similar to `get_AA_R` and `get_SS_R`, plus `pw90_spin_hall`, outputs `SH_R`, `SHR_R`, `SR_R`.

### Subroutines `get_SAA_R` and `get_SBB_R` (Spin Hall Conductivity related for Ryoo's method)
- **Description:** Compute specialized real-space matrix elements for Ryoo's SHC method:
    - `get_SAA_R`: $\langle \mathbf{0}n | \sigma_a (\mathbf{r}-\mathbf{R})_b | \mathbf{R}m \rangle$. Reads from `seedname.sIu`.
    - `get_SBB_R`: $\langle \mathbf{0}n | \sigma_a H (\mathbf{r}-\mathbf{R})_b | \mathbf{R}m \rangle$. Reads from `seedname.sHu`.
- **Parameters:** Similar to `get_SHC_R`, outputs `SAA_R` or `SBB_R`.

### Private Helper Subroutines
- **`fourier_q_to_R(num_kpts, nrpts, irvec, kpt_latt, op_q, op_R)`:** Performs the discrete Fourier transform $\text{op_R}(\mathbf{R}) = \frac{1}{N_k} \sum_{\mathbf{q}} e^{-i\mathbf{q}\cdot\mathbf{R}} \text{op_q}(\mathbf{q})$.
- **`get_win_min(num_bands, dis_manifold, ik, win_min, have_disentangled)`:** Determines the index of the lowest band within the disentanglement window for a given k-point.
- **`get_gauge_overlap_matrix(..., S_o, have_disentangled, S, H)`:** Transforms an operator `S_o` (e.g., overlap, Hamiltonian, spin) from the Bloch basis to the Wannier gauge using the `v_matrix` (eigenvectors of $H_W(\mathbf{k})$). If `H` is present, it computes $V^\dagger S_o V$; if `S` is present, it computes $V^\dagger (\text{eigval} \cdot S_o) V$ (this seems to be specific to Hamiltonian construction).

## Important Variables/Constants
The module primarily deals with populating large allocatable arrays that store the real-space matrix elements. These arrays are passed as arguments to the public subroutines.
- `HH_R(:, :, :)`: $\langle \mathbf{0}n | H | \mathbf{R}m \rangle$
- `AA_R(:, :, :, :)`: $\langle \mathbf{0}n | r_\alpha | \mathbf{R}m \rangle$
- `BB_R(:, :, :, :)`: $\langle \mathbf{0}n | H (\mathbf{r}-\mathbf{R})_\alpha | \mathbf{R}m \rangle$
- `CC_R(:, :, :, :, :)`: $\langle \mathbf{0}n | r_\alpha H (\mathbf{r}-\mathbf{R})_\beta | \mathbf{R}m \rangle$
- `SS_R(:, :, :, :)`: $\langle \mathbf{0}n | \sigma_i | \mathbf{R}m \rangle$
- And similar for `SH_R`, `SHR_R`, `SR_R`, `SAA_R`, `SBB_R`.

## Usage Examples
This module is used internally by `postw90.x`, particularly by the `w90_berry` module. The `get_*_R` routines are called as needed before specific Berry phase property calculations.

```fortran
! In w90_berry.F90 (conceptual)
if (eval_ahc) then ! If AHC calculation is requested
  ! Ensure HH_R is available (for constructing v_matrix if needed, or if effective_model)
  call get_HH_R(dis_manifold, ..., HH_R, u_matrix, v_matrix, ...)
  ! Ensure AA_R is available
  call get_AA_R(pw90_berry, ..., AA_R, HH_R, v_matrix, ...)
  ! ... then proceed with AHC calculation using HH_R and AA_R ...
end if

if (eval_shc .and. pw90_spin_hall%method == 'qiao') then
    call get_SHC_R(..., SH_R, SHR_R, SR_R, ...)
    ! ...
end if
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_comms`: For `w90comm_type`, `comms_bcast`, `mpirank`.
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_i`, `twopi`, `eps6`.
    - `w90_error`: For error handling.
    - `w90_io`: For file I/O (`io_file_unit`) and timing.
    - `w90_types`: For `dis_manifold_type`, `kmesh_info_type`, `print_output_type`, etc.
    - `w90_postw90_types`: For `pw90_berry_mod_type`, `wigner_seitz_type`, etc.
    - `w90_utility`: For `utility_zgemmm`, `utility_recip_lattice_base`.
    - `w90_wan_ham`: (Indirectly, as `v_matrix` comes from there)

- **External Libraries:**
    - **LAPACK/BLAS:** Used for matrix multiplications (`zgemm` via `utility_zgemmm`) and potentially other operations within helper routines.

- **Interactions:**
    - Reads various interface files generated by `wannier90.x -pp` or the ab initio code:
        - `seedname.mmn` (overlaps $\langle u_{n\mathbf{k}} | u_{m\mathbf{k+b}} \rangle$)
        - `seedname.eig` (eigenvalues $E_{n\mathbf{k}}$)
        - `seedname.uHu` (matrix elements $\langle u_{n\mathbf{k+b_1}} | H_{\mathbf{k}} | u_{m\mathbf{k+b_2}} \rangle$) - for `get_CC_R`.
        - `seedname.uIu` (overlaps $\langle u_{n\mathbf{k+b_1}} | u_{m\mathbf{k+b_2}} \rangle$) - for `get_FF_R`.
        - `seedname.spn` (spin matrices $\langle u_{n\mathbf{k}} | \sigma_i | u_{m\mathbf{k}} \rangle$) - for `get_SS_R` and `get_SHC_R`.
        - `seedname.sHu` (spin-Hamiltonian matrices $\langle u_{n\mathbf{k}} | \sigma_i H_{\mathbf{k}} | u_{m\mathbf{k+b_2}} \rangle$) - for `get_SBB_R`.
        - `seedname.sIu` (spin-overlap matrices $\langle u_{n\mathbf{k}} | \sigma_i | u_{m\mathbf{k+b_2}} \rangle$) - for `get_SAA_R`.
    - If `effective_model = .true.`, it can read `seedname_HH_R.dat` and `seedname_AA_R.dat` directly.
    - Provides the computed real-space operator matrix elements (e.g., `AA_R`, `HH_R`) to other `postw90.x` modules, primarily `w90_berry`, which then use these to calculate physical properties.
```
