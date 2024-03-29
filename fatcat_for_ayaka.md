# 2018-09-20 16:00:14

Ayaka asked to run FATCAT in a few IDs for for project. This is how we did it
for DLPFC:

```bash
# caterpie
bin=/usr/local/neuro/TORTOISE_V3.1.2/DIFFPREPV312/bin/bin/ConvertOldListfileToNew;
for m in `cat ~/tmp/no_fatcat.txt`; do     
    echo "Working on $m";
    mydir=/mnt/shaw/data_by_maskID/${m};
    mkdir ${mydir}/tortoise_converted_v2;
    if [ -e ${mydir}/edti_proc/edti_DMC_DR_R1.list ]; then
        ${bin} ${mydir}/edti_proc/edti_DMC_DR_R1.list ${mydir}/edti_proc/edti_DMCstructural.nii ${mydir}/tortoise_converted_v2/${m}.list;
    else
        ${bin} ${mydir}/edti_proc/edti_DMC_R1.list ${mydir}/edti_proc/edti_DMCstructural.nii ${mydir}/tortoise_converted_v2/${m}.list;
    fi;
    cp ${mydir}/edti_proc/edti_DMCtemplate.nii ${mydir}/edti_proc/edti_DMCstructural.nii ${mydir}/tortoise_converted_v2/;
done
```

# 2018-09-21 10:15:00

Some of the IDs Ayaka had sent didn't have good DTI (i.e. were bad according to
Labmatrix and didn't have the necessary directories ot be converted).

# 2018-09-24 09:44:12

```bash
# caterpie
for m in `cat ~/tmp/no_fatcat_ok.txt `; do
    echo $m;
    ssh -q helix.nih.gov "mkdir /scratch/sudregp/tortoise_exported_v2/${m}";
    scp -qr /mnt/shaw/data_by_maskID/${m}/tortoise_converted_v2 helix:/scratch/sudregp/tortoise_exported_v2/${m}/exported;
done
```

Then, we copy the export directory of all subjects to biowulf, and run fatcat:

```bash
while read m; do
    echo "export SUBJECTS_DIR=/data/NCR_SBRB/freesurfer5.3_subjects/; bash ~/research_code/dti/run_fatcat_fs_exported.sh /scratch/sudregp/tortoise_exported_v2/ ${m}" >> swarm.fatcat;
done < ~/tmp/no_fatcat_ok.txt
swarm -f swarm.fatcat -t 8 -g 40 --time=24:00:00 --merge-output --logdir trash_bin --job-name fc -m afni,TORTOISE
```

Some of the IDs died because of badly formed BMTXT matrix... not sure if it's
worth fixing them, though.

# 2018-10-01 14:35:08

At this point we have everyone that was run without problems ready. There are 322 scans. So, now it's time to parse those results:

```bash
for m in `cat ~/tmp/done.txt`; do
    if [ -e /mnt/shaw/data_by_maskID/${m}/fatcat_tortoise201_exported/o.pr00_000.grid ]; then
        cp /mnt/shaw/data_by_maskID/${m}/fatcat_tortoise201_exported/o.pr00_000.grid ~/ayaka/${m}.pr00_000.grid;
    fi;
done
mkdir ~/ayaka/parsed
cd ~/ayaka
for f in `/bin/ls -1 *grid`; do
     s=`echo $f | cut -d"." -f 1`;
     echo $s;
     csplit --quiet --prefix=${s}. ${s}.pr00_000.grid /^#/ {*};
     mv ${s}.?? parsed/;
done
```

And we have to run dti/collect_fatcat_grids.R to do the heavy lifting.

# 2018-10-11 12:58:29

The format of the output wasn't exactly what Ayaka needed. She needs a square
matrix per subject, per datatype. So, I created
dti/collect_fatcat_grids_matrixOutput.R to do that.