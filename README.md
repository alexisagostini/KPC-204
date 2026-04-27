Workflow
1. Structures
KPC-2 — crystal structure

Copier
wget https://files.rcsb.org/download/2OV5.pdb
Copier
# PyMOL
load 2OV5.pdb
remove resn HOH+SO4
save structures/KPC2_clean.pdb
KPC-204 — clinical variant

Copier
wget -O structures/KPC204.fasta \
  "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=protein&id=WXU16489.1&rettype=fasta&retmode=text"
Structure prediction → ColabFold
Paste KPC204.fasta · set template_mode = pdb100 · use_amber = True
Download best model → structures/KPC204_raw.pdb

Copier
# PyMOL
load structures/KPC204_raw.pdb
remove resn HOH
save structures/KPC204_clean.pdb
2. Topology
Copier
gmx pdb2gmx -f structures/KPC2_clean.pdb   -o KPC2_processed.gro   -water tip3p -ff amber99sb-ildn
gmx pdb2gmx -f structures/KPC204_clean.pdb -o KPC204_processed.gro -water tip3p -ff amber99sb-ildn
3. Avibactam
Copier
wget "https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/CID/25151352/record/SDF/?record_type=3d" \
     -O avibactam/avibactam.sdf

acpype -i avibactam/avibactam.sdf -c bcc -n -1
Copier
grep "moleculetype" -A2 avibactam/avibactam_GMX.itp
Copier
python3 scripts/merge_gro.py KPC2_processed.gro avibactam/avibactam_GMX.gro KPC2_AVI_complex.gro
Ajouter dans topol.top :

Copier
#include "avibactam/avibactam_GMX.itp"

[ molecules ]
Protein_chain_A   1
MOL               1
4. Solvation
Copier
gmx editconf -f KPC2_AVI_complex.gro -o KPC2_box.gro -c -d 1.0 -bt cubic
gmx solvate  -cp KPC2_box.gro -cs spc216.gro -o KPC2_solv.gro -p topol.top
gmx grompp   -f mdp/ions.mdp -c KPC2_solv.gro -p topol.top -o ions.tpr
gmx genion   -s ions.tpr -o KPC2_ions.gro -p topol.top -pname NA -nname CL -neutral
5. Minimization
Copier
gmx grompp -f mdp/minim.mdp -c KPC2_ions.gro -p topol.top -o minim.tpr
gmx mdrun  -v -deffnm minim
6. Equilibration
NVT — 100 ps

Copier
gmx grompp -f mdp/nvt.mdp -c minim.gro -r minim.gro -p topol.top -n index.ndx -o nvt.tpr
gmx mdrun  -v -deffnm nvt
NPT — 100 ps

Copier
gmx grompp -f mdp/npt.mdp -c nvt.gro -r nvt.gro -p topol.top -n index.ndx -o npt.tpr
gmx mdrun  -v -deffnm npt
7. Production MD — 20 ns
Copier
gmx grompp -f mdp/md.mdp -c npt.gro -p topol.top -n index.ndx -o md.tpr
gmx mdrun  -deffnm md -ntmpi 1 -ntomp 8 -gpu_id 0
8. Analysis
Copier
gmx rms -s md.tpr -f md.xtc -o analysis/rmsd.xvg -tu ns

gmx rmsf -s md.tpr -f md.xtc -o analysis/rmsf.xvg -res

gmx distance -s md.tpr -f md.xtc \
             -select 'resname MOL and name O1; resid 70 and name OG' \
             -oall analysis/distance_ser70.xvg

xmgrace analysis/rmsd.xvg
xmgrace analysis/rmsf.xvg
xmgrace analysis/distance_ser70.xvg
References
Galdadas et al. (2018) Defining the architecture of KPC-2 Carbapenemase — Scientific Reports
KPC-204 sequence — NCBI WXU16489.1
Structure prediction — ColabFold
GROMACS — Manual 2023
