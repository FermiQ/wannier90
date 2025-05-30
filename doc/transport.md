## Overview
The `w90_transport` module is designed to calculate quantum transport properties, specifically ballistic conductance and density of states (DOS), using the Landauer-Büttiker formalism. It is based on the methodology described by M. Buongiorno Nardelli in Phys. Rev. B 60, 7828 (1999). The module can operate in two modes: 'bulk' (for periodic systems treated as a conductor between two identical semi-infinite leads of the same material) and 'lcr' (for a lead-conductor-lead setup where the conductor region can be different from the leads). It utilizes Green's function techniques and transfer matrices to compute these properties as a function of energy.

## Key Components

### Module `w90_transport`
- **Description:** The main module for all transport calculation functionalities.

### Subroutine `tran_main`
- **Description:** The primary driver routine for transport calculations. It orchestrates the workflow based on the `transport_mode`. If Hamiltonian data (`H(R)`) is not read from pre-existing files (`tran_read_ht = .false.`), it first calls routines to obtain and prepare $H(\mathbf{R})$ (from `w90_hamiltonian`), then reduces it to a 1D effective Hamiltonian if needed, and finally calls either `tran_bulk` or `tran_lcr`.
- **Parameters:** Takes a comprehensive set of arguments similar to `plot_main`, including system parameters, Wannier function data, Hamiltonian matrices, $U$ matrices, eigenvalues, lattice information, and control flags from `transport_type` and `w90_calculation_type`.
- **Returns:** Generates output files `seedname_qc.dat` (quantum conductance) and `seedname_dos.dat` (density of states).

### Subroutine `tran_reduce_hr`
- **Description:** Reduces the 3D real-space Hamiltonian $H(\mathbf{R})$ to an effective 1D Hamiltonian `hr_one_dim` along the specified transport direction (`real_space_ham%one_dim_dir`). This involves summing up Hamiltonian elements for $\mathbf{R}$ vectors that project onto the same point along the transport direction.
- **Parameters:** `real_space_ham`, `ham_r` (input 3D H(R)), `hr_one_dim` (output 1D H(R)), `real_lattice`, `irvec`, `mp_grid`, `nrpts`, `num_wann`, `one_dim_vec` (output: index of lattice vector along transport).
- **Returns:** Populates `hr_one_dim`.

### Subroutine `tran_cut_hr_one_dim`
- **Description:** Applies cutoffs to the 1D Hamiltonian `hr_one_dim` based on distance (`real_space_ham%dist_cutoff`) and magnitude (`real_space_ham%hr_cutoff`). It determines the number of principal layers (`num_pl`) based on these cutoffs.
- **Parameters:** `real_space_ham`, `transport`, `print_output`, `hr_one_dim` (input/output), `real_lattice`, `wannier_centres_translated`, `mp_grid`, `irvec_max`, `num_pl` (output), `num_wann`, `one_dim_vec`.

### Subroutine `tran_get_ht`
- **Description:** Constructs the block Hamiltonian matrices $H_{00}$ (intra-principal-layer) and $H_{01}$ (inter-principal-layer) for the bulk transport calculation from `hr_one_dim`. It also shifts the diagonal of $H_{00}$ by the Fermi energy. If `transport%write_ht` is true, these matrices are written to `seedname_htB.dat`.
- **Parameters:** `fermi_energy_list`, `transport` (input/output: sets `num_bb`), `hB0` (output $H_{00}$), `hB1` (output $H_{01}$), `hr_one_dim`, `irvec_max`, `num_pl`, `num_wann`, `seedname`.

### Subroutine `tran_bulk`
- **Description:** Performs the quantum conductance and DOS calculation for a bulk system. It uses the transfer matrix method to compute surface Green's functions and then the Landauer formula.
- **Parameters:** `transport`, `hB0`, `hB1` (can be read from file if `transport%read_ht` is true), `seedname`.
- **Returns:** Writes `seedname_qc.dat` and `seedname_dos.dat`.

### Subroutine `tran_lcr`
- **Description:** Performs the quantum conductance and DOS calculation for a lead-conductor-lead (LCR) setup. It computes surface Green's functions for the left and right leads, then calculates self-energies for the conductor due to the leads, and finally the Green's function of the conductor region to get the transmission.
- **Parameters:** `transport`, Hamiltonian blocks for conductor (`hC`, `hCR`), left lead (`hL0`, `hL1`, `hLC`), and right lead (`hR0`, `hR1`), `seedname`.
- **Returns:** Writes `seedname_qc.dat` and `seedname_dos.dat`.

### Subroutine `tran_transfer`
- **Description:** Iteratively computes the transfer matrices ($T_0$, $\tilde{T}_0$) for a semi-infinite lead using the Lopez-Sancho et al. method. These are needed for calculating surface Green's functions.
- **Parameters:** `tot` (output $T_0$), `tott` (output $\tilde{T}_0$), `h_00` (intra-layer Hamiltonian), `h_01` (inter-layer Hamiltonian), `e_scan_cmp` (complex energy), `nxx` (dimension of Hamiltonian blocks).

