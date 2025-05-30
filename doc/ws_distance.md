## Overview
The `w90_ws_distance` module is designed to compute the optimal Wigner-Seitz (WS) supercell vector that minimizes the distance between a Wannier function (WF) $j$ located in a cell $\mathbf{R}$ (denoted $W_j(\mathbf{R})$) and a WF $i$ located in the home cell ($W_i(\mathbf{0})$). Essentially, for each pair $(W_i(\mathbf{0}), W_j(\mathbf{R}))$, it finds the supercell lattice vector $\mathbf{S}$ (a multiple of `mp_grid` times primitive lattice vectors) such that $W_j(\mathbf{R}+\mathbf{S})$ is in the Wigner-Seitz cell of $W_i(\mathbf{0})$. This is particularly important for correctly interpolating quantities like the Hamiltonian in real space, ensuring that the shortest possible equivalent vector is always used. The module also handles cases where multiple such $\mathbf{S}$ vectors yield the same minimal distance (degeneracy).

## Key Components

### Module `w90_ws_distance`
- **Description:** Contains routines for Wigner-Seitz cell distance calculations.

### Subroutine `ws_translate_dist`
- **Description:** This is the main routine that calculates the optimal supercell translations. For every pair of Wannier functions $(i, j)$ and every real-space lattice vector $\mathbf{R}$ (from `irvec`), it determines the integer shifts (multiples of `mp_grid` dotted with lattice vectors) required to bring $W_j(\mathbf{R})$ into the Wigner-Seitz cell of $W_i(\mathbf{0})$. It stores these shifts and their degeneracies in the `ws_distance` derived type.
- **Parameters:**
    - `ws_distance` (type(ws_distance_type), intent(inout)): Output structure to store results.
    - `ws_region` (type(ws_region_type), intent(in)): Contains parameters like tolerance (`ws_distance_tol`) and search size (`ws_search_size`).
    - `num_wann` (integer, intent(in)): Number of Wannier functions.
    - `wannier_centres` (real(kind=dp), intent(in)): Cartesian coordinates of WF centers.
    - `real_lattice` (real(kind=dp), intent(in)): Real-space lattice vectors.
    - `mp_grid(3)` (integer, intent(in)): Monkhorst-Pack grid dimensions defining the supercell.
    - `nrpts` (integer, intent(in)): Number of $\mathbf{R}$ vectors.
    - `irvec(3, nrpts)` (integer, intent(in)): The $\mathbf{R}$ vectors.
    - `error` (type(w90_error_type), allocatable, intent(out)): Error object.
    - `comm` (type(w90comm_type), intent(in)): MPI communicator.
    - `force_recompute` (logical, optional, intent(in)): If true, forces recalculation even if `ws_distance%done` is true.
- **Returns:** Populates `ws_distance%irdist`, `ws_distance%crdist`, and `ws_distance%ndeg`. Sets `ws_distance%done` to true.

### Subroutine `R_wz_sc`
- **Description:** An internal helper routine. Given an input vector `R_in` and a reference center `R0`, it finds the equivalent vector `R_bz = R_in + S` (where `S` is a superlattice vector defined by `mp_grid`) such that `R_bz` is closest to `R0` (i.e., in the Wigner-Seitz cell of `R0` defined by the superlattice). It also finds all degenerate `S` vectors that give the same minimal distance.
- **Parameters:**
    - `R_in(3)` (real(DP), intent(in)): The initial vector.
    - `R0(3)` (real(DP), intent(in)): The center of the Wigner-Seitz cell.
    - `ndeg` (integer, intent(out)): Degeneracy (number of equivalent shortest vectors).
    - `R_out(3, ndegenx)` (real(DP), intent(out)): The `ndeg` equivalent shortest vectors.
    - `shifts(3, ndegenx)` (integer, intent(out)): The superlattice shifts (in fractional coordinates of the primitive cell, scaled by `mp_grid`) corresponding to each `R_out`.
    - `mp_grid`, `real_lattice`, `inv_lattice`, `ws_search_size`, `ws_distance_tol`, `error`, `comm`.
- **Returns:** Populates `ndeg`, `R_out`, and `shifts`.

### Subroutine `ws_write_vec`
- **Description:** Writes the calculated Wigner-Seitz shift vectors to a formatted text file named `seedname_wsvec.dat`. For each original $(\mathbf{R}, i, j)$ triplet, it writes the number of degenerate shortest shifts and the shifts themselves (as difference vectors relative to the original $\mathbf{R}$). If `use_ws_distance` is false, it writes zero shifts with degeneracy 1.
- **Parameters:** `ws_distance`, `nrpts`, `irvec`, `num_wann`, `use_ws_distance` (logical flag from input), `seedname`, `error`, `comm`.
- **Returns:** Creates the `seedname_wsvec.dat` file.

