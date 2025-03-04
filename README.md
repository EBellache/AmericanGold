# AmericanGold

# README: Multi-Step Relaxation with Spin Constraints (Log-Periodic Pulsation Approximation)

This set of input scripts demonstrates a **three-step constrained relaxation** approach in **Quantum ESPRESSO** for a **tetragonal AuAmO\(_3\)-based perovskite** with **\(\sim 15\%\)** Hg doping on O sites, strong **spin–orbit coupling**, and **DFT+U** corrections (on Am \(5f\) orbitals). We sequentially lower the spin-constraint penalty to emulate “pulsations” that reorganize the magnetic/skyrmion configuration.

## Table of Contents
1. [Overview](#overview)
2. [Files Description](#files-description)
3. [Logic: Log-Periodic Pulsations via Spin Constraints](#logic)
4. [How to Run](#how-to-run)
5. [Additional Notes](#additional-notes)

---

## 1. Overview <a name="overview"></a>

In many advanced materials, especially those with **non-collinear magnetism** (e.g., skyrmions or strongly spin–orbit-coupled states), we want to stabilize a specific spin texture. Real-time time-dependent DFT is often too expensive. Instead, we approximate **time-dependent pulses** by performing **multiple relaxation cycles** with gradually decreasing spin-constraint strength. 

This approach:
- Begins with a **large penalty** on atomic spin directions to hold a chosen spin configuration (like a partial swirl or topological pattern).
- **Relaxes** the structure while the spins remain near that forced orientation.
- Moves to a smaller penalty, letting spins partially adapt to local energy minima.
- Ends with a very small or zero penalty, hopefully retaining a metastable spin texture reminiscent of the final “pulsed” state.

We then do a **final SCF** with unconstrained magnetization for the desired property calculations (band structure, DOS, topological indices, etc.).

---

## 2. Files Description <a name="files-description"></a>

### `relax1.in`
- **Purpose**: First relaxation with **high penalty** (`lambda=10.0`) in `constrained_magnetization='atomic direction'`.
- **What it does**: Strongly enforces your chosen initial spin directions across the doping supercell. The lattice is tetragonal (`ibrav=6`), and we have 10 atoms: 2 Au, 2 Am, 5 O, 1 Hg doping an O site.
- **Key parameters**:
  - `nspin=4`, `noncolin=.true.`, `lspinorb=.true.` for non-collinear + SOC.
  - `lda_plus_u=.true.` with `Hubbard_l(2)=3` and `Hubbard_u(2)=4.0` to handle Am \(5f\) correlation.
  - `lambda=10.0` ensures the initial spin angles (either default or specified) remain near enforced directions.

### `relax2.in`
- **Purpose**: Second relaxation cycle with **moderate penalty** (`lambda=3.0`).
- **What it does**: Uses `restart_mode='restart'` so it picks up the wavefunction/geometry from `relax1.in`. Now the system can deviate more from the forced spin directions, partially relaxing magnetization and positions. 
- **Key parameters**: 
  - Similar system and HPC parameters, but `lambda=3.0` in &SYSTEM.
  - Typically fewer steps to converge, since geometry is partly relaxed.

### `relax3.in`
- **Purpose**: Third relaxation cycle with **low penalty** (`lambda=1.0`) or smaller.
- **What it does**: The final stage of “log periodic pulses.” The system can further refine the structure while keeping a mild spin constraint. 
- **After**: We typically do a final SCF with `constrained_magnetization='none'` or a tiny `lambda=0.2` to get a stable or metastable spin-texture solution.

---

## 3. Logic: Log-Periodic Pulsations via Spin Constraints <a name="logic"></a>

In experimental setups, we might apply **time-dependent fields** in multiple pulses of decreasing amplitude and frequency to reorganize a skyrmion. Here, we approximate that by:

1. **High penalty**: (like the “largest amplitude pulse”) – the system’s spins remain near the artificially imposed angles. 
2. **Intermediate penalty**: (the second or “middle amplitude pulse”) – partial freedom to move. 
3. **Low penalty**: (final or “lowest amplitude pulse”) – the system can almost settle into the shape it “prefers,” but not so strongly that it reverts to a trivial ferromagnet.

This staged approach systematically shakes the system from an artificially forced configuration into a stable or metastable pattern that might represent a topological spin arrangement.

---

## 4. How to Run <a name="how-to-run"></a>
1**Run Relaxation Steps**:
   - **relax1**: 
     ```bash
     mpirun -np 64 pw.x -in relax1.in > relax1.out
     ```
     Wait for it to converge (`nstep=200` is a max). Check final geometry in `relax1.out`.
   - **relax2**: 
     ```bash
     mpirun -np 64 pw.x -in relax2.in > relax2.out
     ```
     Make sure `prefix` and `restart_mode='restart'` are consistent so it continues from `'AuAmHg_relax1.save'`.
   - **relax3**:
     ```bash
     mpirun -np 64 pw.x -in relax3.in > relax3.out
     ```
     This final partial constraint hopefully yields a near-skyrmion configuration.

2.**Final SCF**:
   - After relax3, you typically create a separate `scf.in` with `constrained_magnetization='none'` (or a minimal `lambda`) to let the system finalize the spin distribution. Something like:
     ```bash
     &CONTROL
       calculation = 'scf'
       prefix      = 'AuAmHg_final'
       ...
     /
     &SYSTEM
       ...
       constrained_magnetization='none'
       ...
     /
     ```
   - Then run:
     ```bash
     mpirun -np 64 pw.x -in scf.in > scf.out
     ```
   - Now you have a final wavefunction + geometry reflecting your “pulsation-driven” spin arrangement.

3.**Analysis**:
   - You can proceed with band structure, DOS, Berry curvature, etc., reading the final wavefunction from `'AuAmHg_final.save/'`.

---

## 5. Additional Notes <a name="additional-notes"></a>

1. **Spin Angles**:  
   - If you want a specific swirl or chiral texture, define multiple species (Au1, Au2, etc.) or set `angle1(i)=..., angle2(i)=...` for each species. Large `lambda` ensures the code keeps those angles. Over multiple runs, you reduce `lambda`.

2. **Doping Setup**:
   - Here we forcibly replaced 1 O with 1 Hg in a 2×1×1 or single tetragonal cell. For better doping realism, use a supercell or check multiple doping sites for distribution effects.

3. **DFT+U + SOC**:
   - We used `lda_plus_u=.true.` with `U=4.0` eV on Am’s \(5f\). Tuning U is crucial for correct localization. If Au or Hg also have partially filled orbitals, consider a small U for them as well.
   - The overhead of `noncolin` + `lspinorb` + `DFT+U` is significant, so keep k-mesh moderate at first. Refine once you see stable convergence.

4. **Convergence**:
   - Each relaxation might take tens to hundreds of SCF steps. If you see slow or oscillatory behavior, adjust `conv_thr`, `mixing_beta`, or the k-point mesh.  
   - Check the final total energies in `relax1.out`, `relax2.out`, `relax3.out` to see if the system is truly converged.

5. **Runtime**:
   - On a 64-core HPC node, each relaxation might run a few hours to a day, depending on system size, cutoffs, etc. Typically test smaller cutoffs or fewer k-points, then scale up for final production runs.

---

**We hope this clarifies** how to implement a multi-step spin-constrained approach for approximating log-periodic pulsations in Quantum ESPRESSO.