### Subroutine `tran_green`
- **Description:** Constructs surface or bulk Green's functions using the computed transfer matrices.
- **Parameters:** `tot`, `tott`, `h_00`, `h_01`, `e_scan`, `g` (output Green's function), `igreen` (flag: -1 for left surface, 1 for right surface, 0 for bulk), `invert` (flag: 0 for $G^{-1}$, 1 for $G$), `nxx`.

### LCR Specific Helper Routines:
- **`tran_find_integral_signatures`:** Reads `seedname.unkg` (Bloch states in G-vector basis) and calculates "signatures" (integrals of Wannier functions with spherical harmonics-like basis functions) used to distinguish and sort WFs, especially for LCR setups.
- **`tran_lcr_2c2_sort`:** Sorts Wannier functions into principal layers for the LCR geometry, ensuring consistent grouping and ordering across different layers and unit cells. Uses Wannier centers and signatures.
- **`master_sort_and_group`:** A general helper for `tran_lcr_2c2_sort` to group and sort WFs based on their coordinates in specified directions.
- **`sort`:** Sorts WFs based on a single coordinate.
- **`group`:** Groups WFs based on proximity in one coordinate.
- **`check_and_sort_similar_centres`:** Uses signatures to sort WFs that have very similar centers.
- **`tran_parity_enforce`:** Adjusts the sign of Hamiltonians based on WF signatures to ensure consistent parities across unit cells.
- **`tran_lcr_2c2_build_ham`:** Constructs the various Hamiltonian blocks (`hC`, `hL0`, `hL1`, `hLC`, etc.) required for the LCR calculation from the sorted WFs and 1D Hamiltonian.

### File I/O Helpers for Hamiltonian Blocks:
- **`tran_read_htX`:** Reads $H_{00}$ and $H_{01}$ from `_htB.dat` or `_htL.dat`/`_htR.dat`.
- **`tran_read_htC`:** Reads $H_{00}$ for the central conductor from `_htC.dat`.
- **`tran_read_htXY`:** Reads coupling Hamiltonians like $H_{LC}$ or $H_{CR}$ from `_htLC.dat`/`_htCR.dat`.

## Important Variables/Constants

- **`transport_type` (Derived Type from `w90_wannier90_types`):** Contains control parameters for transport calculations:
    - `mode` (character): 'bulk' or 'lcr'.
    - `win_min`, `win_max`, `energy_step`: Energy range and step for calculations.
    - `num_bb`, `num_ll`, `num_rr`, `num_cc`, `num_lc`, `num_cr`: Number of WFs in different regions/principal layers for LCR.
    - `num_bandc`: Width of band-diagonal $H_C$ matrix.
    - `read_ht`, `write_ht`: Logicals to control reading/writing of Hamiltonian blocks.
    - `use_same_lead`: For LCR, if left and right leads are identical.
    - `num_cell_ll`, `num_cell_rr`: Number of unit cells in a principal layer of the leads.
    - `group_threshold`: Distance threshold for grouping WFs.
    - `easy_fix`: Logical for a simple parity fix based on signatures.
- **`eta` (complex(kind=dp), parameter):** Small imaginary part added to energy ($E+i\eta$) for calculating retarded Green's functions. Value: `(0.0_dp, 0.0005_dp)`.
- **`nterx` (integer, parameter, value=50):** Maximum iterations for the transfer matrix calculation.
- Hamiltonian blocks: `hB0`, `hB1` (bulk); `hC`, `hCR`, `hL0`, `hL1`, `hLC`, `hR0`, `hR1` (LCR).

## Usage Examples
Transport calculations are invoked by setting `transport = .true.` in `seedname.win` and specifying `transport_mode`.

**Input file `seedname.win` for bulk transport:**
```
transport = .true.
transport_mode = 'bulk'
tran_win_min = -5.0
tran_win_max = 5.0
tran_energy_step = 0.01
num_wann = ...
! Other necessary parameters for wannierisation (kpoints, bands, etc.)
! Real-space Hamiltonian H(R) must be obtainable either from wannierisation
! or by setting tran_read_ht = .true. and providing seedname_htB.dat
```

**Input file `seedname.win` for LCR transport:**
```
transport = .true.
transport_mode = 'lcr'
tran_win_min = -2.0
tran_win_max = 2.0
! Define num_ll, num_rr, num_cc, etc.
! tran_num_cell_ll = ...
! tran_group_threshold = ...
! tran_read_ht = .true. ! Typically true, expecting _htL.dat, _htR.dat, _htC.dat etc.
```
The `tran_main` routine is called from `wannier_prog.F90` if `w90_calculation%transport` is true.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: For `dp`, `cmplx_0`, `cmplx_1`, `cmplx_i`, `pi`, `eps*`.
    - `w90_error`: For error handling.
    - `w90_comms`: For MPI communication (though less prominent than in other modules, used for `comm` type).
    - `w90_io`: For timing and file I/O (`io_file_unit`, `io_date`).
    - `w90_hamiltonian`: To get $H(\mathbf{R})$ if not read from file.
    - `w90_types`: Defines `wannier_data_type`, `print_output_type`, etc.
    - `w90_wannier90_types`: Defines `transport_type`, `w90_calculation_type`, etc.

- **External Libraries:**
    - **LAPACK/BLAS:** `ZGESV` (complex general matrix solve using LU decomposition) is used in `tran_transfer` and `tran_green`. `ZGBSV` (complex general band matrix solve) is used in `tran_lcr`. `ZGEMM` is used for matrix multiplications.

- **Interactions:**
    - Reads real-space Hamiltonian $H(\mathbf{R})$ from `w90_hamiltonian` module or from pre-computed files (`_ht*.dat`).
    - Uses Wannier function centers (`wannier_data%centres`, `wannier_centres_translated`) for sorting and grouping in LCR mode.
    - Reads `seedname.unkg` if LCR mode with signature-based sorting is used.
    - Outputs quantum conductance to `seedname_qc.dat` and density of states to `seedname_dos.dat`.
    - Can write out intermediate Hamiltonian blocks (`_ht*.dat`) and sorted XYZ coordinates (`_centres.xyz`).
```
