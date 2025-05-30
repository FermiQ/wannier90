## Overview
The `w90_readwrite` module serves as a shared utility for parsing input files (`seedname.win`) and handling some common data I/O operations needed by both the main `wannier90.x` executable and the post-processing tool `postw90.x`. Its primary responsibilities include loading the input file into memory, providing routines to extract keywords and data blocks, reading and distributing checkpoint files (`seedname.chk`), and writing standard headers to output files. This module is crucial for setting up calculation parameters and managing persistent data.

## Key Components

### Module `w90_readwrite`
- **Description:** The main module for common input parsing and I/O routines.

### Input File Handling
- **`w90_readwrite_in_file(seedname, error, comm)`:** Loads the `seedname.win` file into an internal character array (`in_data`). It ignores comments and blank lines and converts all text to lowercase. Tabs are converted to spaces.
- **`w90_readwrite_get_keyword(keyword, found, ...)`:** Searches for a specific `keyword` in the loaded input data. If found, it extracts its value (character, logical, integer, or real) and marks the line as processed.
- **`w90_readwrite_get_keyword_vector(keyword, found, length, ...)`:** Similar to `get_keyword` but for reading a vector of values of a given `length`.
- **`w90_readwrite_get_keyword_block(keyword, found, rows, columns, ...)`:** Reads a data block enclosed by `begin keyword` and `end keyword`. It handles unit conversions (e.g., Angstrom to Bohr) if specified within the block.
- **`w90_readwrite_get_block_length(keyword, found, rows, ...)`:** Determines the number of data lines (`rows`) within a specified block.
- **`w90_readwrite_get_range_vector(keyword, found, length, lcount, ...)`:** Reads a vector that can include ranges (e.g., `1,2,5-10`). If `lcount` is true, it only counts the number of elements.
- **`w90_readwrite_clean_infile(stdout, seedname, error, comm)`:** After all expected keywords are read, this checks if any unprocessed lines remain in `in_data`. If so, it prints them and signals an error, indicating unrecognized input.
- **`w90_readwrite_clear_keywords(comm)`:** A helper for `clean_infile` that attempts to "consume" all known keywords from both `wannier90.x` and `postw90.x` to ensure only genuinely unrecognized lines trigger an error.

### Checkpoint File Handling
- **`w90_readwrite_read_chkpt(dis_manifold, exclude_bands, ...)`:** Reads data from a checkpoint file (`seedname.chk`). This includes consistency checks (lattice parameters, k-points, etc.) and loads key data like $U$ matrices, $M$ matrices, Wannier centers, and spreads.
- **`w90_readwrite_chkpt_dist(dis_manifold, wannier_data, ...)`:** Distributes the data read by `read_chkpt` (on the root node) to all other MPI processes.

### Specific Data Reading Routines
A series of `w90_readwrite_read_X` subroutines are provided to parse specific sections or parameters from the input file, often calling the generic `get_keyword` or `get_keyword_block` routines internally. Examples:
- `w90_readwrite_read_verbosity`
- `w90_readwrite_read_num_wann`
- `w90_readwrite_read_exclude_bands`
- `w90_readwrite_read_mp_grid`
- `w90_readwrite_read_kpoints`
- `w90_readwrite_read_lattice`
- `w90_readwrite_read_atoms`
- `w90_readwrite_read_projections` (within `w90_readwrite_get_projections`)
- `w90_readwrite_read_kpath` (within `w90_readwrite_get_keyword_kpath`)
- `w90_readwrite_read_fermi_energy`
- `w90_readwrite_read_dis_manifold` (for disentanglement windows)

