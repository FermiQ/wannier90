## Overview
The `w90_utility` module provides a collection of general-purpose helper routines used throughout the Wannier90 code. These utilities include linear algebra operations (often wrappers for BLAS/LAPACK calls), coordinate transformations, matrix manipulations, string processing, and mathematical functions like error functions and Gaussian smearing functions. This module helps to centralize common operations, promoting code reuse and maintainability.

## Key Components

### Module `w90_utility`
- **Description:** The main module containing various utility subroutines and functions.

### Linear Algebra and Matrix Operations
- **`utility_zgemm(c, a, transa, b, transb, n)`:** Wrapper for `zgemm` to compute $C = op(A) op(B)$ for square complex matrices of size $n \times n$.
- **`utility_zgemm_new(a, b, c, transa_opt, transb_opt)`:** A more general wrapper for `zgemm` using assumed-shape arrays, allowing for non-square matrices.
- **`utility_zgemmm(a, transa, b, transb, c, transc, prod1, eigval, prod2)`:** Computes triple matrix product $op(A)op(B)op(C)$ and optionally $op(A)diag(eigval)op(B)op(C)$.
- **`utility_zdotu(a, b)`:** Computes dot product of two complex vectors `a` and `b`.
- **`utility_det3(A)`:** Returns the determinant of a $3 \times 3$ real matrix.
- **`utility_inv3(a, b, det)`:** Computes the adjoint (`b`) and determinant (`det`) of a $3 \times 3$ real matrix `a`.
- **`utility_inv2(a, b, det)`:** Computes the adjoint (`b`) and determinant (`det`) of a $2 \times 2$ real matrix `a`.
- **`utility_inverse_mat(a, b)`:** Computes the inverse (`b`) of a $3 \times 3$ real matrix `a`.
- **`utility_metric(lattice, metric)`:** Calculates the metric tensor for a given lattice.
- **`utility_diagonalize(mat, dim, eig, rot, error, comm)`:** Diagonalizes a `dim` x `dim` Hermitian matrix `mat`, returning eigenvalues `eig` and eigenvectors `rot`. Wrapper for `ZHPEVX`.
- **`utility_rotate(mat, rot, dim)`:** Computes $(rot)^\dagger \cdot mat \cdot rot$.
- **`utility_rotate_new(mat, rot, N, reverse)`:** Computes $(rot)^\dagger \cdot mat \cdot rot$ or $rot \cdot mat \cdot (rot)^\dagger$. Overwrites `mat`.
- **`utility_matmul_diag(mat1, mat2, dim)`:** Computes diagonal elements of $mat1 \cdot mat2$.
- **`utility_rotate_diag(mat, rot, dim)`:** Computes diagonal elements of $(rot)^\dagger \cdot mat \cdot rot$.
- **`utility_commutator_diag(mat1, mat2, dim)`:** Computes diagonal elements of $[mat1, mat2]$.
- **`utility_re_tr_prod(a, b)` / `utility_im_tr_prod(a, b)`:** Computes $Re(Tr(A \cdot B))$ / $Im(Tr(A \cdot B))$.
- **`utility_re_tr(mat)` / `utility_im_tr(mat)`:** Computes $Re(Tr(mat))$ / $Im(Tr(mat))$.

### Coordinate Transformations
- **`utility_recip_lattice_base(real_lat, recip_lat, volume)`:** Calculates reciprocal lattice vectors and cell volume from real-space lattice vectors.
- **`utility_recip_lattice(real_lat, recip_lat, volume, error, comm)`:** Same as above but includes a check for near-zero volume.
- **`utility_frac_to_cart(frac, cart, real_lat)`:** Converts fractional coordinates to Cartesian coordinates.
- **`utility_cart_to_frac(cart, frac, inv_lat)`:** Converts Cartesian coordinates to fractional coordinates using the inverse real lattice.
- **`utility_translate_home(vec, real_lat)`:** Translates a Cartesian vector `vec` into the home unit cell.

### String Manipulation
- **`utility_strip(string)`:** Removes all blank spaces from a string.
- **`utility_lowercase(string)`:** Converts a string to lowercase.
- **`utility_string_to_coord(string_tmp, outvec, error, comm)`:** Parses a string like "0.0,1.0,0.5" into a 3-element real vector `outvec`.

