# Usage

Simulation is carried out in vacuum. During the simulation harmonic potential is applied on distances between COMs of each polymer and COM of membrane and it is only applied when this distance is more than *R* nm. Position restraints are applied on lipids to keep membrane integral.

### Generate topology
```
gmx pdb2gmx -f lipodisk.pdb -o lipodisk_processed.gro -ff oplsaa_lipids_polymers -water spce -p topol.top
```
### Move posres and itp to their own folders and define where to find them
```
for name in `ls topol_*`
do
sed -i -- 's/\#include \"posre_/\#include \"..\/posres\/posre_/g' $name
done
sed -i -- 's/\#include \"topol_/\#include \"itp\/topol_/g' topol.top
```
### Define position restraints that will be applied during NPT simulation to keep membrane integral
```
echo "#ifdef POSRES_LIPIDS" >> topol_Other*.itp
var=`ls posre_Other*.itp`
echo "#include \"../posres/$var\"" >> topol_Other*.itp
echo "#endif" >> topol_Other*.itp

mv posre*.itp posres/.
mv topol*.itp itp/.
```
### Set box size
```
gmx editconf -f SMALP_processed.gro -o SMALP_newbox.gro -d 0.1 -c -bt triclinic
```
### Prepare and run EM
```
gmx grompp -f config/minim.mdp -c lipodisk_newbox.gro -r lipodisk_newbox.gro -o lipodisk_em.tpr -p topol.top -po mdout.mdp -maxwarn 1
gmx mdrun -deffnm lipodisk_em
```
### Prepare and run short NVT equilibration
```
gmx grompp -f config/nvt.mdp -c lipodisk_em.gro -r lipodisk_em.gro -o lipodisk_nvt.tpr -p topol.top -po mdout.mdp -maxwarn 1
gmx mdrun -deffnm lipodisk_nvt
```
### Prepare index file with groups for pulling and mdp file
```
gmx make_ndx -f SMALP_nvt.tpr<<!
q
!
python gen_index.py -i lipodisk.pdb -o index_more.ndx
cat index_more.ndx >> index.ndx
python gen_mdp.py -m config/lipodisk.mdp -i index.ndx -o config/lipodisk_flatbot.mdp
```

### Prepare and run NPT
```
gmx editconf -f lipodisk_nvt.gro -o lipodisk_bigbox.gro -box 30 30 8 -c -bt triclinic
gmx grompp -f config/lipodisk_flatbot.mdp -c lipodisk_bigbox.gro -r lipodisk_bigbox.gro -n index.ndx -o lipodisk_flatbot.tpr -p topol.top -po mdout.mdp -maxwarn 1
gmx mdrun -deffnm lipodisk_flatbot
```
### If any error occured during simulation
To derive frame at *Nth* step (N ps) use *trjconv*:
```
gmx trjconv -f lipodisk_flatbot.trr -o lipodisk_1.pdb -s lipodisk_flatbot.tpr -dump N<<!
0
!
```
Then it is needed to add TER in new *.pdb* file:
```
python add_ter_between_chains.py -i lipodisk_1.pdb -o lipodisk_ter.pdb
```