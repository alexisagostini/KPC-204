1. Structures
KPC-2 (WT)
Copier
wget https://files.rcsb.org/download/2OV5.pdb
Copier
# PyMOL
load 2OV5.pdb
remove resn HOH+SO4
save KPC2_clean.pdb
KPC-204 (clinical variant — WXU16489.1)
Copier
# Download sequence
wget -O KPC204.fasta \
  "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=protein&id=WXU16489.1&rettype=fasta&retmode=text"
Copier
# AlphaFold2 structure prediction
→ https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/AlphaFold2.ipynb
  query_sequence : KPC204.fasta
  template_mode  : pdb100
  use_amber      : True
→ Download best model → KPC204_raw.pdb
Copier
# PyMOL
load KPC204_raw.pdb
remove resn HOH
save KPC204_clean.pdb
2. Topology
Copier
gmx pdb2gmx -f KPC2_clean.pdb   -o KPC2_processed.gro   -water tip3p -ff amber99sb-ildn
gmx pdb2gmx -f KPC204_clean.pdb -o KPC204_processed.gro -water tip3p -ff amber99sb-ildn
3. Avibactam
Copier
# Download
wget "https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/CID/25151352/record/SDF/?record_type=3d&response_type=save&response_basename=avibactam" -O avibactam.sdf

# Parameterize
acpype -i avibactam.sdf -c bcc -n -1
# → avibactam_GMX.itp / avibactam_GMX.gro
Copier
# Check molecule name
grep "moleculetype" -A2 avibactam_GMX.itp
Copier
# Merge coordinates
python3 merge_gro.py KPC2_processed.gro avibactam_GMX.gro KPC2_AVI_complex.gro
<details> <summary>merge_gro.py</summary>
Copier
import sys

def merge_gro(prot, lig, out):
    p = open(prot).readlines()
    l = open(lig).readlines()
    atoms_p = p[2:-1]
    atoms_l = l[2:-1]
    total = len(atoms_p) + len(atoms_l)
    with open(out, 'w') as f:
        f.write(p[0])
        f.write(f"{total}\n")
        f.writelines(atoms_p)
        f.writelines(atoms_l)
        f.write(p[-1])

merge_gro(sys.argv[1], sys.argv[2], sys.argv[3])
</details>
Copier
# Edit topol.top — add after forcefield includes :
#include "avibactam_GMX.itp"

