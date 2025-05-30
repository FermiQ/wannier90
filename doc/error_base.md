## Overview
The `w90_error_base` module provides the foundational elements for error handling in Wannier90. It defines the basic derived type `w90_error_type` used to store error information (a code and a message). It also contains the core subroutine `set_base_error` to allocate and populate this error type, and a special routine `recv_error` for MPI synchronization purposes. This module serves as the lowest level of the error handling mechanism, with more specific error codes and higher-level handling routines built upon it in the `w90_error` module.

## Key Components

### Module `w90_error_base`
- **Description:** The main module defining the basic error data structure and low-level error setting routines.

### Type `w90_error_type`
- **Description:** A derived data type designed to encapsulate error information.
- **Fields:**
    - `code` (integer): An integer code representing the type of error.
    - `message` (character(len=256)): A human-readable string describing the error.
- **Contains:**
    - `final :: untrapped_error`: A finalizer subroutine.

### Subroutine `set_base_error(err, mesg, code)`
- **Description:** This is the fundamental subroutine for signaling an error. It allocates an object of `w90_error_type` and populates it with the provided error message and integer code.
- **Parameters:**
    - `err` (type(w90_error_type), allocatable, intent(out)): The error object to be allocated and filled.
    - `mesg` (character(len=*), intent(in)): The error message string.
    - `code` (integer, intent(in)): The integer code for the error.
- **Returns:** Allocates `err` and sets its `message` and `code` components.

### Subroutine `recv_error(err)`
- **Description:** A special-purpose subroutine primarily used by the `w90_comms` module for MPI error synchronization. When a process learns of an error on another MPI rank (without knowing the specific details initially), this routine allocates an error object and sets its code to `code_remote`.
- **Parameters:**
    - `err` (type(w90_error_type), allocatable, intent(out)): The error object to be allocated.
- **Returns:** Allocates `err` and sets `err%code` to `code_remote`.

### Subroutine `untrapped_error(err)`
- **Description:** A finalizer subroutine for the `w90_error_type`. This routine is intended to be called if an `w90_error_type` object is deallocated without the error it represents being properly handled (i.e., "trapped" and processed). It writes a message to standard error (unit 0) and stops the program. This acts as a safeguard against silently ignored errors.
- **Parameters:**
    - `err` (type(w90_error_type), intent(in)): The error object being finalized.
- **Returns:** Prints an "UNTRAPPED ERROR" message and stops execution.

## Important Variables/Constants

- **`code_remote` (integer, parameter, value=-99):** A special error code used to indicate that an error was triggered on a different MPI rank. This is used by `recv_error`.

## Usage Examples
The `set_base_error` routine is not typically called directly by most of the Wannier90 code. Instead, the higher-level routines from the `w90_error` module (like `set_error_fatal`, `set_error_input`, etc.) are used, which in turn call `set_base_error`.

However, a conceptual example of direct usage would be:
```fortran
use w90_error_base
type(w90_error_type), allocatable :: my_error_status
character(len=100) :: detailed_message
integer :: my_specific_error_code

! ... some condition detected ...
detailed_message = "A very specific problem occurred here."
my_specific_error_code = 101 

call set_base_error(my_error_status, detailed_message, my_specific_error_code)

! Later, check and handle the error
if (allocated(my_error_status)) then
  write(*,*) "Error Code:", my_error_status%code
  write(*,*) "Message:", trim(my_error_status%message)
  ! ... potentially stop or attempt recovery ...
  deallocate(my_error_status) ! This would call untrapped_error if not handled
end if
```
The `recv_error` subroutine is used internally by `w90_comms` during error synchronization.
The `untrapped_error` finalizer is invoked automatically by the Fortran runtime if an allocated `w90_error_type` variable goes out of scope or is deallocated without being explicitly handled (e.g., by checking `if (allocated(err))` and then `deallocate(err)`).

## Dependencies and Interactions

- **Internal Dependencies:** None. This module is self-contained and provides the basic error structure.
- **External Libraries:** None.

- **Interactions:**
    - This module is the foundation for the `w90_error` module. `w90_error` uses `w90_error_type` and calls `set_base_error` to create error objects with specific codes and messages relevant to different error conditions (fatal, input, file I/O, etc.).
    - The `w90_comms` module uses `recv_error` as part of its MPI error synchronization mechanism (`comms_sync_err`). If `comms_sync_err` detects that an error occurred on another process but the local process has `ierr=0`, it calls `recv_error` to set a local error flag indicating a remote error.
    - The Fortran runtime interacts with the `untrapped_error` finalizer. If an `w90_error_type` object is destroyed while still holding error information (i.e., it was allocated and not subsequently deallocated after being checked), `untrapped_error` is automatically called.
```
