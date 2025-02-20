# DNA Molecular Dynamics Simulation Using AMBER

This procedure is adapted from the AMBER tutorial B1 for DNA molecular dynamics simulations. It outlines the steps to generate, minimize, equilibrate, and run molecular dynamics simulations for a DNA molecule using AMBER.
For detailed tutorial, visit its official website: https://ambermd.org/tutorials/basic/tutorial1/

## Prerequisites

Before starting, set up your environment by running the following commands in your terminal:

```bash
export AMBERHOME=/path/to/amber16
export PATH=$PATH:$AMBERHOME/bin
source $AMBERHOME/amber.sh
```

## 1. Creating the DNA Structure
Use the NAB program in AMBER to generate the DNA structure with a selected sequence.

Create the `a-dna.nab` text file:
```
molecule m;
m = fd_helix( "adna", "cccgcgccc", "dna" );
putpdb( "a-dna-9base.pdb", m, "-wwpdb");
```

Run the following commands on your terminal:

```
$AMBERHOME/bin/nab a-dna.nab
./a.out
```

## 2. Creating Input Files
Run AMBER tleap with the DNA force field:

```
$AMBERHOME/bin/tleap -s -f $AMBERHOME/dat/leap/cmd/leaprc.DNA.bsc1
```

In the interactive prompt, enter:

```
> source leaprc.water.tip3p
> adna = loadpdb a-dna-9base.pdb
> addions adna Na+ 0
> saveamberparm adna a-dna_charges_only.prmtop a-dna_charges_only.inpcrd
> solvateoct adna TIP3PBOX 8.0
> addionsrand Na+ 15 Cl- 15
> saveamberparm adna a-dna_wat.prmtop a-dna_wat.inpcrd
> quit
```

## 3. Minimization Steps

### a. Minimizing Only the Water Molecules

First, minimize the water molecules in your system.

Create `a-dna_min1.in` with:
```
&cntrl
 imin=1, ntx=1, irest=0, ntpr=1, ntf=1, ntxo=1,
 ntb=1, cut=10.0, nsnb=10, ntr=1,
 maxcyc=1000, ncyc=500, ntmin=1,
 restraintmask=':1-34', restraint_wt=25.0,
&end
```

Then run this on the terminal:
```
$AMBERHOME/bin/sander -O -i a-dna_min1.in -o a-dna_min1.out -p a-dna_wat.prmtop -c a-dna_wat.inpcrd -r a-dna_min1.rst -ref a-dna_wat.inpcrd
```

### b. Minimizing Water Molecules and Counterions

Next, minimize everything except the DNA molecule.

Create `a-dna_min2.in` with:
```
Minimizing the system 
with 25 kcal/mol restraints on DNA, 500 steps of steepest descent and 500 of conjugated gradient
 &cntrl
 imin=1, ntx=1, irest=0, ntpr=1, 
 ntf=1, ntxo=1, ntb=1, cut=10.0,
 nsnb=10, ntr=1, maxcyc=1000,ncyc=500,
 ntmin=1,restraintmask=':1-18',restraint_wt=25.0,
 &end
 &ewald
   ew_type = 0, skinnb = 1.0,
 &end
```

Then run the following command on the terminal:
```
mpirun –np 12 sander.MPI -O -i a-dna_min2.in -o a-dna_min2.out -p a-dna_wat.prmtop -c a-dna_min1.rst -r a-dna_min2.rst -ref a-dna_min1.rst
```

### b. Minimizing the Whole System

Create `a-dna_min3.in` with:
```
Minimizing whole system without restraints on DNA, 1500 steps of steepest descent and 1500 of conjugated gradient
 &cntrl
 imin=1,ntx=1,irest=0,ntpr=1, 
 ntf=1,ntxo=1,ntb=1,cut=10.0,
 nsnb=10,ntr=0,maxcyc=3000,ncyc=1500,
 ntmin=1,
 &end
 &ewald
   ew_type = 0, skinnb = 1.0,
 &end
```

