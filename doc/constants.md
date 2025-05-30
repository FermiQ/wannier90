## Overview
The `w90_constants` module defines various constants used throughout the Wannier90 code. These include mathematical constants (e.g., $\pi$), numerical precision parameters, convergence tolerances, and fundamental physical constants (e.g., electron charge, Planck's constant, Bohr radius). The physical constants can be set to values from different CODATA recommendations (e.g., CODATA2006, CODATA2010) via preprocessor flags, ensuring consistency and allowing for updates as more precise values become available.

## Key Components

### Module `w90_constants`
- **Description:** The sole module in this file, it encapsulates all constant definitions and related type declarations.

### Type `w90_physical_constants_type`
- **Description:** A derived data type to group some of the key physical constants and version strings, making them easily passable to other routines, particularly for writing header information.
- **Fields:**
    - `bohr` (real(kind=dp)): The Bohr radius in Angstroms.
    - `bohr_version_str` (character(len=75)): String indicating the source of the Bohr radius value.
    - `constants_version_str1` (character(len=75)): String indicating the CODATA version used.
    - `constants_version_str2` (character(len=75)): String providing the URL for CODATA constants.

### Type `pw90_physical_constants_type`
- **Description:** A more comprehensive derived data type grouping a wider range of physical constants and version strings. This seems intended for passing a larger set of constants.
- **Fields:**
    - `elem_charge_SI` (real(kind=dp)): Elementary charge in Coulombs.
    - `elec_mass_SI` (real(kind=dp)): Electron mass in kg.
    - `hbar_SI` (real(kind=dp)): Reduced Planck constant in J*s.
    - `k_B_SI` (real(kind=dp)): Boltzmann constant in J/K.
    - `eps0_SI` (real(kind=dp)): Vacuum permittivity in F/m.
    - `eV_au` (real(kind=dp)): Conversion factor from eV to atomic units of energy (Hartree).
    - `eV_seconds` (real(kind=dp)): Conversion factor from eV to seconds (related to $\hbar/eV$).
    - `bohr` (real(kind=dp)): The Bohr radius in Angstroms.
    - `bohr_version_str` (character(len=75)): String indicating the source of the Bohr radius value.
    - `constants_version_str1` (character(len=75)): String indicating the CODATA version used.
    - `constants_version_str2` (character(len=75)): String providing the URL for CODATA constants.


## Important Variables/Constants

### Generic Constants
- **`i64` (integer, parameter):** Defines a 64-bit integer kind.
- **`dp` (integer, parameter):** Defines the kind for double precision real numbers (typically `kind(1.0d0)`).
- **`pi` (real(kind=dp), parameter):** The mathematical constant $\pi$.
- **`twopi` (real(kind=dp), parameter):** $2\pi$.
- **`cmplx_i` (complex(kind=dp), parameter):** The imaginary unit $i = (0, 1)$.
- **`cmplx_0` (complex(kind=dp), parameter):** Complex zero $(0, 0)$.
- **`cmplx_1` (complex(kind=dp), parameter):** Complex one $(1, 0)$.
- **`maxlen` (integer, parameter, value=255):** Maximum column width for input file parsing.

### Numerical Convergence Constants
- **`eps2`, `eps5`, `eps6`, `eps7`, `eps8`, `eps10` (real(kind=dp), parameter):** A series of small positive numbers representing various tolerance levels used for numerical comparisons and convergence checks (e.g., $10^{-2}, 10^{-5}, \dots, 10^{-10}$).
- **`smearing_cutoff` (real(kind=dp), parameter, value=10.0):** Cutoff for smearing functions, likely in units of smearing width.
- **`min_smearing_binwidth_ratio` (real(kind=dp), parameter, value=2.0):** A ratio used to decide whether to apply smearing or use direct binning, typically in density of states calculations.

### Physical Constants (SI units and conversions)
The module defines parameters for fundamental physical constants. The specific values depend on the preprocessor flags `CODATA2006` or `CODATA2010`.
- **`elem_charge_SI` (real(kind=dp), parameter):** Elementary charge (e.g., $1.602176565 \times 10^{-19}$ C for CODATA2010).
- **`elec_mass_SI` (real(kind=dp), parameter):** Electron mass (e.g., $9.10938291 \times 10^{-31}$ kg for CODATA2010).
- **`hbar_SI` (real(kind=dp), parameter):** Reduced Planck constant (e.g., $1.054571726 \times 10^{-34}$ J s for CODATA2010).
- **`k_B_SI` (real(kind=dp), parameter):** Boltzmann constant (e.g., $1.3806488 \times 10^{-23}$ J/K for CODATA2010).
- **`bohr_magn_SI` (real(kind=dp), parameter):** Bohr magneton (e.g., $927.400968 \times 10^{-26}$ J/T for CODATA2010).
- **`eps0_SI` (real(kind=dp), parameter):** Vacuum permittivity (e.g., $8.854187817 \times 10^{-12}$ F/m, same for CODATA2006 and CODATA2010).
- **`speedlight_SI` (real(kind=dp), parameter):** Speed of light in vacuum ($299792458.0$ m/s, exact).
- **`eV_au` (real(kind=dp), parameter):** Conversion factor from electronvolts to atomic units of energy (Hartrees).
- **`eV_seconds` (real(kind=dp), parameter):** Electronvolt expressed in seconds (related to $\hbar/eV$, useful for time-frequency conversions).
- **`bohr_angstrom_internal` (real(kind=dp), parameter):** Conversion factor from Bohr radius to Angstroms (e.g., $0.52917721092$ for CODATA2010).
- **`bohr` (real(kind=dp), parameter):** The Bohr radius in Angstroms. Its value can be taken from CODATA or a legacy Wannier90 v1.x value depending on the `USE_WANNIER90_V1_BOHR` preprocessor flag.
- **`constants_version_str1`, `constants_version_str2`, `bohr_version_str` (character, parameter):** Strings used for printing header information about the constants being used.

### Preprocessor Flags
- **`CODATA2006`, `CODATA2010`:** Control which set of physical constant values are compiled. The default is `CODATA2006` if neither is explicitly defined, though `CODATA2010` is preferred if available.
- **`USE_WANNIER90_V1_BOHR`:** If defined, uses a deprecated value for the Bohr radius from Wannier90 version 1.x. Otherwise, uses the value from the selected CODATA set.

## Usage Examples
These constants are used globally throughout the Wannier90 code.

```fortran
! Example of using double precision kind and pi
use w90_constants, only: dp, pi
real(kind=dp) :: area, radius
radius = 2.5_dp
area = pi * radius**2

! Example of using a tolerance
use w90_constants, only: dp, eps8
real(kind=dp) :: difference
difference = value1 - value2
if (abs(difference) < eps8) then
  ! Consider value1 and value2 to be equal
end if

! Example of using a physical constant
use w90_constants, only: dp, bohr
real(kind=dp) :: length_angstroms, length_bohr
length_bohr = 1.0_dp ! Length in Bohr radii
length_angstroms = length_bohr * bohr ! Convert to Angstroms
```
The types `w90_physical_constants_type` and `pw90_physical_constants_type` can be used to pass a bundle of constants to subroutines, for example, when writing output file headers:
```fortran
subroutine write_my_output_header(physics_constants)
  use w90_constants, only: w90_physical_constants_type
  type(w90_physical_constants_type), intent(in) :: physics_constants

  write(*,*) physics_constants%bohr_version_str
  write(*,*) physics_constants%constants_version_str1
  write(*,*) "Bohr radius used:", physics_constants%bohr
end subroutine write_my_output_header
```

## Dependencies and Interactions

- **Internal Dependencies:** None. This module is designed to be self-contained in its definitions.
- **External Libraries:** None.
- **Interactions:**
    - This module is fundamental and is `USE`d by almost every other module in Wannier90 that requires precise numerical values, tolerances, or physical constants.
    - The choice of constants (CODATA2006 vs. CODATA2010, and the Bohr radius version) is determined at compile time via preprocessor flags. This affects the numerical results of calculations involving these constants.
    - The `w90_physical_constants_type` and `pw90_physical_constants_type` are used to pass sets of constants to other parts of the code, often for logging or ensuring consistent use of values.
```