### Subroutine `clean_ws_translate`
- **Description:** Deallocates arrays within the `ws_distance` type and resets its `done` flag to false. This is used to force recalculation if `ws_translate_dist` is called again with `force_recompute = .true.`.
- **Parameters:** `ws_distance` (type(ws_distance_type), intent(inout)).

## Important Variables/Constants

- **`ndegenx` (integer, parameter, value=8):** Maximum expected degeneracy for a point in a 3D Wigner-Seitz cell (e.g., a vertex of a cube touching 8 cells).
- **`ws_distance_type` (Derived Type from `w90_types`):**
    - **`irdist(3, ndegenx, num_wann, num_wann, nrpts)` (integer, allocatable):** Stores the components of the *total* integer lattice vector (primitive cell units) that connects $W_i(\mathbf{0})$ to the Wigner-Seitz cell equivalent of $W_j(\mathbf{R})$. This is $\mathbf{R} + \mathbf{S}$.
    - **`crdist(3, ndegenx, num_wann, num_wann, nrpts)` (real(DP), allocatable):** Cartesian version of `irdist`.
    - **`ndeg(num_wann, num_wann, nrpts)` (integer, allocatable):** Stores the degeneracy (number of equivalent shortest vectors) for each $(i, j, \mathbf{R})$ pair.
    - **`done` (logical):** Flag to indicate if the calculation has been performed.
- **`ws_region_type` (Derived Type from `w90_types`):**
    - **`use_ws_distance` (logical):** Master switch read from input to enable/disable this functionality.
    - **`ws_distance_tol` (real(kind=dp)):** Tolerance for comparing distances.
    - **`ws_search_size(3)` (integer):** Defines the search range for superlattice vectors $\mathbf{S}$.

## Usage Examples
This module is typically used when interpolating quantities like the Hamiltonian or Berry curvature elements, where it's important to use matrix elements $H_{ij}(\mathbf{R}_{eff})$ where $\mathbf{R}_{eff}$ is the shortest vector connecting $W_i$ to an image of $W_j$.

1.  **Initialization (called once):**
    ```fortran
    ! In plot_main or similar, before interpolation loops
    if (ws_region%use_ws_distance) then
        call ws_translate_dist(ws_distance, ws_region, num_wann, &
                               wannier_data%centres, real_lattice, mp_grid, nrpts, &
                               irvec, error, comm)
        ! ... error check ...
    end if
    ```

2.  **Writing the shift vectors (optional output):**
    ```fortran
    ! If output_file%write_hr or write_tb etc. is true
    if (ws_region%use_ws_distance .and. .not. ws_distance%done) then ! Ensure it's computed
        call ws_translate_dist(...)
    end if
    call ws_write_vec(ws_distance, nrpts, irvec, num_wann, ws_region%use_ws_distance, &
                      seedname, error, comm)
    ```

3.  **Using the results in an interpolation loop (conceptual):**
    ```fortran
    ! When calculating H(k) = sum_R exp(ikR) H(0,R)
    ! ham_kprm(i,j) = sum_{irpt=1,nrpts} sum_{ideg=1,ws_distance%ndeg(i,j,irpt)} &
    !    exp(i * k_vec .dot. ws_distance%crdist(:,ideg,i,j,irpt)) * &
    !    ham_r(i,j,irpt) / real(ws_distance%ndeg(i,j,irpt), dp)
    ```
    The `seedname_wsvec.dat` file provides the necessary $\mathbf{S}$ vectors (as `irdist - irvec`) if one wants to apply these shifts externally or for debugging.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`.
    - `w90_error`: For error handling (`w90_error_type`, `set_error_alloc`, `set_error_fatal`, `set_error_file`).
    - `w90_utility`: For coordinate transformations (`utility_cart_to_frac`, `utility_frac_to_cart`) and matrix inversion (`utility_inverse_mat`).
    - `w90_types`: Defines `ws_region_type` and `ws_distance_type`.
    - `w90_io`: For `io_file_unit`, `io_date` in `ws_write_vec`.

- **External Libraries:** None.

- **Interactions:**
    - This module is primarily called by `w90_plot` (specifically `plot_interpolate_bands`) and potentially other routines that need to perform Fourier transforms from real space to k-space using the $H(\mathbf{R})$ matrix elements.
    - It uses Wannier function centers (`wannier_data%centres`) and the real-space lattice vectors (`real_lattice`, `mp_grid`) to determine the Wigner-Seitz cell geometry.
    - The output `ws_distance` structure is then used by the calling routines to select the correct (shortest) $H(\mathbf{R}_{eff})$ for the Fourier sum.
    - The parameter `ws_region%use_ws_distance` (read from `seedname.win`) controls whether this machinery is active.
```
