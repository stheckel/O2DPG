# O2DPG - Monte Carlo Simulation

Withing this directory structure are located the scripts and configuration to run Monte Carlo simulations of the ALICE experiment within the O2 project.

## Adding QC Tasks to the simulation script

Below are the steps to integrate a new QC Task to the main simulation, reco and QC workflow.

1. Build O2, QualityControl and O2DPG with with `o2` defaults:
```
aliBuild build O2 QualityControl O2DPG --defaults o2 -j <jobs>
```

2. Make sure that the setup works by loading the environment and running the example script.
It runs a series of tasks, which are usually DPL workflows that depend on each other and write/read processing results in form of ROOT files.
It will simulate 3 TimeFrames, reconstruct them and run any QC.
Corresponding files will be created in the current directory, QC objects will be also uploaded to QCDB. 
```
alienv enter O2/latest QualityControl/latest O2DPG/latest
cd MC/run/examples
./O2DPG_pp_minbias_multiple_tf_qc.sh
```
If the script does not succeed, contact the repository maintainers.
Sometimes an intermittent issue might appear, then it might be worth executing the script again - it will pick up from the latest failed task.

3. Prepare a QC config file of your Task.
Please make sure to put the following default parameters in the Activity section:
```
     "Activity": {
       ...
       "provenance": "qc_mc",
       "passName": "passMC",
       "periodName": "SimChallenge"
     },

```
With the future developments, they will be overwritten with production-specific values.
Also, since the processing time is not critical, one can avoid using data sampling in most cases and use "direct" data sources (see QC doc)

Put the file to MC/config/QC/json directory or make sure it is installed in the QC package.

4. In `o2dpg_sim_workflow.py`, find the big loop over simulated TimeFrames and the QC section within.
Add your QC following the example below.
See the explanation of particular lines for more information.
```
for tf in range(1, NTIMEFRAMES + 1):
  ...
  if includeFullQC or includeLocalQC:
    ...
    ### Primary vertex
    addQCPerTF(taskName='vertexQC',
               needs=[PVFINDERtask['name']], # defines which tasks should run before this QC workflow, so the relevant results are available
               readerCommand='o2-primary-vertex-reader-workflow', # defines what command should be used to read input files and put before o2-qc workflow
               configFilePath='json://${O2DPG_ROOT}/MC/config/QC/json/vertexing-qc-direct-mc.json') # path to the QC config file
```
The lines above will make the QC task ran for each simulated and reconstructed timeframe separately.
The intermediate results will be stored and merged in one file in the `QC` directory.

5. Open the `o2dpg_qc_finalization_workflow.py` script and find the `include_all_QC_finalization` function.

This part makes the later parts of QC (Checks, Aggregators, uploading to QCDB) run after all TimeFrames are processed.
Add your QC following the example below. Use the same task name and config file as in the previous point.
```
def include_all_QC_finalization(ntimeframes, standalone):
  ...
  add_QC_finalization('vertexQC', 'json://${O2DPG_ROOT}/MC/config/QC/json/vertexing-qc-direct-mc.json')

```

6. Delete the files generated by the workflow during step 2 and run the `O2DPG_pp_minbias_multiple_tf_qc.sh` script again.
Verify that the QC Task succeeds.
Log are available under task names in their working directories: tf<n> when processing TFs and QC during finalization.

7. Ask Catalin to add the file with QC results to the list of merged files on Grid productions. The file has the same name as `taskName`, but with the `.root` suffix. If you update the task name, also please let Catalin know.
