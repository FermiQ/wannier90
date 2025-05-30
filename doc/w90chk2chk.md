## Overview
The `w90chk2chk.F90` file contains a standalone utility program named `w90chk2chk`. This program is designed to convert Wannier90 checkpoint files (`seedname.chk`) between unformatted (binary, standard) and formatted (text, portable) versions. This is particularly useful when needing to transfer checkpoint files between different computer architectures which might have incompatible binary formats.

The file is structured with three modules:
1.  `w90chk_parameters`: Defines global variables (using `save` attribute) to store data read from or written to the checkpoint file. These variables mirror the main data structures used in Wannier90 for a calculation run.
2.  `wannchk_data`: Seems to be another module for storing global parameters, potentially for data not covered by `w90chk_parameters` or more specific to the main `wannier90.x` calculation details that might also be in a checkpoint file.
3.  `w90_conv`: Contains the core logic for the conversion utility, including command-line parsing, reading from one format, and writing to the other.

## Key Components

### Program `w90chk2chk`
- **Description:** The main program that drives the checkpoint file conversion. It parses command-line arguments to determine the conversion direction (unformatted to formatted, or vice-versa) and the `seedname`.
- **Usage:**
    - `w90chk2chk.x -export [SEEDNAME]` or `w90chk2chk.x -u2f [SEEDNAME]`: Converts `SEEDNAME.chk` (unformatted) to `SEEDNAME.chk.fmt` (formatted).
    - `w90chk2chk.x -import [SEEDNAME]` or `w90chk2chk.x -f2u [SEEDNAME]`: Converts `SEEDNAME.chk.fmt` (formatted) to `SEEDNAME.chk` (unformatted).
    - If `SEEDNAME` is omitted, it defaults to "wannier".
- **Output:** Creates a log file `w90chk2chk.log`.

### Module `w90_conv`
- **Subroutine `conv_get_seedname(stdout, seedname)`:** Parses command-line arguments to get the `seedname` and determines the conversion direction (sets `export_flag`).
- **Subroutine `conv_read_chkpt(checkpoint, stdout, seedname)`:** Reads an unformatted (binary) checkpoint file (`seedname.chk`) and populates the global variables in modules `w90chk_parameters` and `wannchk_data`.
- **Subroutine `conv_read_chkpt_fmt(checkpoint, stdout, seedname)`:** Reads a formatted (text) checkpoint file (`seedname.chk.fmt`) and populates the global variables.
- **Subroutine `conv_write_chkpt(checkpoint, stdout, seedname)`:** Writes the data stored in the global variables to an unformatted (binary) checkpoint file (`seedname.chk`).
- **Subroutine `conv_write_chkpt_fmt(checkpoint, stdout, seedname)`:** Writes the data stored in the global variables to a formatted (text) checkpoint file (`seedname.chk.fmt`).
- **Subroutine `io_error(error_msg, stdout, seedname)`:** A local error handling routine that prints a message and stops the program.
- **Subroutine `print_usage(stdout)`:** Prints usage instructions for the `w90chk2chk.x` executable.
- **Variable `export_flag` (logical, save):** True if converting unformatted to formatted, false otherwise.
- **Variable `header` (character(len=33), save):** Stores the header line read from the input checkpoint file.

### Module `w90chk_parameters`
- **Description:** Declares various global variables (with `save` attribute) that mirror the main data structures stored in a Wannier90 checkpoint file. These include:
    - `wannier_data` (type `wannier_data_type`): Stores Wannier function centers and spreads.
    - `kmesh_info` (type `kmesh_info_type`): Stores k-mesh related information.
    - `dis_manifold` (type `dis_manifold_type`): Stores disentanglement manifold data.
    - `exclude_bands(:)` (integer, allocatable): List of excluded bands.
    - `kpt_latt(:, :)` (real(dp), allocatable): K-point coordinates.
    - `have_disentangled` (logical): Flag for disentanglement status.
    - `num_exclude_bands`, `num_kpts`, `num_bands`, `num_wann` (integer): Dimensions.
    - `eigval(:, :)` (real(dp), allocatable): Eigenvalues.
    - `u_matrix_opt(:, :, :)` (complex(dp), allocatable): $U_{opt}$ matrix from disentanglement.
    - `u_matrix(:, :, :)` (complex(dp), allocatable): $U$ matrix from wannierisation.
    - `mp_grid(3)` (integer): Monkhorst-Pack grid dimensions.
    - `num_proj` (integer): Number of projections.
    - `real_lattice(3,3)`, `recip_lattice(3,3)` (real(dp)): Lattice vectors.
    - `spec_points` (type `kpoint_path_type`): Data for k-point paths (though seemingly unused by the chk read/write logic itself, might be legacy or for future use).

### Module `wannchk_data`
- **Description:** Declares additional global variables, possibly for parameters that are part of the broader Wannier90 calculation state but also included in checkpoints.
    - `w90_calcs` (type `w90_calculation_type`): General calculation parameters.
    - `lsitesymmetry` (logical): Site symmetry flag.
    - `symmetrize_eps` (real(dp)): Tolerance for symmetrization.
    - `fermi_surface_data` (type `fermi_surface_plot_type`): Fermi surface plotting parameters.
    - `tran` (type `transport_type`): Transport calculation parameters.
    - `select_proj` (type `select_projection_type`): Projection selection parameters.
    - `eig_found` (logical): Flag if eigenvalues were found/read.
    - `a_matrix`, `m_matrix_orig`, `m_matrix_orig_local`, `m_matrix`, `m_matrix_local` (complex(dp), allocatable): Various forms of projection and overlap matrices.
    - `omega_invariant` (real(dp)): The $\Omega_I$ spread functional value.

## Important Variables/Constants
The key variables are the global parameters defined within the `w90chk_parameters` and `wannchk_data` modules, as these are what get read from and written to the checkpoint files. The `export_flag` in `w90_conv` controls the direction of conversion.

## Usage Examples
This is a command-line tool.
1.  **To convert binary `.chk` to text `.chk.fmt` (for portability):**
    ```bash
    w90chk2chk.x -export myseedname
    ```
    This reads `myseedname.chk` and creates `myseedname.chk.fmt`.

2.  **To convert text `.chk.fmt` back to binary `.chk` (after transferring):**
    ```bash
    w90chk2chk.x -import myseedname
    ```
    This reads `myseedname.chk.fmt` and creates `myseedname.chk`.

A log file `w90chk2chk.log` is generated during execution.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_types`: Provides the definitions for derived types like `wannier_data_type`, `kmesh_info_type`, `dis_manifold_type`, `kpoint_path_type`.
    - `w90_constants`: For `dp` (double precision kind) and `eps6` (a small tolerance used in `conv_read_chkpt`).
    - `w90_wannier90_types`: Provides definitions for `w90_calculation_type`, `fermi_surface_plot_type`, `transport_type`, `select_projection_type`.
    - `w90_io`: The `w90_conv` module uses `io_file_unit` for getting available file units.

- **External Libraries:** None.

- **Interactions:**
    - This program interacts directly with Wannier90 checkpoint files (`.chk` and `.chk.fmt`).
    - It does not interact with other parts of the Wannier90 codebase at runtime but relies on the data structure definitions from `w90_types` and `w90_wannier90_types` to correctly interpret and write checkpoint file contents.
    - The structure of the checkpoint file (order and type of data) is implicitly defined by the read/write operations in `conv_read_chkpt`, `conv_read_chkpt_fmt`, `conv_write_chkpt`, and `conv_write_chkpt_fmt`. Any changes to the main Wannier90 checkpointing mechanism would require corresponding updates to this utility.
```
