# HeH+ Potential Energy Surface with VQE

This project solves the Qiskit Fall Fest 2026 quantum chemistry side challenge:

> Plot the potential energy surface of the HeH+ molecule and defend the choice of basis set, fermion-to-qubit mapping, ansatz, and optimizer.

The notebook calculates the ground-state energy of HeH+ over bond lengths from 0.5 to 3.0 angstrom using the Variational Quantum Eigensolver (VQE). Exact diagonalization provides a reference at every geometry.

![HeH+ potential energy curve](HeH_PES.png)

## Results

| Quantity | Result |
| --- | ---: |
| Bond-length range | 0.5 to 3.0 angstrom |
| Number of geometries | 26 |
| Grid minimum | 0.90 angstrom |
| Minimum VQE energy | -2.86261758 Hartree |
| Largest VQE error | 5.42e-7 Hartree |
| Energy at 3.0 angstrom | -2.80783461 Hartree |
| Approximate binding gap to 3.0 angstrom | 0.05478297 Hartree, about 1.49 eV |

The VQE and exact curves overlap at the plotted scale. The reported equilibrium distance is the minimum on a grid with 0.1 angstrom spacing, not a fully optimized molecular geometry.

## Method

For each bond length, the notebook:

1. Defines HeH+ with two electrons using `charge=1` and `spin=0`.
2. Uses PySCF to generate one-electron and two-electron integrals.
3. Maps the electronic Hamiltonian to a four-qubit operator.
4. Uses VQE to minimize the electronic energy.
5. Adds the nuclear repulsion energy to obtain the total molecular energy.
6. Compares the result with exact diagonalization.
7. Reuses the optimized parameters as the initial point for the next bond length.

## Choice of basis set: STO-3G

STO-3G is a minimal basis set. In this calculation it produces two spatial orbitals, or four spin orbitals, so the full electronic problem requires only four qubits with the selected mapping.

This basis was chosen because the challenge focuses on demonstrating the quantum chemistry workflow and explaining the algorithmic choices. STO-3G keeps every geometry fast enough to scan and benchmark during a hackathon while still producing the expected bound-state minimum and dissociation trend.

The tradeoff is limited quantitative accuracy. A larger basis such as 6-31G would describe the electronic wavefunction more accurately but would increase the orbital count, qubit count, circuit size, and optimization cost. Therefore, the equilibrium distance and binding energy reported here should be interpreted as STO-3G model results rather than high-accuracy experimental predictions.

## Choice of mapping: Jordan-Wigner

The Jordan-Wigner mapping assigns one qubit to each spin orbital and translates fermionic creation and annihilation operators into Pauli operators. It maps this problem to four qubits.

Jordan-Wigner was selected for transparency and reliability. The occupation of each spin orbital has a direct qubit interpretation, which makes the Hartree-Fock state, particle number, and mapped Hamiltonian easier to inspect and explain. It also works directly with Qiskit Nature's Hartree-Fock and UCCSD circuits without an additional symmetry-selection step.

Parity mapping could reduce the qubit count by exploiting particle-number symmetries. That is an attractive option for hardware execution, but Jordan-Wigner is preferable here because four qubits remain inexpensive on a statevector simulator and the direct orbital-to-qubit correspondence makes the demonstration easier to verify.

## Choice of ansatz: Hartree-Fock plus UCCSD

The circuit starts from the Hartree-Fock state and applies the unitary coupled-cluster singles and doubles ansatz, UCCSD. For this two-electron, two-spatial-orbital problem, the ansatz contains only three parameters.

UCCSD was chosen because it is based on electronic excitations rather than a generic hardware-efficient circuit. It preserves the chemistry structure of the problem and provides a clear connection to established coupled-cluster methods. Starting from Hartree-Fock with all variational parameters set to zero gives the optimizer a physically meaningful initial state.

The results support this choice: UCCSD reproduces the exact diagonalization curve with a maximum observed error of approximately 5.42e-7 Hartree. The tradeoff is that UCCSD circuits become deep and expensive as the number of orbitals grows. That limitation is small for HeH+ but important for larger molecules and noisy hardware.

## Choice of optimizer: SLSQP

SLSQP is a deterministic, gradient-based classical optimizer. It was chosen because `StatevectorEstimator` provides a smooth, noiseless objective function and the UCCSD circuit has only three parameters. At the 0.8 angstrom test geometry, the optimizer reached the minimum in 14 function evaluations.

The potential-energy scan also uses a warm start: the optimal parameters from one bond length initialize the next calculation. Neighboring molecular geometries have similar ground states, so this reduces repeated optimization work and helps maintain a smooth curve.

SLSQP is less suitable for noisy quantum-hardware measurements because shot noise can make numerical gradients unreliable. SPSA or another noise-tolerant optimizer would be a stronger choice for a hardware experiment. This project uses SLSQP because its goal is to evaluate the chemistry and ansatz accuracy on a noiseless simulator.

## Validation

Every VQE result is compared with `NumPyMinimumEigensolver` using the molecular problem's particle-number filter. The total energy from the exact solver already includes nuclear repulsion. For the direct VQE calculation, the notebook adds nuclear repulsion to the electronic eigenvalue explicitly before comparing the two methods.

The close agreement confirms that the optimizer converged and that the selected UCCSD ansatz represents the ground state accurately within the STO-3G model. It does not demonstrate quantum advantage because both algorithms run on a classical computer and the molecular model is intentionally small.

## Repository contents

- `QuantumChemistry.ipynb`: complete calculation and analysis
- `HeH_PES.png`: potential-energy curve
- `HeH_PES_results.csv`: numerical data for all geometries
- `requirements.txt`: Python dependencies

## Run the notebook

Create and activate a virtual environment, then install the dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Start Jupyter and run the cells in `QuantumChemistry.ipynb` from top to bottom:

```bash
jupyter lab
```

The scan writes `HeH_PES_results.csv` and `HeH_PES.png` to the current working directory.

## Possible extensions

- Refine the grid near the minimum to improve the equilibrium-distance estimate.
- Compare STO-3G with a larger basis set.
- Compare Jordan-Wigner with parity mapping and symmetry reduction.
- Repeat selected geometries with a shot-based or noisy estimator.
- Test SPSA and error-mitigation methods for hardware execution.

## References

- [Qiskit Nature getting started](https://qiskit-community.github.io/qiskit-nature/getting_started.html)
- [Qiskit Nature VQE with a UCC ansatz](https://qiskit-community.github.io/qiskit-nature/howtos/vqe_ucc.html)
- [Qiskit Nature ground-state solvers](https://qiskit-community.github.io/qiskit-nature/tutorials/03_ground_state_solvers.html)
