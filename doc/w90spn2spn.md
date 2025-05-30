## Overview
The `w90spn2spn.F90` file provides a utility program named `w90spn2spn`. This tool is designed to convert Wannier90 spin matrix files (`seedname.spn`) between unformatted (binary) and formatted (text) versions. This conversion capability is essential for transferring these files between different computer systems that may not share the same binary file formats, ensuring portability of spin-related data.

The file is structured with two main modules:
1.  `w90spn_parameters`: Defines global variables (using the `save` attribute) to store the dimensions (`num_bands`, `num_kpts`) read from the `.spn` file.
2.  `w90_conv_spn`: Contains the core logic for the conversion utility, including command-line argument parsing, reading the `.spn` file in one format, and writing it out in the other. It also stores the actual spin matrix data (`spn_o`).

## Key Components

### Program `w90spn2spn`
- **Description:** The main executable that orchestrates the conversion of `.spn` files. It parses command-line arguments to determine the `seedname` and the direction of conversion (unformatted to formatted or vice-versa).
- **Usage:**
    - `w90spn2spn.x -export [SEEDNAME]` or `w90spn2spn.x -u2f [SEEDNAME]`: Converts `SEEDNAME.spn` (unformatted) to `SEEDNAME.spn.fmt` (formatted).
    - `w90spn2spn.x -import [SEEDNAME]` or `w90spn2spn.x -f2u [SEEDNAME]`: Converts `SEEDNAME.spn.fmt` (formatted) to `SEEDNAME.spn` (unformatted).
    - If `SEEDNAME` is omitted, it defaults to "wannier".
- **Output:** Creates a log file named `w90spn2spn.log`.

### Module `w90_conv_spn`
- **Subroutine `conv_get_seedname(stdout, seedname)`:** Parses command-line arguments to determine the `seedname` and sets the `export_flag` to indicate the conversion direction.
- **Subroutine `conv_read_spn(stdout, seedname)`:** Reads an unformatted (binary) `.spn` file. It first reads the header and dimensions (`num_bands`, `num_kpts`), then allocates and reads the spin matrix elements into the global `spn_o` array. It reconstructs the full matrix from the lower triangle (or upper, depending on storage convention) assuming Hermiticity for off-diagonal elements.
- **Subroutine `conv_read_spn_fmt(stdout, seedname)`:** Reads a formatted (text) `.spn.fmt` file. Similar to `conv_read_spn`, it reads header, dimensions, and then the spin matrix elements into `spn_o`.
- **Subroutine `conv_write_spn(stdout, seedname)`:** Writes the data stored in the global `spn_o` array to an unformatted (binary) `.spn` file. It writes the header, dimensions, and then the lower triangle of the spin matrices.
- **Subroutine `conv_write_spn_fmt(stdout, seedname)`:** Writes the data from `spn_o` to a formatted (text) `.spn.fmt` file.
- **Subroutine `io_error(error_msg, stdout, seedname)`:** A local error handling routine that prints an error message and stops the program.
- **Subroutine `print_usage(stdout)`:** Prints the command-line usage instructions for `w90spn2spn.x`.
- **Variable `export_flag` (logical, save):** If true, conversion is from unformatted to formatted. If false, from formatted to unformatted.
- **Variable `header` (character(len=60), save):** Stores the header line read from the input `.spn` or `.spn.fmt` file.
- **Variable `spn_o(:, :, :, :)` (complex(kind=dp), allocatable, save):** A 4D array storing the spin matrices. Dimensions are likely (num_bands, num_bands, num_kpts, 3) for the three Pauli matrices.

### Module `w90spn_parameters`
- **Description:** This module holds global parameters read from the header of the `.spn` file.
- **Variables:**
    - `num_kpts` (integer, save): Total number of k-points.
    - `num_bands` (integer, save): Number of bands.

## Important Variables/Constants
- **`spn_o` array in `w90_conv_spn`:** This is the primary data structure holding the spin matrices $\langle u_{m\mathbf{k}} | \sigma_i | u_{n\mathbf{k}} \rangle$.
- **`num_bands`, `num_kpts` in `w90spn_parameters`:** Store the dimensions of the spin matrices.
- **`export_flag` in `w90_conv_spn`:** Controls the direction of the file format conversion.

## Usage Examples
This is a command-line utility.

1.  **To convert a binary `.spn` file to a formatted text file (e.g., for transferring to a different system):**
    ```bash
    w90spn2spn.x -export myrun
    ```
    This will read `myrun.spn` and create `myrun.spn.fmt`.

2.  **To convert a formatted text `.spn.fmt` file back to a binary `.spn` file (e.g., after transferring):**
    ```bash
    w90spn2spn.x -import myrun
    ```
    This will read `myrun.spn.fmt` and create `myrun.spn`.

A log file `w90spn2spn.log` is generated during the program's execution.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_types`: Likely used implicitly if any types from it were needed, though not explicitly `use`d in the provided snippet for `w90spn_parameters` (it's used in the module definition).
    - `w90_constants`: Uses `dp` for double precision kind and `eps6` (though `eps6` is not directly used in the provided snippet, it might be used in parts of `w90_io` it calls).
    - `w90_io`: The `w90_conv_spn` module uses `io_file_unit` for obtaining available file unit numbers.

- **External Libraries:** None.

- **Interactions:**
    - The program interacts with `.spn` (binary) and `.spn.fmt` (formatted text) files. These files contain matrix elements of the Pauli spin matrices in the basis of Bloch states, typically generated by an ab initio code interfaced with Wannier90 when `spinors = .true.` and spin-related properties are calculated.
    - The utility ensures that users can move these spin data files between systems with potentially different binary representations.
```
