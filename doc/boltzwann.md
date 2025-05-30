## Overview
The `w90_boltzwann` module, part of `postw90.x`, calculates semi-classical transport coefficients using the Boltzmann Transport Equation (BTE) within the constant relaxation time approximation. It primarily computes the Transport Distribution Function (TDF), and from it, derives electrical conductivity, Seebeck coefficient, and electronic contribution to thermal conductivity as a function of chemical potential ($\mu$) and temperature ($T$). The methodology is based on the work by Pizzi et al., Comp. Phys. Comm. 185, 422 (2014). It can also optionally compute the density of states (DOS).

## Key Components

### Module `w90_boltzwann`
- **Description:** The main module for Boltzmann transport property calculations.

### Subroutine `boltzwann_main`
- **Description:** This is the top-level driver routine for the BoltzWann module. It initializes arrays for temperature and chemical potential grids, calls `calcTDFandDOS` to compute the TDF (and DOS if requested), and then integrates the TDF with the Fermi-Dirac distribution derivative to obtain transport coefficients (electrical conductivity $\sigma$, Seebeck coefficient $S$, and thermal conductivity $\kappa_e$) over the specified $(\mu, T)$ grid. Results are written to various output files.
- **Parameters:**
    - `pw90_boltzwann` (type `pw90_boltzwann_type`): Contains all BoltzWann specific input parameters (energy ranges, k-mesh, smearing, etc.).
    - Other parameters are similar to those in `w90_berry_main`, providing system information, Wannier function data, Hamiltonian and matrix elements (e.g., `HH_R`, `SS_R` if spin-decomposed), `eigval`, `real_lattice`, etc.
- **Returns:** Generates output files: `seedname_tdf.dat`, `seedname_elcond.dat`, `seedname_sigmas.dat` ($\sigma S$), `seedname_seebeck.dat`, `seedname_kappa.dat`, and optionally `seedname_boltzdos.dat`.

### Subroutine `calcTDFandDOS`
- **Description:** Calculates the Transport Distribution Function (TDF) $\Sigma_{\alpha\beta}(E) = \sum_{\mathbf{k},n} \tau v_{n\mathbf{k},\alpha} v_{n\mathbf{k},\beta} (-\frac{\partial f_0}{\partial E})_{E=E_{n\mathbf{k}}}$ (where $\tau$ is the constant relaxation time, often factored out or set to 1 fs by default, and the derivative of Fermi function is replaced by a smearing function of $E-E_{n\mathbf{k}}$). It iterates over a k-mesh, computes band energies $E_{n\mathbf{k}}$ and group velocities $v_{n\mathbf{k},\alpha} = \frac{1}{\hbar} \frac{\partial E_{n\mathbf{k}}}{\partial k_\alpha}$ using routines from `w90_wan_ham`, and then bins these contributions onto an energy grid to form the TDF. If `pw90_boltzwann%calc_also_dos` is true, it also computes the DOS using routines from `w90_dos`.
- **Parameters:** Similar to `boltzwann_main`, plus `TDF` (output), `TDFEnergyArray` (energy grid for TDF).
- **Returns:** Populates the `TDF` array and, if requested, the `DOS_all` array (which is then written to file).

### Subroutine `TDF_kpt`
- **Description:** Calculates the k-point resolved contribution to the TDF. For each band $n$ at a given k-point, it computes $v_{n\mathbf{k},\alpha} v_{n\mathbf{k},\beta}$ and adds its contribution to the TDF, smeared by a function of $(E - E_{n\mathbf{k}})$. It handles spin decomposition if `spin_decomp` is true.
- **Parameters:** `pw90_boltzwann`, `eig_k` (eigenvalues at current k), `deleig_k` (band derivatives $\partial E/\partial k$ at current k), `EnergyArray` (energy grid for TDF), `TDF_k` (output k-point contribution).
- **Returns:** Populates `TDF_k`.

