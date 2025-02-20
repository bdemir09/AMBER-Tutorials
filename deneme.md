#DNA Molecular Dynamics Simulation Using AMBER

This procedure is adapted from the AMBER tutorial B1 for DNA molecular dynamics simulations. It outlines the steps necessary to generate, minimize, equilibrate, and run molecular dynamics simulations for a DNA molecule using AMBER.

Prerequisites

Before starting, set up your environment by running the following commands in your terminal:

export AMBERHOME=/path/to/amber16
export PATH=$PATH:$AMBERHOME/bin
source $AMBERHOME/amber.sh

1. Creating the DNA Structure

Use the NAB program in AMBER to generate the DNA structure.

a. Create the a-dna.nab text file:

molecule m;
m = fd_helix( "adna", "cccgcgccc", "dna" );
putpdb( "a-dna-9base.pdb", m, "-wwpdb");

Run the following commands:

$AMBERHOME/bin/nab a-dna.nab
./a.out

2. Creating Input Files

Run the following command:

$AMBERHOME/bin/tleap -s -f $AMBERHOME/dat/leap/cmd/leaprc.DNA.bsc1

In the interactive prompt, enter:

> source leaprc.water.tip3p
> adna = loadpdb a-dna-9base.pdb
> addions adna Na+ 0
> saveamberparm adna a-dna_charges_only.prmtop a-dna_charges_only.inpcrd
> solvateoct adna TIP3PBOX 8.0
> addionsrand Na+ 15 Cl- 15
> saveamberparm adna a-dna_wat.prmtop a-dna_wat.inpcrd
> quit

3. Minimization Steps

a. Minimizing Only the Solution

Create a-dna_min1.in with:

&cntrl
 imin=1, ntx=1, irest=0, ntpr=1, ntf=1, ntxo=1,
 ntb=1, cut=10.0, nsnb=10, ntr=1,
 maxcyc=1000, ncyc=500, ntmin=1,
 restraintmask=':1-34', restraint_wt=25.0,
&end

Run:

$AMBERHOME/bin/sander -O -i a-dna_min1.in -o a-dna_min1.out -p a-dna_wat.prmtop -c a-dna_wat.inpcrd -r a-dna_min1.rst -ref a-dna_wat.inpcrd

b. Minimizing Solution and Counterions

Create a-dna_min2.in and run:

mpirun -np 12 sander.MPI -O -i a-dna_min2.in -o a-dna_min2.out -p a-dna_wat.prmtop -c a-dna_min1.rst -r a-dna_min2.rst -ref a-dna_min1.rst

c. Minimizing the Entire System

Create a-dna_min3.in and run:

mpirun -np 12 sander.MPI -O -i a-dna_min3.in -o a-dna_min3.out -p a-dna_wat.prmtop -c a-dna_min2.rst -r a-dna_min3.rst -ref a-dna_min2.rst

4. Equilibration and Molecular Dynamics

a. Heating the System

Create heat.in and run:

mpirun -np 12 $AMBERHOME/bin/sander.MPI -O -i heat.in -o a-dna_heat.out -p a-dna_wat.prmtop -c a-dna_min3.rst -r a-dna_heat.rst -ref a-dna_min3.rst -x a-dna_heat.mdcrd

b. Equilibrating the System

Create equal.in and run:

mpirun -np 12 $AMBERHOME/bin/sander.MPI -O -i equal.in -o a-dna_equal.out -p a-dna_wat.prmtop -c a-dna_heat.rst -r a-dna_equal.rst -ref a-dna_heat.rst -x a-dna_equal.mdcrd

c. Running Molecular Dynamics

Create a-dna_md.in and run:

mpirun -np 12 $AMBERHOME/bin/sander.MPI -O -i a-dna_md.in -o md.out -p a-dna_wat.prmtop -c a-dna_equal.rst -r md.rst -x md.mdcrd -inf mdinfo

5. Post-processing

a. Reimage Trajectory

Create reimage.ptraj:

trajin md.mdcrd
center :1-20
image familiar
trajout reimaged_dna_md.nc netcdf

Run:

cpptraj a-dna_wat.prmtop reimage.ptraj

b. Remove Water Molecules

Create process1.cpptraj:

parm dna.prmtop
trajin md.nc
autoimage
strip :WAT
trajout md-nowater.nc netcdf

Run:

cpptraj -i process1.cpptraj

This procedure enables users to simulate DNA dynamics effectively using AMBER.