# Edit [ molecules ] section :
Protein_chain_A   1
MOL               1        ; match name from moleculetype
4. Solvation & Neutralization
Copier
gmx editconf -f KPC2_AVI_complex.gro -o KPC2_box.gro  -c -d 1.0 -bt cubic
gmx solvate  -cp KPC2_box.gro -cs spc216.gro           -o KPC2_solv.gro -p topol.top
Copier
gmx grompp -f ions.mdp -c KPC2_solv.gro -p topol.top -o ions.tpr
gmx genion  -s ions.tpr -o KPC2_ions.gro -p topol.top -pname NA -nname CL -neutral
<details> <summary>ions.mdp</summary>
Copier
integrator      = steep
nsteps          = 0
cutoff-scheme   = Verlet
coulombtype     = PME
rcoulomb        = 1.0
rvdw            = 1.0
pbc             = xyz
</details>
5. Energy Minimization
Copier
gmx grompp -f minim.mdp -c KPC2_ions.gro -p topol.top -o minim.tpr
gmx mdrun  -v -deffnm minim
<details> <summary>minim.mdp</summary>
Copier
integrator  = steep
emtol       = 1000.0
emstep      = 0.01
nsteps      = 50000
cutoff-scheme = Verlet
coulombtype = PME
rcoulomb    = 1.0
rvdw        = 1.0
pbc         = xyz
</details>
6. Equilibration
NVT — 100 ps
Copier
gmx grompp -f nvt.mdp -c minim.gro -r minim.gro -p topol.top -n index.ndx -o nvt.tpr
gmx mdrun  -v -deffnm nvt
<details> <summary>nvt.mdp</summary>
Copier
define          = -DPOSRES
integrator      = md
nsteps          = 50000
dt              = 0.002
cutoff-scheme   = Verlet
coulombtype     = PME
rcoulomb        = 1.0
rvdw            = 1.0
tcoupl          = V-rescale
tc-grps         = Protein Non-Protein
tau_t           = 0.1   0.1
ref_t           = 300   300
pcoupl          = no
gen_vel         = yes
gen_temp        = 300
gen_seed        = -1
constraints     = h-bonds
</details>
NPT — 100 ps
Copier
gmx grompp -f npt.mdp -c nvt.gro -r nvt.gro -p topol.top -n index.ndx -o npt.tpr
gmx mdrun  -v -deffnm npt
<details> <summary>npt.mdp</summary>
Copier
define          = -DPOSRES
integrator      = md
nsteps          = 50000
dt              = 0.002
cutoff-scheme   = Verlet
coulombtype     = PME
rcoulomb        = 1.0
rvdw            = 1.0
tcoupl          = V-rescale
tc-grps         = Protein Non-Protein
tau_t           = 0.1   0.1
ref_t           = 300   300
pcoupl          = Parrinello-Rahman
pcoupltype      = isotropic
tau_p           = 2.0
ref_p           = 1.0
compressibility = 4.5e-5
gen_vel         = no
continuation    = yes
constraints     = h-bonds
</details>
7. Production MD — 20 ns
Copier
gmx grompp -f md.mdp -c npt.gro -p topol.top -n index.ndx -o md.tpr
gmx mdrun  -deffnm md -ntmpi 1 -ntomp 8 -gpu_id 0
<details> <summary>md.mdp</summary>
Copier
integrator              = md
nsteps                  = 10000000     ; 20 ns
dt                      = 0.002
nstxout-compressed      = 5000         ; .xtc every 10 ps
nstenergy               = 5000
nstlog                  = 5000
cutoff-scheme           = Verlet
coulombtype             = PME
rcoulomb                = 1.0
rvdw                    = 1.0
tcoupl                  = V-rescale
tc-grps                 = Protein Non-Protein
tau_t                   = 0.1   0.1
ref_t                   = 300   300
pcoupl                  = Parrinello-Rahman
pcoupltype              = isotropic
tau_p                   = 2.0
ref_p                   = 1.0
compressibility         = 4.5e-5
gen_vel                 = no
continuation            = yes
constraints             = h-bonds
</details>
8. Analysis
Copier
# Global stability
gmx rms  -s md.tpr -f md.xtc -o rmsd.xvg -tu ns
# → select Backbone (4) for both reference and fit

# Per-residue flexibility
gmx rmsf -s md.tpr -f md.xtc -o rmsf.xvg -res
# → focus residues 210-220 (Ω-loop)

# Avibactam ↔ Ser70 distance
gmx distance -s md.tpr -f md.xtc \
             -select 'resname MOL and name O1; resid 70 and name OG' \
             -oall distance_ser70.xvg

# Plot
xmgrace rmsd.xvg
xmgrace rmsf.xvg
xmgrace distance_ser70.xvg
File Tree
Copier
KPC204_MD/
├── structures/
│   ├── KPC2_clean.pdb
│   └── KPC204_clean.pdb
├── avibactam/
│   ├── avibactam_GMX.itp
│   └── avibactam_GMX.gro
├── mdp/
│   ├── ions.mdp
│   ├── minim.mdp
│   ├── nvt.mdp
│   ├── npt.mdp
│   └── md.mdp
├── scripts/
│   └── merge_gro.py
└── analysis/
    ├── rmsd.xvg
    ├── rmsf.xvg
    └── distance_ser70.xvg
Reference
Galdadas et al., Scientific Reports 2018 — KPC-2 allosteric networks
KPC-204 sequence : NCBI WXU16489.1
Structure prediction : ColabFold
