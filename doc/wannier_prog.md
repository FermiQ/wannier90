## Overview
This file contains the main program `wannier`, which drives the Wannier90 code. Wannier90 is a program for calculating maximally-localised Wannier functions (MLWFs) from a set of Bloch states obtained from an electronic structure calculation. It plays a central role in post-processing the results of quantum mechanical simulations to obtain a chemically intuitive understanding of bonding and to compute various material properties.

## Key Components

### Program `wannier`
- **Description:** The main executable program for the Wannier90 code. It orchestrates the entire process of reading input, performing calculations (disentanglement, wannierisation, plotting, transport), and writing output.
- **Parameters:** Takes a seedname as a command-line argument, which is used as the base for input (`.win`), output (`.wout`), and other auxiliary files.
- **Returns:** Does not return a value in the traditional sense, but produces various output files containing Wannier functions, band structures, densities of states, and other computed properties. It also writes log files (`.wout`, `.werr`).

### Subroutine `prterr`
- **Description:** A utility subroutine defined within the `wannier` program to handle and print error messages. It is used to report errors encountered during the execution and terminate the program gracefully, especially in a parallel (MPI) environment.
- **Parameters:**
    - `error`: An object of type `w90_error_type` containing details about the error.
    - `stdout`: Integer unit number for standard output.
    - `stderr`: Integer unit number for standard error.
    - `comm`: An object of type `w90comm_type` for MPI communication.
- **Returns:** Does not return a value but prints error messages to standard error and standard output, and then stops the program.

## Important Variables/Constants

- **`seedname` (character(len=50)):** The base name for input and output files (e.g., `wannier.win`, `wannier.wout`).
- **`num_wann` (integer):** The number of Wannier functions to be computed.
- **`num_bands` (integer):** The number of Bloch bands read from the electronic structure calculation.
- **`num_kpts` (integer):** The number of k-points in the Brillouin zone.
- **`real_lattice(3,3)` (real(kind=dp)):** The real-space lattice vectors.
- **`recip_lattice(3,3)` (real(kind=dp)):** The reciprocal-space lattice vectors.
- **`kpt_latt(:,:)` (real(kind=dp), allocatable):** Coordinates of the k-points.
- **`eigval(:,:)` (real(kind=dp), allocatable):** Eigenvalues of the Bloch states at each k-point.
- **`m_matrix_orig_local (:,:,:,:)` (complex(kind=dp), allocatable):** Overlap matrices $ M_{mn}^{(\mathbf{k,b})} = \langle u_{m\mathbf{k}} | u_{n\mathbf{k+b}} \rangle $ between Bloch states at neighboring k-points.
- **`a_matrix (:,:,:)` (complex(kind=dp), allocatable):** Projection matrices $ A_{mn}^{(\mathbf{k})} = \langle \psi_{m\mathbf{k}} | g_n \rangle $ of Bloch states onto trial localized orbitals.
- **`u_matrix_opt (:,:,:)` (complex(kind=dp), allocatable):** Unitary matrices that define the optimal subspace in disentanglement.
- **`u_matrix (:,:,:)` (complex(kind=dp), allocatable):** Unitary matrices that rotate the Bloch states to obtain the MLWFs.
- **`comm` (type(w90comm_type)):** Handles MPI communication.
- **`on_root` (logical):** True if the current process is the root MPI process, false otherwise.
- **`dryrun` (logical):** If true, the program performs a dry run, checking inputs and setup without performing extensive calculations.
- **Many type variables (e.g., `atom_data_type`, `wann_control_type`, `kmesh_info_type`):** These are derived data types (structures) that hold various control parameters, system information, and data arrays used throughout the calculation. They are defined in the various `w90_*.F90` modules.

## Usage Examples
The `wannier` program is typically run from the command line after an initial electronic structure calculation (e.g., from Quantum ESPRESSO, VASP, Siesta).

1.  **Prepare input file:** Create an input file named `seedname.win` (e.g., `wannier.win`) that specifies parameters for the Wannier function calculation, such as the number of Wannier functions, k-point mesh, disentanglement windows, and which properties to calculate.
2.  **Run the program:**
    ```bash
    wannier90.x seedname
    ```
    (The executable might be named `wannier90.x` or simply `wannier` depending on the installation.)
3.  **Analyze output:** The program will generate `seedname.wout` (main log file), `seedname.chk` (checkpoint file), files containing Wannier functions (e.g., `seedname_u.mat`, `seedname_u_dis.mat`), band structure plots (`seedname.eig`, `seedname_band.dat`, `seedname_band.kpt`), and other requested outputs.

## Dependencies and Interactions

- **Internal Dependencies:** (Modules used via `USE` statements)
    - `w90_constants`: Defines physical constants and precision (`dp`).
    - `w90_error`: Handles error reporting and management.
    - `w90_types`: Defines core data types used throughout Wannier90.
    - `w90_io`: Handles input/output operations, including reading the `.win` file.
    - `w90_hamiltonian`: Deals with the Hamiltonian in the Wannier function representation.
    - `w90_kmesh`: Manages the k-point mesh and nearest-neighbor k-point information.
    - `w90_disentangle`: Performs the disentanglement procedure to select an optimal subspace of Bloch states.
    - `w90_overlap`: Calculates and manages overlap matrices ($M^{(\mathbf{k,b})}$) and projection matrices ($A^{(\mathbf{k})}$).
    - `w90_wannierise`: Performs the iterative procedure to minimize the spread of the Wannier functions.
    - `w90_plot`: Generates data for plotting band structures, Wannier functions, Berry curvature, etc.
    - `w90_transport`: (If enabled) Calculates transport properties like anomalous Hall conductivity.
    - `w90_comms`: Handles MPI communication.
    - `w90_sitesym`: (If enabled) Deals with site symmetries.
    - `w90_readwrite`: Contains subroutines for reading and writing various data, including checkpoint files.
    - `w90_wannier90_types`: Specific types for the main wannier90 program.
    - `w90_wannier90_readwrite`: Specific read/write routines for the main program.

- **External Libraries:**
    - **MPI (Message Passing Interface):** Used for parallel execution on multiple processors. The code includes preprocessor directives (`#ifdef MPI`) to enable MPI support and can use different MPI interfaces (`mpi_f08`, `mpi`, or `mpif.h`).
    - **LAPACK/BLAS:** Implicitly used for linear algebra operations, although not directly visible in `wannier_prog.F90`, these are typically linked during compilation and used by various numerical routines.

- **Interactions:**
    - **Input Files:** Reads parameters and data from `seedname.win`, `seedname.mmn`, `seedname.amn`, `seedname.eig`, and potentially other files generated by an ab initio code.
    - **Output Files:** Writes results to `seedname.wout`, `seedname.werr`, `seedname.chk`, `seedname_u.mat`, `seedname_u_dis.mat`, plotting data files, and transport data files.
    - **Ab Initio Codes:** `wannier_prog.F90` (Wannier90) is designed to work as a post-processing tool for various electronic structure codes (e.g., Quantum ESPRESSO, VASP, Siesta, etc.). These codes provide the necessary Bloch states, eigenvalues, and overlap information.
    - The program follows a sequential workflow:
        1. Read input parameters.
        2. Read Bloch state information (overlaps, projections, eigenvalues).
        3. (Optional) Perform disentanglement if `num_bands > num_wann`.
        4. Perform Wannierisation to obtain MLWFs.
        5. (Optional) Perform plotting of band structures, Wannier functions, etc.
        6. (Optional) Perform transport calculations.
