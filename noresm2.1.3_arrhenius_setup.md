# NorESM2.1.3 on arrhenius – setup recap

*Date:* 2026-10-12
*Status:* Model builds successfully (test case test_f19). Short test run not yet confirmed.

## Key paths

| What | Path / value |
|---|---|
| Source code | /nobackup/proj/disk/luftlab/personal/$USER/NorESM2.1.3 |
| Cases | /nobackup/proj/disk/luftlab/personal/$USER/cases/ |
| Model output | /nobackup/proj/disk/luftlab/personal/$USER/noresm_out/ |
| Shared configs | /nobackup/proj/disk/cesm_input-data/shared/config/cesm2.2.2/ |
| My backups | ~/noresm_arrhenius_config/ (fixed arrhenius config files) |
| Slurm account | naiss2026-3-656-cpu (NAISS 2026/3-656) |

> *Requirement:* membership in the EC-Earth input-data project (netCDF for Intel 2023 is installed there).

---

## 1. Shell environment

Run in every new shell, before building or creating cases:

```
module load buildenv-intel/2023a-eb
export MODULEPATH=/nobackup/proj/disk/ecearth3_input-data/apps/EasyBuild/el9_epyc9005/modules/all/Compiler/GCCcore/12.3.0:/nobackup/proj/disk/ecearth3_input-data/apps/EasyBuild/el9_epyc9005/modules/all/MPI/intel/2023.1.0/impi/2021.9.0:$MODULEPATH
module load netCDF-Fortran/4.6.1 netCDF/4.9.2 netCDF-C++4/4.3.1
```

If a newer Python module is loaded (check module list), unload it.

## 2. Conda environment (one-time)

The yml pins bioconda builds of Perl packages. With conda's default channel_priority=strict these are excluded ("excluded by strict repo priority"), so the solve fails. Fix: relax channel priority for this command only:

```
CONDA_CHANNEL_PRIORITY=flexible conda env create -n noresm_env \
  -f /nobackup/proj/disk/cesm_input-data/shared/config/cesm2.2.2/environment.yaml
```

Provides Python 3.11 plus the Perl libraries CIME needs.


## 2.1 Every time: 

```
conda activate noresm_env
```

## 3. Get the code at the 2.1.3 release (one-time)

```
cd /nobackup/proj/disk/luftlab/personal/$USER
git clone https://github.com/NorESMhub/NorESM.git NorESM2.1.3
cd NorESM2.1.3
git checkout release-noresm2.1.3          # cloning alone gives master, NOT 2.1.3
./manage_externals/checkout_externals
./manage_externals/checkout_externals -S  # verify: no 'M'; 'e-o' (optional) entries are fine
```

Resulting versions:

| Component | Version |
|---|---|
| cime | cime5.6.10_NorESM2_3_r5 |
| cam | cam_noresm2_1_v1.0.2 |
| clm | release-clm5.0.14-Nor_v1.0.5 |
| blom | v1.5.2 |