## 4. Equilibration and Molecular Dynamics

### a. Heating the System

Generate `heat.in` input file with:

```
Heating the system with 25 kcal/mol restraints on DNA and counterions, V=const
 &cntrl
 imin=0, ntx=1, ioutfm=0, ntpr=5, 
 ntwr=500, ntwx=500, ntwe=500,
 nscm=5000,
 ntf=2, ntc=2,
 ntb=1, ntp=0,ntxo=1,
 nstlim=50000, t=0.0, dt=0.002,
 cut=10.0,
 tempi=0.0, ntt=1,
 ntr=1,nmropt=1,
 restraintmask=':1-34',
 restraint_wt=25.0,
 &end
 &wt type='TEMP0',istep1=0,
  istep2=5000, value1=0.0,value2=300.0,
 &end
 &wt type='TEMP0',istep1=5001,
   istep2=50000, value1=300.0,value2=300.0,  
 &end
 &wt type='END',  &end
```

Next, run the following command on the terminal:

```
mpirun –np 12 $AMBERHOME/bin/sander.MPI -O -i heat.in -o a-dna_heat.out -p a-dna_wat.prmtop -c a-dna_min3.rst -r a-dna_heat.rst -ref a-dna_min3.rst -x a-dna_heat.mdcrd
```

### b. Equilibrating the System

b.	Generate `equal.in` input file with:
```
Equilibrating the system with 0.5 kcal/mol restraints
on DNA, during 50ps, at constant T= 300K & P=1ATM and coupling = 0.2
 &cntrl
 imin=0, ntx=5, ntpr=100,
 ntwr=500, ntwx=500, 
 ntwe=500,nscm=5000,ntf=2, ntc=2,
 ntb=2, ntp=1, tautp=0.2, taup=0.2,
 nstlim=25000,t=0.0, dt=0.002,
 cut=10.0,
 ntt=1,ntxo=1,ioutfm=0,
 ntr=1,irest=1,restraintmask=':1-18',
 restraint_wt=0.5,
 &end
 &ewald
 ew_type = 0, skinnb = 1.0,
 &end
```

Then, run the following commands on the terminal:

```
mpirun –np 12 $AMBERHOME/bin/sander.MPI -O -i equal.in -o a-dna_equal.out -p a-dna_wat.prmtop -c a-dna_heat.rst -r a-dna_equal.rst -ref a-dna_heat.rst -x a-dna_equal.mdcrd
```

### c. Running MD simulations

Generate `a-dna_md.in` input file:
```
production
 &cntrl
    imin=0, ntx=7, 
    ntpr=500,ntwr=1000, 
    ntwx=1000,ntwe=1000,nscm=1000,
    ntf=2, ntc=2,ntb=2, ntp=1, tautp=5.0, taup=5.0,
    nstlim=1000000, t=0.0, dt=0.002,
    cut=10.0,ntt=1,irest=1,iwrap=1,ntxo=1, ig = -1,
    ioutfm=0,
 &end
 &ewald
    ew_type = 0, skinnb = 1.0,
 &end
```

Next, run the following commands on the terminal:
```
mpirun -np 12 $AMBERHOME/bin/sander.MPI -O -i a-dna_md.in -o md.out -p a-dna_wat.prmtop -c a-dna_equal.rst -r md.rst -x md.mdcrd -inf mdinfo
```

## 5. Post Processing

### Reimage the trajectory back to its original box:

Generate cpptraj file `reimage.ptraj` with:
```
trajin md.mdcrd
center :1-20
image familiar
trajout reimaged_dna_md.nc netcdf
```

Next, run this command on the terminal:
```
cpptraj a-dna_wat.prmtop reimage.ptraj
```

Note from the official website:
(Note: These tutorials are meant to provide illustrative examples of how to use the AMBER software suite to carry out simulations that can be run on a simple workstation in a reasonable period of time. They do not necessarily provide the optimal choice of parameters or methods for the particular application area.)
Copyright Ross Walker 2015
























