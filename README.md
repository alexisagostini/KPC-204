# KPC-204
## Construction
### Structures

#### dowload 2OV5 (KPC-2)
```bash
wget https://files.rcsb.org/download/2OV5.pdb
```
#### clean
``` Python
load 2OV5.pdb
remove resn HOH        # Remove H2O
remove resn SO4        # Remove sulfits
save KPC2_clean.pdb
```
#### introduces mutations KPC-204 with PyMOL (Wizard > Mutagenesis)
```python
load KPC2_clean.pdb
wizard mutagenesis
cmd.get_wizard().set_mode('NO_ROTAMERS')
cmd.get_wizard().do_select('104/')   # select the residu 104
cmd.get_wizard().set_aa('ARG')       # introduction of the mutation
cmd.get_wizard().apply()
save KPC204_clean.pdb
```
###  GROMACS
Gromacs is an opensource software of molecular dynamics
#### Preparation of the protein's topologie
```Bash
# KPC-2
gmx pdb2gmx -f KPC2_clean.pdb -o KPC2_processed.gro \
            -water tip3p -ff amber99sb-ildn

# KPC-204
gmx pdb2gmx -f KPC204_clean.pdb -o KPC204_processed.gro \
            -water tip3p -ff amber99sb-ildn
```
### Parameter of avibactam with ACPYPE
GROMACS is adapted to protein but not on avibactam, ACPYPE is a molecular tool that translate a molecule to a language that GROMACS understand 

```bash
wget "https://pubchem.ncbi.nlm.nih.gov/rest/pug/compound/CID/25151352/record/SDF/?record_type=3d&response_type=save&response_basename=avibactam" -O avibactam.sdf
acpype -i avibactam.sdf -c bcc -n -1
# ->avibactam_GMX.itp and avibactam_GMX.gro
```
GROMACS needs the 2 files.gro in one file 
```bash
cat KPC2_processed.gro avibactam_GMX.gro > KPC2_AVI_complex.gro
```
I also need to include his parameters
Add the presence of avibactam

```bash
nano KPC2_AVI_complex.gro
```
```markdown
#include "amber99sb-ildn.ff/forcefield.itp"
#include "amber99sb-ildn.ff/tip3p.itp"
#include "avibactam_GMX.itp"          ← Add parameters

[ system ]
KPC2 + Avibactam in water

[ molecules ]
Protein_chain_A    1
avibactam          1                  ← Add the presence
```
#### Creation of the environment 
I dive the protein in a realistic environment
##### the box
```bash
gmx editconf -f KPC2_AVI_complex.gro -o KPC2_box.gro \
             -c -d 1.0 -bt cubic
```
c = possition of the protein central in the box
d 1.0 = 1 nanometer between the protein and the box
bt cubic = forme of the box (cube)
##### the solvent
the media where the protein is, is water I add water
``` Bash
gmx solvate -cp KPC2_box.gro -cs spc216.gro \
            -o KPC2_solv.gro -p topol.top
```
cp = the protein to solve
cs spc216.gro = the most often use for proteine precharged (gain time) (density , diffusion coef and dielectric constante are really similare than true water )
-o output
-p topol.top update automaticaly the lvl of water

#### neutralisation with ions
Creation of ions
```bash
nano ions.mdp
```
```Markdown
; ions.mdp - fichier de paramètres pour l'ajout d'ions
integrator  = steep        # algorithm of diminution of gradiant
emtol       = 1000.0   
emstep      = 0.01
nsteps      = 0            # no stimulation (just for the compilation)

nstlist         = 1
cutoff-scheme   = Verlet
ns_type         = grid
coulombtype     = cutoff   # methode of calculation
rcoulomb        = 1.0      # 1nm max
rvdw            = 1.0
pbc             = xyz      # 3 axis
```

```bash
gmx grompp -f ions.mdp -c KPC2_solv.gro \
           -p topol.top -o ions.tpr

gmx genion -s ions.tpr -o KPC2_ions.gro \
           -p topol.top -pname NA -nname CL -neutral
```
## Start
### minimisation of the energie

file creation