> *Note:* checkout_externals refuses to run if cime/cam have local modifications. If you ever need to re-run it: back up the modified files (listed in [step 6](#6-files-modified-in-the-source-tree)), git restore them, run checkout_externals, then copy the backups back in.

## 4. Machine config files (cime)

Copy the cesm_input-data arrhenius machine files into cime:

```
cd /nobackup/proj/disk/luftlab/personal/$USER/NorESM2.1.3
cp /nobackup/proj/disk/cesm_input-data/shared/config/cesm2.2.2/config_{batch,compilers,machines}.xml \
   cime/config/cesm/machines/
```

Two fixes are needed in config_compilers.xml for NorESM's cime5.6.10.

### (a) Schema fix: lowercase compile_threaded

compile_threaded="TRUE"/"FALSE" must be lowercase.

Symptom: case.setup fails with 'TRUE' is not a valid value of the atomic type 'xs:boolean'.

```
CIMEMACH=/nobackup/proj/disk/luftlab/personal/$USER/NorESM2.1.3/cime/config/cesm/machines
sed -i -E 's/compile_threaded=(["'"'"'])TRUE\1/compile_threaded=\1true\1/g; s/compile_threaded=(["'"'"'])FALSE\1/compile_threaded=\1false\1/g' $CIMEMACH/config_compilers.xml
```

### (b) BLOM needs -r8

BLOM needs -r8 (default real = 8 bytes). CESM has no BLOM, so the flag is missing.

Symptom: the blom build fails in phy/mod_checksum.F90 with
error #6633: The type of the actual argument differs from the type of the dummy argument
(xcaget declares plain real, callers pass real(r8)).

Add inside the <FFLAGS> block of <compiler MACH="arrhenius" COMPILER="intel">:

xml
<append MODEL="blom"> -r8 </append>


In my file this was inserted before line 147:

```
sed -i '147i\    <append MODEL="blom"> -r8 </append>' $CIMEMACH/config_compilers.xml
```

### Validate

Check all three files against the cime schemas:

```
XSD=/nobackup/proj/disk/luftlab/personal/$USER/NorESM2.1.3/cime/config/xml_schemas
xmllint --noout --schema $XSD/config_compilers_v2.xsd $CIMEMACH/config_compilers.xml
xmllint --noout --schema $XSD/config_machines.xsd     $CIMEMACH/config_machines.xml
xmllint --noout --schema $XSD/config_batch.xsd        $CIMEMACH/config_batch.xml
```

Fixed versions are backed up in ~/noresm_arrhenius_config/.

## 5. PE layout for arrhenius (config_pes.xml)

Anna's config_pes.xml adds one self-contained block (37 lines):
<grid name="a%1.9x2.5"> → <mach name="arrhenius"> … ntasks -1 (one full node), nthrds 1, rootpe 0.

It must go in *both* files (cam's is used for F/AMIP compsets, the top-level one for coupled):

- cime_config/config_pes.xml
- components/cam/cime_config/config_pes.xml

*Do not* overwrite the 2.1.3 files with Anna's full file – insert only the block:

```
sed -n '854,890p' ~/noresm_arrhenius_config/config_pes_toplevel.xml > ~/noresm_arrhenius_config/arrhenius_pes_block.xml
for f in cime_config/config_pes.xml components/cam/cime_config/config_pes.xml; do
  sed -i '/<config_pes/r '"$HOME"'/noresm_arrhenius_config/arrhenius_pes_block.xml' "$f"
done
xmllint --noout cime_config/config_pes.xml components/cam/cime_config/config_pes.xml
```

> *Limitation:* only covers f19 atmosphere grids (a%1.9x2.5, e.g. f19_tn14). Other resolutions (f09 etc.) need their own arrhenius block. Multi-node runs reportedly don't work yet (per Anna), so -1 = one node is intentional.

## 6. Files modified in the source tree

| File | Change |
|---|---|
| cime/config/cesm/machines/config_machines.xml | replaced (cesm_input) |
| cime/config/cesm/machines/config_batch.xml | replaced (cesm_input) |
| cime/config/cesm/machines/config_compilers.xml | replaced (cesm_input) + TRUE→true + blom -r8 |
| cime_config/config_pes.xml | + arrhenius block |
| components/cam/cime_config/config_pes.xml | + arrhenius block |

Backups in ~/noresm_arrhenius_config/:

- config_batch.xml, config_compilers.xml, config_machines.xml
- config_pes.xml / config_pes_toplevel.xml (Anna's full files, identical)
- arrhenius_pes_block.xml (just the block)

> Re-copy the fixed config_compilers.xml here after any edit.

## 7. Create, set up and build a case

```
cd /nobackup/proj/disk/luftlab/personal/$USER/NorESM2.1.3/cime/scripts
./create_newcase --case /nobackup/proj/disk/luftlab/personal/$USER/cases/test_f19 \
  --mach arrhenius --res f19_tn14 --compset NSSP370frc2 \
  --output-root /nobackup/proj/disk/luftlab/personal/$USER/noresm_out --run-unsupported

cd /nobackup/proj/disk/luftlab/personal/$USER/cases/test_f19
./xmlchange PROJECT=naiss2026-3-656-cpu     # default from Hamish's file is his project
./case.setup
./case.build
```

Notes:

- Machine name is arrhenius.
- case.setup / case.build are run from the *case* directory, not cime/scripts.
- CHARGE_ACCOUNT still defaults to support_staff.
- After changing config_compilers.xml, case.setup --reset does *not* regenerate the compiler macros in this cime version. Do:

  ```
  rm -f Macros.make Macros.cmake
  ./case.setup --reset
  grep -n r8 Macros.make           # verify the flag made it in
  ./case.build --clean ocn         # (or whichever component is affected)
  ./case.build
```  

## 8. Test run (next step – not yet done)

```
./xmlchange STOP_OPTION=ndays,STOP_N=5
./xmlchange JOB_WALLCLOCK_TIME=01:00:00 --subgroup case.run
./preview_run
./case.submit
squeue -u $USER ; tail -f CaseStatus     # expect "case.run success"
```

Logs: /nobackup/proj/disk/luftlab/personal/$USER/noresm_out/test_f19/run/ (check cesm.log.* first).

---

## Troubleshooting summary

| Symptom | Cause / fix |
|---|---|
| conda: excluded by strict repo priority (perl-*) | CONDA_CHANNEL_PRIORITY=flexible ([step 2](#2-conda-environment-one-time)) |
| checkout_externals: externals "not in a clean state" | Local edits in cime/cam; back up, git restore, re-run |
| git checkout tag: "local changes would be overwritten" | Top-level cime_config/config_pes.xml modified; back up + git restore first |
| case.setup: 'TRUE' not valid xs:boolean | Lowercase compile_threaded ([step 4a](#a-schema-fix-lowercase-compile_threaded)) |
| blom: error #6633 in mod_checksum.F90 | Missing -r8 for blom ([step 4b](#b-blom-needs--r8)) + regenerate Macros |
| case.build: "must invoke case.setup" | case.setup failed earlier – fix that first |
