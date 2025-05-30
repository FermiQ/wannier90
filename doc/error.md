## Overview
The `w90_error` module provides a centralized system for error handling within the Wannier90 code. It builds upon the `w90_error_base` module by defining specific error codes and providing a set of subroutines to set these errors. A key feature of this module is its integration with the MPI communication layer (`w90_comms`), ensuring that error states are synchronized across all MPI processes. This prevents situations where one process encounters an error while others continue, which could lead to deadlocks or inconsistent results in parallel execution.

## Key Components

### Module `w90_error`
- **Description:** The main module that defines error codes and provides subroutines for setting and synchronizing error states.

### Error Setting Subroutines
- **General Form:** `set_error_X(err, mesg, comm)`
- **Description:** These subroutines are used to signal different types of errors. They take an allocatable `w90_error_type` object (`err`), an error message string (`mesg`), and an MPI communicator (`comm`). Internally, they call `set_base_error` (from `w90_error_base`) to populate the `err` object with the message and a specific error code. Crucially, they then call `comms_sync_err` to ensure all MPI processes are aware of the error.
- **Specific Routines:**
    - `set_error_fatal(err, mesg, comm)`: For critical errors that should terminate the program.
    - `set_error_alloc(err, mesg, comm)`: For errors occurring during memory allocation.
    - `set_error_dealloc(err, mesg, comm)`: For errors occurring during memory deallocation.
    - `set_error_mpi(err, mesg, comm)`: For errors related to MPI communication itself (though not used directly within `w90_comms` to avoid circular dependencies).
    - `set_error_input(err, mesg, comm)`: For errors related to invalid user input.
    - `set_error_file(err, mesg, comm)`: For errors related to file I/O operations.
    - `set_error_unconv(err, mesg, comm)`: For situations where a numerical procedure fails to converge.
    - `set_error_warn(err, mesg, comm)`: For non-critical warnings (e.g., failure to plot something).

## Important Variables/Constants

These are integer parameters representing specific error codes:
- **`code_ok` (value=0):** Indicates no error.
- **`code_fatal` (value=1):** A fatal error.
- **`code_alloc` (value=2):** Memory allocation error.
- **`code_dealloc` (value=3):** Memory deallocation error.
- **`code_mpi` (value=4):** MPI related error.
- **`code_input` (value=5):** Invalid input error.
- **`code_file` (value=6):** File operation error.
- **`code_unconv` (value=-1):** Failure to converge (non-fatal warning).
- **`code_warning` (value=-2):** General warning (non-fatal).

## Usage Examples
Throughout the Wannier90 codebase, when a recoverable or fatal error condition is detected, one of these subroutines is called.

```fortran
use w90_error, only: set_error_input, w90_error_type
use w90_comms, only: w90comm_type ! Assuming comm is initialized
type(w90_error_type), allocatable :: error_status
type(w90comm_type) :: comm
integer :: user_param

! ... read user_param ...

if (user_param < 0) then
  call set_error_input(error_status, "User parameter cannot be negative.", comm)
  ! The main program would then check if 'error_status' is allocated
  ! and handle the error, usually by printing a message and stopping.
  if (allocated(error_status)) then
     ! call prterr(error_status, stdout, stderr, comm) ! or similar
     stop
  end if
end if
```
If an error is set (e.g., `error_status` becomes allocated), the calling routine is expected to check this and react accordingly, often by propagating the error up or printing a message and terminating using a routine like `prterr` (defined in `wannier_prog.F90`).

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_error_base`: This module is fundamental as it provides the `w90_error_type` derived type and the `set_base_error` subroutine, which is called by all error-setting routines in `w90_error` to actually store the error message and code.
    - `w90_comms`: This module is crucial for parallel execution. The `comms_sync_err` subroutine from `w90_comms` is called by each `set_error_X` routine to ensure that if one MPI process detects an error, all other processes are notified and their error status is updated. This prevents deadlocks and ensures consistent termination or error handling.

- **External Libraries:**
    - None directly. However, it interacts with MPI through the `w90_comms` module.

- **Interactions:**
    - This module is used by almost all other modules in Wannier90. Whenever an operation can fail (e.g., file read, memory allocation, invalid input, numerical instability), routines from `w90_error` are called to signal the problem.
    - The error codes defined here provide a standardized way to categorize errors.
    - The synchronization with `w90_comms` is a key interaction point for robust parallel execution. The main program (`wannier_prog.F90`) typically has a central error printing routine (`prterr`) that checks the error status and prints messages before exiting.
```
