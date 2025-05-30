## Overview
The `w90_comms` module provides a set of wrappers for MPI (Message Passing Interface) routines. Its purpose is to centralize and simplify parallel communication operations within the Wannier90 code. It handles different MPI interface standards (MPI_F08, MPI90, legacy mpif.h) through preprocessor directives and provides both error-synchronized and non-error-synchronized versions of common MPI calls like broadcast, reduce, scatter, gather, send, and receive. This abstraction layer makes the main codebase cleaner and facilitates easier management of parallel execution.

## Key Components

### Module `w90_comms`
- **Description:** The main module encapsulating all MPI wrapper subroutines, derived types for communicators and statuses, and interface blocks for generic procedure calls.

### Derived Type `w90comm_type`
- **Description:** A wrapper for the MPI communicator.
- **Fields:**
    - `comm`: Stores the MPI communicator handle (either `type(mpi_comm)` for MPI_F08 or `integer` for older interfaces).

### Derived Type `w90stat_type`
- **Description:** A wrapper for the MPI status object.
- **Fields:**
    - `stat`: Stores the MPI status (either `type(mpi_status)` for MPI_F08, `integer :: stat(MPI_STATUS_SIZE)` for MPI90, or a dummy integer if MPI is not used).

### Generic Interfaces (e.g., `comms_bcast`, `comms_reduce`)
- **Description:** Provide generic names for communication routines that can operate on different data types (integer, real, complex, logical, character) and array ranks. These interfaces map to specific module procedures. For example, `comms_bcast` can call `comms_bcast_int`, `comms_bcast_real`, etc.

### Subroutines with `comms_` prefix (e.g., `comms_bcast_int`, `comms_reduce_real`)
- **Description:** These are the main communication routines that wrap MPI calls. They typically include a call to `comms_sync_err` to ensure all MPI processes have a consistent error state before proceeding with the communication.
- **Parameters:** Typically include the data array, its size/count, MPI communicator (`comm`), and an error object (`error`). Routines like `reduce` also take an operation type (`op` like 'SUM', 'PRD').
- **Returns:** Perform the MPI operation and set the `error` object if an MPI error occurs.

### Subroutines with `comms_no_sync_` prefix (e.g., `comms_no_sync_bcast_int`, `comms_no_sync_reduce_real`)
- **Description:** These are versions of the communication routines that *do not* perform the explicit error synchronization via `comms_sync_err`. They are intended for use in specific situations where error handling is managed differently or for performance reasons, but require careful usage.
- **Parameters & Returns:** Similar to their `comms_` counterparts but without the initial `comms_sync_err` call.

### Function `mpirank(comm)`
- **Description:** Returns the rank of the calling process within the communicator.
- **Parameters:** `comm` (type `w90comm_type`).
- **Returns:** Integer rank of the process (0 if not using MPI).

### Function `mpisize(comm)`
- **Description:** Returns the total number of processes in the communicator.
- **Parameters:** `comm` (type `w90comm_type`).
- **Returns:** Integer size of the communicator (1 if not using MPI).

### Subroutine `comms_sync_err(comm, error, ierr)`
- **Description:** Synchronizes the error state across all MPI processes. If any process reports an error (`ierr /= 0`), all processes will have their `error` object allocated (or `ierr` set if `DISABLE_ERROR_SYNC` is defined). This ensures consistent error handling in parallel execution.
- **Parameters:** `comm`, `error` (allocatable error object), `ierr` (local error flag).

### Subroutine `comms_array_split(numpoints, counts, displs, comm)`
- **Description:** Calculates how to distribute `numpoints` elements of an array among the available MPI processes. It returns `counts` (number of elements per process) and `displs` (displacement for each process's chunk in a global array), which are suitable for use with `MPI_Scatterv` or `MPI_Gatherv`.
- **Parameters:** `numpoints` (total elements), `counts` (output array of counts per process), `displs` (output array of displacements), `comm`.

## Important Variables/Constants

- **`code_mpi` (integer, parameter, value=4):** An error code specific to MPI errors, used with the `w90_error_base` module.
- **`mpi_send_tag` (integer, parameter, value=77):** An arbitrary tag used for MPI send/receive operations.
- **`root_id` (integer, parameter, value=0):** The rank of the root process, typically 0.
- **Preprocessor Macros (`MPI`, `MPI08`, `MPI90`, `MPIH`, `DISABLE_ERROR_SYNC`):** These control which MPI interface is used and whether error synchronization is active. They are defined at compile time.

## Usage Examples
The routines in this module are used throughout Wannier90 whenever data needs to be exchanged or aggregated across MPI processes.

**Example: Broadcasting data from root to all processes:**
```fortran
use w90_comms
type(w90comm_type) :: comm
type(w90_error_type), allocatable :: error
integer :: my_rank, data_value

my_rank = mpirank(comm)
if (my_rank == root_id) then
  data_value = 123 ! Only root sets the value
end if

! Broadcast data_value from root to all other processes
call comms_bcast(data_value, 1, error, comm)
if (allocated(error)) call prterr(error, stdout, stderr, comm) ! Handle error

! Now all processes have data_value = 123
```

**Example: Reducing data (summing) from all processes to root:**
```fortran
real(kind=dp) :: local_sum, global_sum
local_sum = ... ! Calculated on each process

! Sum local_sum from all processes and store in global_sum on root
call comms_reduce(local_sum, 1, 'SUM', error, comm)
if (allocated(error)) call prterr(error, stdout, stderr, comm)

if (mpirank(comm) == root_id) then
  ! global_sum now holds the total sum on the root process
  print *, "Total sum:", global_sum
end if
```

**Example: Splitting an array for parallel processing:**
```fortran
integer :: num_total_items, my_node_id, num_nodes
integer, allocatable :: counts(:), displs(:)
num_nodes = mpisize(comm)
my_node_id = mpirank(comm)

allocate(counts(0:num_nodes-1))
allocate(displs(0:num_nodes-1))

call comms_array_split(num_total_items, counts, displs, comm)

! Each process now knows its share: counts(my_node_id)
! and its starting offset: displs(my_node_id)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: Uses `dp` for defining the precision of real/complex types.
    - `w90_error_base`: Uses `w90_error_type` and `set_base_error` for error handling. The `recv_error` function is also mentioned (though its definition is likely in `w90_error_base` or `w90_error`).

- **External Libraries:**
    - **MPI (Message Passing Interface):** This module is fundamentally dependent on an MPI library being available if the `MPI` preprocessor macro is defined. It interfaces with MPI using specific Fortran bindings (`mpi_f08`, `mpi`, or `mpif.h`).

- **Interactions:**
    - This module is used by virtually all other modules in Wannier90 that perform parallel operations or need to distribute/collect data.
    - It provides the basic building blocks for parallel data management, ensuring that data is correctly broadcast, reduced, scattered, or gathered across different MPI processes.
    - The error synchronization (`comms_sync_err`) plays a crucial role in maintaining consistent program state and behavior in a parallel environment. If one process encounters an error, others are made aware, preventing deadlocks or divergent behavior.
    - The choice of MPI interface (via preprocessor flags) allows Wannier90 to be compiled with different MPI implementations and standards.
```
