# Lattice mold calculations of mW Ice Ih nucleation rate at T=220K

````{note}
In this section `/` is the package's root folder.
````
Here we provide a detailed instructions to calculate the nucleation rate of Ice Ih with the mW model at $T=220K$ and $p=1bar$ using the Lattice Mold technique that requires the `square/well pair_style` available in LAMMPS. 

The data file (`39mold.xyz`), the potential file for mW (`mW.sw`) and LAMMPS script (`mw_lattmold.in`) are provided in the directory `/examples/mw_mold/` and in this work example we will navigate through those files to explain them in detail.

````{note}
The mW pair style is part of the MANYBODY package. See the [Build package](https://docs.lammps.org/Build_package.html) page for more info.
````

The lattice mold technique consists of 5 different steps. All the steps can be found in {footcite:t}`sanchez2022homogeneous`, and they can be summarized as:  

1. Create an appropriate spheroidal mold and embed it in the liquid at the correct density
2. Choice of the optimal well radius $r_{0w}$ to extrapolate, so two water molecules fit inside the same well.
3. Calculation of the [well occupancy](#well-occupancy) curves for each well depth below the optimal radius. 
4. Calculation of the [average nucleation time](#average-nucleation-time) to obtain the nucleation rate at each well radius.
5. [Extrapolation of the nucleation rate](#extrapolation-of-the-nucleation-rate) to the optimal radius $r_{0w}$.

The configuration (step 1) can be created easily using the liquid and crystal configuration at the corresponding $(p,T)$ conditions for the desired Ice (Ih in this example). Here, we provide the system data file of a mold made of 39 wells at $T=220K$ and $p=1bar$ (see the figure below). 

![Step-1](../figs/LatticeMold/Fig1.png "Conf_LM")

The optimal radius is easily calculated by running a simulation of a single well and sweeping the well radii for a fixed depth of $\sim8k_BT$. The radius above which the occupancy exceeds the 100%, *i.e.* more than one water molecule can access the well, is considered as the optimal radius. For the current example we have to extrapolate to $r_{0w}=1.32Å$ (see figure below).

![Step-2](../figs/LatticeMold/Fig2.jpg "Opt_Rad")


## Well occupancy 

The calculation of the well occupancy for different well radii under the optimal well radius consists of the following steps:

1. Create the directory for sweeping different radii ($r_w=0.85,0.99,1.12Å$), and in each directory, create a for each well depth considered for the calculation. This is a truncated range of values of $\epsilon$ in $k_{B}T$:

```
0.00001
0.10
0.20
...
2.4
2.5
3.5
5.0
8.0
```
Please note that the number of well depths included in the calculation may increase to capture with more accuracy the transition of well filling. This should vary with the well radius.

2. Copy the LAMMPS script file (`mw_lattmold.in`) in each subdirectory along with the configuration file (`39mold.xyz `) and the mW potential file (`mW.sw`).
3. The LAMMPS script contains several variables that are important to know to properly perform the simulations:

```
# ---------------------------- Define variables --------------------------------
variable  nts          equal  99000000   # production number of time-steps
variable  T            equal  220.0      # Temperature of the system in K
variable  nkT          equal  8          # Well depth in kT
variable  ts           equal  1          # length of the ts (in fs units)
variable  width        equal  0.25       # (reduced) width of the square well potential
variable  alpha        equal  0.017025   # exponent of the square well potential (0.005*3.405)
variable  seed         equal  23782      # velocity seed
variable  NtsTdamp     equal  100        # Number of ts to damp temperature
variable  thermoSteps  equal  1000       # Number of ts to write properties on screen
variable  restartSteps equal  1000000    # Number of ts before write restart file
variable  dumpSteps    equal  250000     # Number of ts before write dump file


# --------------------- Derivate variables -------------------------------------
variable D           equal ${nkT}*8.314*$T/4184    # Depth of well (real units)
variable rw          equal ${width}*3.405          # Width of the well (real units)
variable cutoff_well equal  ${rw}*4.0              # Well potential cutoff (real units)


####   Define Mold   ####
read_data      39mold.xyz  # contains the positions and masses and other information
group freeze type 2 
group unfrozen subtract all freeze

```

For this step, the typical run must be approximately 5 ns (with `dt=1 fs`), and that can be controlled by the parameter `nts` which must be set to `nts=5000000`. 
Regarding the interaction potential, the parameter `width` stands for the well radius so this must be changed for the different studies radii during this step `width=0.25,0.29,0.33` in reduced units. 
The parameter `nkT` gives the well depth in $k_BT$  and must sweep the values presented above. 
Regarding the velocity seed, the variable `seed` controls the initial velocity seed. 
Also, there are some variables that might be interesting to know: 
- `thermoSteps` gives the number of timesteps to print the thermos. 
- `restartSteps` indicates the frequency of saving the restart files.
- `dumpSteps` is the number of steps to save the trajectory in the dump file and for this step can be set above 200000 as it is not required in this step.


5. Launch the simulation for each radius and well depth.
6. The `thermo_style` is configured to show some magnitudes that are crucial for the calculation of the well occupancy curves. We need to get the average number of well occupancy for each value of `nkT` so that we print the potential contribution due to mW-well interaction (`c_1`, column 8):

```
# ------------- Output thermo information and averaged variables ---------------
compute 1 all pair square/well
compute 2 all pair sw
compute mytemp unfrozen temp
thermo_style   custom step c_mytemp pe etotal press vol enthalpy c_1 c_2 spcpu  # energy and thermo properties printed to output
thermo_modify  flush yes  # flush the buffer and write the output in real time
thermo         ${thermoSteps}      # how often (in steps) will write the properties of thermo_style to the output
```

Thus, the calculation of the well occupancy for each depth can be estimated easily by taking the average over all the simulation of this value:
  
$$\langle Nw \rangle=4184\cdot c_1 /(nkT\cdot 8.314\cdot T)$$

6. Plot the different curves of well occupancy for the different radii. The result should look similar to the figure below

![Step-3](../figs/LatticeMold/Fig3.jpg "Well_ocu")

The free energy difference between the liquid and the liquid with the precritical cluster is calculated as 

$$\Delta G^*=N_w\cdot\epsilon_{max}-\int_{\epsilon_0}^{\epsilon_{max}}d\epsilon_{sw}\, \langle N_{sw}(\epsilon_{sw})\rangle,$$

where $N_{w}$ is the total number of wells in the mold ($39$), $\epsilon_{max}$ is the maximum well depth to evaluate this integral, 
$\langle N_{sw}(\epsilon_{sw})\rangle$ is the average number of mold sites occupied by liquid molecules obtained previously, 
and $\epsilon_0$ is the minimum value of epsilon considered in the calculation (close to zero). 
The final calculation of the free energy difference must include the rotational and translational degrees of freedom such that

$$\Delta G/k_B T=\Delta G^*/k_B T + \ln(\rho_f V_w)- \ln(8\pi^2),$$

where $\rho_f$ is the fluid number density, and $V_w$ is the volume of a single well.



## Average nucleation time 

The calculation of the optimal radius for extrapolation of the interfacial energy includes the following steps:

## Extrapolation of the nucleation rate 

The calculation of the optimal radius for extrapolation of the interfacial energy includes the following steps:


## Thermodynamic integration 

Once the optimal radius is estimated, the next step consists in thermodynamic integration of different radii above the optimal value of $r_w$. The calculation of the interfacial energy for the different well radii includes the following steps:

1. Create a directory for each radius to be integrated ($r_w=0.33,0.34,0.35\sigma$) and in each directory, create a for each well depth considered for the calculation. This is a truncated range of values of $\epsilon$ in $k_{B}T$:

```
0.00001
0.1
0.2
...
1.9
2.0
2.3
2.6
3
3.5
4
4.5
5
6
7
8
```

2. Copy the LAMMPS script file (`lj_mold.in`) in each subdirectory along with the configuration file (`mold_100.lmp`).

3. The variables of the LAMMPS script presented in previous section need to be changed slightly. For this step, the typical run must be of the order of hundred of thousands time-steps (with `dt=1e-3`), controlled by the parameter `nts`. Regarding the interaction potential, the parameter `rw` that controls the well radius must be changed for the different radii `rw=0.33,0.34,0.35` (in $\sigma$). The parameter `nkT` (well depth) must change it for each simulation with the corresponding value. Also, the `thermoSteps` should have a reasonable value (1000 is recommended), and `dumpSteps` can be set above 50000 timesteps as the trajectory is not needed for this step

4. Launch the simulation for each radius and well depth. 

5. 5.	The `thermo_style` is configured to show some magnitudes that are crucial for the calculation of the well occupancy curves. We need to get the average number of well occupancy for each value of `nkT` so that we print the potential contribution due to mW-well interaction (`c_1`, column 8):

```
# ------------- Output thermo information and averaged variables ---------------
variable well equal c_1*count(all)
variable nall equal count(all)
compute mytemp melt temp
compute 1 all pair square/well
thermo    ${thermoSteps}
thermo_style  custom step pe epair press ke c_mytemp lx ly lz pxx pyy pzz c_1 v_well v_nall spcpu density
```


````{note}
For real units the multiplication by the number of particles in the system is not necessary.
````

The calculation of the well occupancy for each depth can be estimated easily by taking the average over all the simulation of this value:

$$\langle Nw \rangle=c_1\cdot n_all/(nkT\cdot T)$$


````{note}
Please note that the system requires a time to reach the steady state so that the analysis must be performed discarding after $t\approx10\tau$. This equilibration time may vary depending on the system under study (water, hard-spheres, salt…)
````

In the following figure the curves of well occupancy vs. well depth for the different radii are presented.


![Step-2\label{Occupancy}](../figs/Fig3.png)

## Extrapolation and interfacial energy calculation

After the analysis in the previous step, one obtain a curve of well occupancy vs well depth for each radius so that the interfacial energy is calculated as

$$\gamma(r_w )=\frac{1}{2l^2 } \left[N_w\cdot\epsilon_{max}-\int_{\epsilon_0}^{\epsilon_{max}}d\epsilon\, N_w(\epsilon) \right],$$
where $N_w$ is the total number of wells and $l$ is the short side of the box that can be obtained from the thermo (`lx`, `ly`, columns 7 and 8 int the `thermo`). The resulting integrals are provided in the following table:

|         $r_w/\sigma$)        |  0.33 |  0.34 |  0.35 |
|:----------------------------:|:-----:|:-----:|:-----:|
| $\gamma/\sigma^{-2}\epsilon$ | 0.363 | 0.357 | 0.348 |

To obtain the interfacial energy, you now shall extrapolate the value of the interfacial energy to the optimal radius ($r_{0w}=0.32\sigma$) using a linear fit. According to the interfacial energy provided in the table the interfacial energy is

$$\gamma=0.370(8) \epsilon\sigma^{−2}$$

![Step-3](../figs/Fig4.png "Extrapolation")

This mold integration reported for the same system an interfacial energy of $\gamma=0.372(8) \epsilon\sigma^{−2}$ extrapolating to an optimal radius of $r_{0w}=0.315\sigma$ (please see the work by Espinosa *et al*{footcite:t}`espinosa2014mold`). Additionally, another work using the cleaving technique{footcite:t}`davidchack2003direct` reported a value of $\gamma=0.371(3) \epsilon\sigma^{−2}$ for the same system.




```{footbibliography}

```