### Function `MinusFermiDerivative(E, mu, KT)`
- **Description:** Calculates the negative derivative of the Fermi-Dirac distribution function with respect to energy, $-\frac{\partial f_0}{\partial E} = \frac{1}{kT} \frac{e^{(E-\mu)/kT}}{(e^{(E-\mu)/kT}+1)^2}$. This function is used as the weighting factor when integrating the TDF to get transport coefficients.
- **Parameters:** `E` (energy), `mu` (chemical potential), `KT` ($k_B T$).
- **Returns:** The value of $-\frac{\partial f_0}{\partial E}$.

## Important Variables/Constants

- **`pw90_boltzwann_type` (from `postw90_types.F90`):**
    - `kmesh%mesh(3)`: Dimensions of the k-mesh for BoltzWann interpolation.
    - `tdf_energy_step`, `dos_energy_step`: Energy step for TDF and DOS grids.
    - `tdf_smearing%type_index`, `tdf_smearing%fixed_width`: Smearing type and width for TDF.
    - `dos_smearing%use_adaptive`, `dos_smearing%adaptive_prefactor`, `dos_smearing%adaptive_max_width`: Adaptive smearing parameters for DOS.
    - `mu_min`, `mu_max`, `mu_step`: Range and step for chemical potential.
    - `temp_min`, `temp_max`, `temp_step`: Range and step for temperature.
    - `relax_time` (real(kind=dp)): Constant relaxation time (in fs).
    - `bandshift`, `bandshift_firstband`, `bandshift_energyshift`: Parameters for applying a scissor operator.
    - `dir_num_2d`: If non-zero, specifies calculation for a 2D system (1=x, 2=y, 3=z is non-periodic direction).
- **Tensor component indices (`XX`, `XY`, `YY`, `XZ`, `YZ`, `ZZ`):** Parameters (1 to 6) for accessing components of symmetric 3x3 tensors stored in packed format.
- **Output arrays:** `TDF`, `ElCond`, `SigmaS`, `Seebeck`, `Kappa`.

## Usage Examples
This module is invoked by `postw90.x` by setting `boltzwann = .true.` in `seedname.win`.

**Example `seedname.win` section for BoltzWann:**
```
boltzwann = .true.
boltz_kmesh = 50 50 50  ! Dense k-mesh for interpolation
boltz_tdf_energy_step = 0.001
boltz_mu_min = -1.0
boltz_mu_max = 1.0
boltz_mu_step = 0.05
boltz_temp_min = 100
boltz_temp_max = 500
boltz_temp_step = 100
boltz_relax_time = 10.0 ! in fs
boltz_calc_also_dos = .true. ! Optionally calculate DOS
dos_smr_type = 'adaptive'   ! Smearing for DOS (if calc_also_dos)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `w90_comms`: For MPI communication (gathering results, broadcasting).
    - `w90_constants`: For `dp`, physical constants through `pw90_physical_constants_type`.
    - `w90_dos`: Uses `dos_get_k` and `dos_get_levelspacing` if DOS calculation is requested.
    - `w90_io`: For file I/O (`io_file_unit`) and timing.
    - `w90_utility`: For matrix inversion (`utility_inv3`, `utility_inv2`) and smearing functions (`utility_w0gauss`).
    - `w90_error`: For error handling.
    - `w90_types`: For shared data structures.
    - `w90_postw90_types`: For `postw90`-specific types like `pw90_boltzwann_type`.
    - `w90_get_oper`: To get real-space Hamiltonian (`HH_R`) and spin matrices (`SS_R`).
    - `w90_wan_ham`: For `wham_get_eig_deleig` to get band energies and their k-derivatives (velocities).
    - `w90_spin`: If `spin_decomp` is true, uses `spin_get_nk` for spin-projected DOS.

- **External Libraries:**
    - **LAPACK/BLAS:** Used implicitly via matrix operations in utility and Hamiltonian modules.

- **Interactions:**
    - `boltzwann_main` is the entry point called by `postw90.x`.
    - Requires MLWFs and real-space Hamiltonian matrix elements from a prior `wannier90.x` run.
    - Computes band velocities by interpolating the Wannier Hamiltonian on a dense k-mesh defined by `boltz_kmesh`.
    - Outputs several files (`seedname_tdf.dat`, `seedname_elcond.dat`, etc.) containing the calculated transport properties.
    - The accuracy of the results heavily depends on the quality of the MLWFs and the density of the interpolation k-mesh.
```
