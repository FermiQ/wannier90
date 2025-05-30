## Overview
The `w90_io` module in Wannier90 is dedicated to handling various input/output operations and timing functionalities. This includes parsing command-line arguments, providing routines to get the current date and time, managing file units for opening files, and offering stopwatch-like features to measure CPU and wall-clock time consumed by different parts of the code. It also defines the version string for Wannier90.

## Key Components

### Module `w90_io`
- **Description:** The main module for all I/O and timing related functionalities.

### Subroutine `io_commandline(prog, dryrun, seedname)`
- **Description:** Parses the command-line arguments provided when Wannier90 or postw90 is executed. It identifies flags like `-pp` (post-processing), `-dryrun`, `-h` (help), `-v` (version), and extracts the `seedname` for the calculation.
- **Parameters:**
    - `prog` (character(len=50), intent(in)): Name of the calling program (e.g., "wannier90", "postw90").
    - `dryrun` (logical, intent(out)): Set to true if a dry run is requested.
    - `seedname` (character(len=50), intent(inout)): The base name for input/output files.
- **Returns:** Sets `dryrun`, `seedname`, and `post_proc_flag` (module variable). Prints help or version info and stops if requested.

### Subroutine `io_get_seedname(seedname)`
- **Description:** A simpler routine specifically to extract the `seedname` from command-line arguments. It also checks for the `-pp` flag to set `post_proc_flag`. If `seedname.win` is passed, it strips the `.win` extension.
- **Parameters:**
    - `seedname` (character(len=50), intent(inout)): The base name for input/output files.
- **Returns:** Sets `seedname` and `post_proc_flag`.

### Subroutine `io_date(cdate, ctime)`
- **Description:** Retrieves the current date and time from the system and formats them into human-readable strings.
- **Parameters:**
    - `cdate` (character(len=9), intent(out)): The formatted date (e.g., "DDMonYYYY").
    - `ctime` (character(len=9), intent(out)): The formatted time (e.g., "HH:MM:SS").

### Function `io_time()`
- **Description:** Returns the elapsed CPU time in seconds since its first call within the program execution. It uses the standard Fortran `cpu_time` intrinsic.
- **Returns:** `real(kind=dp)`: Elapsed CPU time.

### Function `io_wallclocktime()`
- **Description:** Returns the elapsed wall-clock time in seconds since its first call. It uses the standard Fortran `system_clock` intrinsic.
- **Returns:** `real(kind=dp)`: Elapsed wall-clock time.

### Function `io_file_unit()`
- **Description:** Finds and returns an unused Fortran unit number that can be safely used to open a new file. It starts checking from unit 10 upwards.
- **Returns:** `integer`: An available unit number.

### Stopwatch Subroutines
- **`io_stopwatch_start(tag, timers)`:** Starts or restarts a timer associated with a given `tag`. Records the start CPU time and increments the call count for that tag.
- **`io_stopwatch_stop(tag, timers)`:** Stops the timer associated with `tag` and accumulates the elapsed CPU time since the corresponding `io_stopwatch_start` call.
- **`io_print_timings(timers, stdout)`:** Prints a formatted table of all recorded stopwatch timers to the specified `stdout` unit, showing the tag, number of calls, and total CPU time for each.
- **Parameters:**
    - `tag` (character(len=*), intent(in)): A string identifying the code section being timed.
    - `timers` (type(timer_list_type), intent(inout)): A derived type (`timer_list_type` from `w90_types`) that stores information for multiple timers.
    - `stdout` (integer, intent(in)): The unit number for outputting the timings table.

## Important Variables/Constants

- **`post_proc_flag` (logical, public, save):** A module-level variable that is set to `.true.` if the `-pp` (post-processing mode) flag is detected in the command-line arguments. This flag indicates that Wannier90 should primarily focus on generating files needed by an ab-initio code for calculating overlaps and projections, rather than performing a full Wannier function calculation.
- **`w90_version` (character(len=10), parameter, public):** A constant string holding the version of Wannier90 (e.g., "3.1.0 ").

## Usage Examples

**Parsing command line and getting seedname:**
```fortran
! In wannier_prog.F90
use w90_io, only: io_commandline, io_get_seedname
character(len=50) :: seed_name = "wannier" ! Default
logical :: is_dry_run

! Full parsing (handles -h, -v, -dryrun, -pp)
call io_commandline("wannier90", is_dry_run, seed_name)
! or simpler seedname extraction:
! call io_get_seedname(seed_name)
! if (post_proc_flag) then ...
```

**Timing a code section:**
```fortran
use w90_io, only: io_stopwatch_start, io_stopwatch_stop, io_print_timings
use w90_types, only: timer_list_type ! Assuming timer_list_type is defined here
type(timer_list_type) :: my_timers
integer :: out_unit = 6 ! Standard output

call io_stopwatch_start("MyExpensiveLoop", my_timers)
! ... code for the expensive loop ...
call io_stopwatch_stop("MyExpensiveLoop", my_timers)

! At the end of the program or a major section:
call io_print_timings(my_timers, out_unit)
```

**Getting a free file unit:**
```fortran
use w90_io, only: io_file_unit
integer :: new_unit
new_unit = io_file_unit()
open(unit=new_unit, file="my_output.dat", status="unknown")
```

**Getting date and time for logging:**
```fortran
use w90_io, only: io_date
character(len=9) :: current_date, current_time
call io_date(current_date, current_time)
write(*,*) "Calculation started on ", trim(current_date), " at ", trim(current_time)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_constants`: Uses `dp` for defining the precision of real types used in timing functions.
    - `w90_types`: Uses `timer_list_type` and `nmax` (max number of timers) for the stopwatch functionality. (The definition of `timer_list_type` is expected to be in `w90_types`).

- **External Libraries:** None. It uses standard Fortran intrinsics like `command_argument_count`, `get_command_argument`, `cpu_time`, `date_and_time`, `system_clock`, and `inquire`.

- **Interactions:**
    - This module is used early in the program execution (in `wannier_prog.F90`) to parse command-line arguments and determine the `seedname` and operational mode (e.g., `post_proc_flag`, `dryrun`).
    - The timing functions (`io_time`, `io_wallclocktime`, and the stopwatch routines) are used throughout the code to measure performance of different sections.
    - `io_file_unit` is used whenever a new file needs to be opened to avoid conflicts with existing unit numbers.
    - `io_date` is often used for logging purposes, for example, in writing headers to output files.
    - The `w90_version` constant is used when printing version information or in output file headers.
```