### Mathematical Functions (Smearing, Error Functions)
- **`utility_wgauss(x, n)`:** Computes approximate theta functions for smearing:
    - `n >= 0`: Methfessel-Paxton scheme of order `n`.
    - `n = -1`: Marzari-Vanderbilt cold smearing.
    - `n = -99`: Fermi-Dirac function.
- **`utility_w0gauss(x, n, error, comm)`:** Computes the derivative of `utility_wgauss` (approximations to the delta function).
- **`utility_w0gauss_vec(x, n, error, comm)`:** Vectorized version of `utility_w0gauss` (currently only implemented for some `n`).
- **`qe_erf(x)`:** Error function $erf(x)$, using Cody's rational approximations.
- **`qe_erfc(x)`:** Complementary error function $erfc(x) = 1 - erf(x)$.
- **`gauss_freq(x)`:** Computes $(1+erf(x/\sqrt{2}))/2$.

### Other Utilities
- **`utility_compar(a, b, ifpos, ifneg)`:** Compares two 3-element real vectors `a` and `b`. Sets `ifpos=1` if $a \approx b$, and `ifneg=1` if $a \approx -b$.

## Important Variables/Constants
This module primarily provides functions and subroutines. It uses constants defined in `w90_constants` (e.g., `dp`, `cmplx_0`, `cmplx_1`, `twopi`, `eps5`, `eps8`).

## Usage Examples

**Matrix multiplication:**
```fortran
use w90_utility, only: utility_zgemm_new
complex(kind=dp) :: A(2,3), B(3,4), C(2,4)
! ... initialize A and B ...
call utility_zgemm_new(A, B, C) ! C = A * B
```

**Coordinate conversion:**
```fortran
use w90_utility, only: utility_frac_to_cart, utility_recip_lattice_base
real(kind=dp) :: real_lattice(3,3), frac_coords(3), cart_coords(3)
! ... initialize real_lattice and frac_coords ...
call utility_frac_to_cart(frac_coords, cart_coords, real_lattice)
```

**Determinant of a 3x3 matrix:**
```fortran
use w90_utility, only: utility_det3
real(kind=dp) :: matrix_A(3,3), determinant_A
! ... initialize matrix_A ...
determinant_A = utility_det3(matrix_A)
```

**Gaussian smearing for Density of States:**
```fortran
use w90_utility, only: utility_w0gauss
real(kind=dp) :: energy_diff, smearing_width, dos_contrib
integer :: smearing_order = 0 ! Gaussian
type(w90_error_type), allocatable :: error
type(w90comm_type) :: comm ! dummy here
! ...
dos_contrib = utility_w0gauss((eigenvalue - energy_point)/smearing_width, smearing_order, error, comm) / smearing_width
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp` (double precision kind) and mathematical/physical constants like `cmplx_0`, `cmplx_1`, `twopi`, `eps*`.
    - `w90_comms`: For `w90comm_type` (though direct MPI calls are not made in most utility routines, it's passed for error handling consistency).
    - `w90_error`: For error reporting (`w90_error_type`, `set_error_fatal`, `set_error_input`).

- **External Libraries:**
    - **BLAS/LAPACK:** Several routines are wrappers or direct calls to BLAS/LAPACK:
        - `zgemm`: Complex matrix-matrix multiplication.
        - `ZHPEVX`: Solves Hermitian packed eigenvalue problems (used in `utility_diagonalize`).
        - `ZGESV`: Solves general complex linear systems (used in `utility_transfer` within `w90_transport`, which is not part of this module but shows typical usage patterns that might inspire utilities).
        - `DSPEV`: Solves real symmetric packed eigenvalue problems. (Example of LAPACK usage, though not directly in a public utility here).
        - `ZGESVD`: Complex singular value decomposition.

- **Interactions:**
    - This module is widely used by most other modules in Wannier90 whenever general mathematical, matrix, string, or coordinate operations are needed.
    - It provides a common, tested set of low-level functions, reducing code duplication and potential for errors in more specialized modules.
    - The smearing functions are critical for post-processing tasks like Density of States (DOS) and other spectral quantity calculations.
```