```bash
nano minim.mdp
```
```markdown
; minim.mdp - Minimisation d'énergie
integrator  = steep         ; algorithme de descente de gradient
emtol       = 1000.0        ; critère d'arrêt (kJ/mol/nm)
emstep      = 0.01          ; taille du pas initial
nsteps      = 50000         ; maximum 50 000 pas

nstlist         = 10
cutoff-scheme   = Verlet
ns_type         = grid
coulombtype     = PME
rcoulomb        = 1.0
rvdw            = 1.0
pbc             = xyz
```
```Bash
gmx grompp -f minim.mdp -c KPC2_ions.gro \
           -p topol.top -o minim.tpr

gmx mdrun -v -deffnm minim
```
### NVT (atomes volume temperature)
```bash
nano nvt.mdp
```
```markdown$
; Paramètres de run
define                  = -DPOSRES      ; Restraints sur la protéine
integrator              = md            ; Algorithme leap-frog
nsteps                  = 50000         ; 2 fs × 50000 = 100 ps
dt                      = 0.002         ; Pas de temps en ps

; Sorties
nstxout                 = 500           ; Sauvegarder les coordonnées toutes les 1 ps
nstvout                 = 500           ; Sauvegarder les vitesses
nstenergy               = 500           ; Sauvegarder l'énergie
nstlog                  = 500           ; Écrire dans le log

; Liaisons
constraint_algorithm    = lincs
constraints             = h-bonds
lincs_iter              = 1
lincs_order             = 4

; Électrostatique
cutoff-scheme           = Verlet
ns_type                 = grid
nstlist                 = 10
rcoulomb                = 1.0
rvdw                    = 1.0
coulombtype             = PME
pme_order               = 4
fourierspacing          = 0.16

; Température (thermostat)
tcoupl                  = V-rescale
tc-grps                 = Protein Non-Protein
tau_t                   = 0.1     0.1
ref_t                   = 300     300

; Pression (désactivée en NVT)
pcoupl                  = no

; Vitesses initiales
gen_vel                 = yes
gen_temp                = 300
gen_seed                = -1
```
```bash
# 1. Compiler avec grompp
gmx grompp -f nvt.mdp -c em.gro -r em.gro -p topol.top -n index.ndx -o nvt.tpr

# 2. Lancer la simulation
gmx mdrun -v -deffnm nvt
```
### NPT (atomes pression temperature)
```bash
nano npt.mdp
```
```markdown
; npt.mdp - Équilibration pression + température
integrator          = md
nsteps              = 50000        ; 100 ps
dt                  = 0.002

; Température
tcoupl              = V-rescale
tc-grps             = Protein Non-Protein
tau_t               = 0.1  0.1
ref_t               = 300  300

; Pression 
pcoupl              = Parrinello-Rahman
pcoupltype          = isotropic
tau_p               = 2.0
ref_p               = 1.0          ; 1 bar
compressibility     = 4.5e-5       ; compressibilité de l'eau

; Restraints sur la protéine
define              = -DPOSRES

; Électrostatique
coulombtype         = PME
rcoulomb            = 1.0
rvdw                = 1.0
pbc                 = xyz
```
```bash
gmx grompp -f nvt.mdp -c minim.gro \
           -r minim.gro -p topol.top -o nvt.tpr

gmx mdrun -deffnm nvt
```
#### production MD 
```bash
nano md.mdp
```
```markdown
; === PRODUCTION MD — 20 ns ===

; Run
integrator              = md
nsteps                  = 10000000      ; 2 fs × 10 000 000 = 20 ns ✅
dt                      = 0.002

; Sorties
nstxout                 = 0             ; Pas de .trr (lourd)
nstvout                 = 0
nstfout                 = 0
nstenergy               = 5000          ; Énergie toutes les 10 ps
nstlog                  = 5000          ; Log toutes les 10 ps
nstxout-compressed      = 5000          ; Trajectoire .xtc toutes les 10 ps
compressed-x-precision  = 1000

; Liaisons
constraint_algorithm    = lincs
constraints             = h-bonds
lincs_iter              = 1
lincs_order             = 4

; Électrostatique
cutoff-scheme           = Verlet
ns_type                 = grid
nstlist                 = 10
rcoulomb                = 1.0
rvdw                    = 1.0
coulombtype             = PME
pme_order               = 4
fourierspacing          = 0.16

; Température
tcoupl                  = V-rescale
tc-grps                 = Protein Non-Protein
tau_t                   = 0.1     0.1
ref_t                   = 300     300

; Pression (NPT — barostat actif)
pcoupl                  = Parrinello-Rahman
pcoupltype              = isotropic
tau_p                   = 2.0
ref_p                   = 1.0
compressibility         = 4.5e-5

; Vitesses — PAS de génération (on repart du NPT)
gen_vel                 = no
continuation            = yes
```
```bash
gmx grompp -f md.mdp -c npt.gro \
           -p topol.top -o md.tpr

gmx mdrun -deffnm md -ntmpi 1 -ntomp 8 -gpu_id 0
```
## Analyse
### RMSD global
``` bash
gmx rms -s md.tpr -f md.xtc -o rmsd.xvg -tu ns
# Sélectionner : 4 (Backbone) pour référence et analyse
```
### RMSF Residu
### Distance avibactam <-> Ser70
### curves