### Other Utilities
- **`w90_readwrite_write_header(bohr_version_str, ...)`:** Writes a standard Wannier90 header (version, authors, citation info, date/time) to the specified output unit.
- **`w90_readwrite_uppercase(atom_data, kpoint_path, ...)`:** Converts certain input strings (atom labels, k-point labels, length units) to a consistent uppercase format for output.
- **`w90_readwrite_get_smearing_type(smearing_index)`:** Converts an integer smearing index to a human-readable string.
- **`w90_readwrite_get_convention_type(sc_phase_conv)`:** Converts an integer phase convention index to a string.
- **`w90_readwrite_get_smearing_index(string, keyword, ...)`:** Converts a string representation of smearing type to its integer index.
- **`w90_readwrite_lib_set_atoms(...)`:** A specialized routine to set up atom data when Wannier90 is used as a library.
- **`w90_readwrite_set_kmesh(spacing, reclat, mesh)`:** Calculates k-mesh dimensions based on a desired k-point spacing.

## Important Variables/Constants

- **`in_data(:)` (character(len=maxlen), allocatable, private):** An internal array storing the lines of the input file after initial processing (comments removed, lowercase).
- **`num_lines` (integer, private):** The number of effective lines in `in_data`.
- **`maxlen` (integer, parameter, from `w90_constants`):** Maximum length of a line read from the input file.

## Usage Examples
This module is primarily used internally by `wannier90.x` (`w90_wannier90_readwrite.F90`) and `postw90.x` (`pw90_readwrite.F90`) to manage their respective input parameters.

**Loading and reading the input file:**
```fortran
! In wannier90.x or postw90.x main routines
call w90_readwrite_in_file(seedname, error, comm)
! ... check error ...

! Read a specific integer keyword
integer :: num_iterations
logical :: found_iter
call w90_readwrite_get_keyword('num_iter', found_iter, error, comm, i_value=num_iterations)
! ... check error and if found_iter is true ...

! Read a block of atom positions
call w90_readwrite_get_keyword_block('atoms_frac', found_atoms, num_atoms_expected, 3, &
                                     atoms_frac_data, error, comm)
! ...
call w90_readwrite_clean_infile(stdout, seedname, error, comm) ! Check for unparsed input
```

**Reading a checkpoint file (on root node):**
```fortran
! In wannier_prog.F90
if (on_root) then
  call w90_readwrite_read_chkpt(dis_manifold, exclude_bands, kmesh_info, kpt_latt, &
                                wannier_data, m_matrix, u_matrix, u_matrix_opt, &
                                real_lattice, omega_invariant, mp_grid, num_bands, &
                                num_exclude_bands, num_kpts, num_wann, checkpoint, &
                                have_disentangled, .false., seedname, stdout, error, comm)
  ! ... check error ...
end if
! Distribute data to other nodes
call w90_readwrite_chkpt_dist(dis_manifold, wannier_data, u_matrix, u_matrix_opt, &
                              omega_invariant, num_bands, num_kpts, num_wann, checkpoint, &
                              have_disentangled, error, comm)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: Uses `dp` for precision and `maxlen` for string lengths.
    - `w90_types`: Uses various derived data types (e.g., `atom_data_type`, `kpoint_path_type`, `proj_input_type`, `dis_manifold_type`, `wannier_data_type`, `kmesh_input_type`, `w90_system_type`, `print_output_type`, `ws_region_type`).
    - `w90_comms`: Uses `w90comm_type` and `comms_bcast` (in `w90_readwrite_chkpt_dist`). Error setting routines also use `comms_sync_err`.
    - `w90_error` / `w90_error_base`: For error reporting (`w90_error_type`, `set_error_input`, `set_error_alloc`, etc.).
    - `w90_io`: For `io_file_unit`, `io_date`, `w90_version`.
    - `w90_utility`: For `utility_lowercase`, `utility_frac_to_cart`, `utility_cart_to_frac`, `utility_inverse_mat`, `utility_string_to_coord`, `utility_strip`, `utility_recip_lattice`.

- **External Libraries:** None.

- **Interactions:**
    - This module is called by the main parameter reading routines in `w90_wannier90_readwrite.F90` (for `wannier90.x`) and `pw90_readwrite.F90` (for `postw90.x`).
    - It reads the `seedname.win` input file.
    - It reads and writes `seedname.chk` checkpoint files.
    - The data parsed by this module (e.g., number of bands, k-points, atomic positions, projection definitions) is fundamental for setting up and controlling the entire Wannier90 calculation.
```
