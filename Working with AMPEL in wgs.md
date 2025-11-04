# Working with AMPEL in wgs

[TOC]

## Easier ssh with VS code:

## Installing AMPEL locally
1. Make sure you have poetry installed. If not, there are instructions for this here: https://python-poetry.org/docs/ 


1. Create a new conda environment with python 3.12, then activate it.
`conda create --name ampelenv python=3.12`
`conda activate ampelenv`

1. In this environment, run each of these lines:
`git clone https://github.com/AmpelProject/Ampel-HU-astro.git`
`cd Ampel-HU-astro/`
`poetry install -E "ztf sncosmo extcats notebook"`
`poetry run ampel config build -out ampel_conf.yaml >& ampel_conf.log`
If the installation stops due to missing package, you can continue installation without the package by setting -stop-on-errors=0
`poetry run ampel config build -out ampel_conf.yaml >& ampel_conf.log -stop-on-errors=0`

1. Check the install by looking at the `ampel_conf.log` file in Ampel-HU-astro
1.  It should list all the AMPEL componenets, missing packages (most likely you can work without these packages ) and end with something like `Total time required:7 minutes 27 seconds`. 

A sample `ampel_conf.log` file :
```
2025-09-19 18:17:05 ConfigCommand:223 INFO
 Building config [use -verbose for more details]

2025-09-19 18:17:05 DistConfigBuilder:45 SHOUT
 Detected ampel components: '['ampel-core', 'ampel-interface', 'ampel-alerts', 'ampel-photometry', 'ampel-ztf', 'ampel-plot', 'ampel-hu-astro']'

2025-09-19 18:24:31 ConfigBuilder:409 INFO
 Erroneous units (import failed):
 ampel.ops.AmpelExceptionPublisher - [ModuleNotFoundError] No module named 'slack_sdk'
 ampel.ztf.t2.T2LightCurveFeatures - [ModuleNotFoundError] No module named 'light_curve'
 ampel.contrib.hu.t2.KafkaAdapter - [ModuleNotFoundError] No module named 'ampel.lsst'
 ampel.contrib.hu.t2.T2MultiXgbClassifier - [ModuleNotFoundError] No module named 'xgboost'
 ampel.contrib.hu.t2.T2RunParsnip - [ModuleNotFoundError] No module named 'timeout_decorator'
 ampel.contrib.hu.t2.T2RunParsnipRiseDecline - [ModuleNotFoundError] No module named 'lcdata'
 ampel.contrib.hu.t2.T2RunSnoopy - [ModuleNotFoundError] No module named 'snpy'
 ampel.contrib.hu.t2.T2TabulatorRiseDecline - [ModuleNotFoundError] No module named 'light_curve'
 ampel.contrib.hu.t2.T2XgbClassifier - [ModuleNotFoundError] No module named 'xgboost'
 ampel.contrib.hu.t3.PlotTransientLightcurves - [ModuleNotFoundError] No module named 'slack_sdk'
 ampel.contrib.hu.t3.SlackSummaryPublisher - [ModuleNotFoundError] No module named 'slack_sdk'
 ampel.contrib.hu.t4.ElasticcTomBridge - [ModuleNotFoundError] No module named 'ampel.lsst'

2025-09-19 18:24:32 ConfigBuilder:505 INFO
 Config file saved as ampel_conf.yaml [4775 lines]

2025-09-19 18:24:32 ConfigCommand:258 INFO
 Total time required: 7 minutes 27 seconds
 

```
## Installing AMPEL on Remote server(wgs):
1. Install poetry at `afs/scratch`. DO NOT install at `$HOME` as you'll run out of space. 

1. Set the poetry cache location also inside `afs/scratch`. By default `.cache` might be still at `$HOME`(https://python-poetry.org/docs/configuration#configuration). Follow the commands:
`cd /afs/scratch`
`touch .cache `
`poetry config –list`
`poetry config cache-dir /path/to/scratch/.cache`


1. Also set the poetry `virtualenvs.prefer-active-python` as `true`, to use the current python of the shell since we want ot use the python of the conda env. Else, poetry may uses its default python version.
`poetry config –list `
`poetry config virtualenvs.prefer-active-python true `



Continue from step 2 of [Installing AMPEL locally](#installing-ampel-locally).

* One last step is to setup the `mongoDB`. wgs provide a central access to `mongoDB`. Get your credentials from :eyes:
* After installing AMPEL set the mongoDB path 


## Running 1000s of parallel AMPEL jobfiles (HTCondor)

This can be achieved by `HTCondor` a high-throughput computing software framework for parallelization. This is readliy available in the wgs. To run multiple AMPEL job files in parallel in the remote server:
* Create a condor submit file (eg: condor.sub). A working example is given below. 
* `condor.sub` submits the job files (.yml) files listed with their full in `jobs.txt`
* A bash scrip **`wrap.sh`** is used that tells the working nodes (where the jobfiles run) to use the python of AMPEL env you have installed. 
* A simple python script **`run_ampel.py`** that runs the jobfile and check for failure and re runs it. 

The jobfle `condor.sub`:
```
Universe       = vanilla
Executable     = wrap.sh
Arguments      = $(ampel_jb)
Error          = log_condor/$BASENAME(ampel_jb).err
Output         = log_condor/$BASENAME(ampel_jb).out
Log            = log_condor/$BASENAME(ampel_jb).log
environment    = "AMPEL_CONFIG_resource.mongo=mongodb://eakshay:p4lV1C910TTFU9N1q7M4@ampeldb.zeuthen.desy.de:27017/admin?authSource=admin&readPreference=primary&appname=MongoDB%20Compass&directConnection=true&ssl=true"
should_transfer_files = YES
when_to_transfer_output = ON_EXIT
transfer_input_files = wrap.sh,run_ampel.py, $(ampel_jb)
max_materialize = 15 

Queue ampel_jb from jobs.txt
```
`max_materialize = 15 ` is used to prevent too many connections at the same time to the archive, which will fail. 
`environment` is used to connect to the mongoDB in wgs

In HTCondor, the Executable is run directly in the execution environment of the worker node. By default, it will just use whatever python is available there (often system python). So, the easiest solution to run a job file in your local python AMPEL environment is to create a shell script that activates your environment (**`wrap.sh`**) and then run the Python script ( **`run_ampel.py`**) to run you jobfile.
The bash script **`wrap.sh`**:
* Activate the local(user) AMPEL env in the worker node
* Runs AMPEL by calling **`run_ampel.py`**

```
#!/bin/bash

# Activate your conda environment
source /afs/ifh.de/user/e/eakshay/scratch/miniforge3/bin/activate ampel_env

# Run your script 
python run_ampel.py "$@"
```

The script **`run_ampel.py`**. 
* Runs an AMPEL jobfile
* Infinite`while loop` to check job failure and rerun. 
```
#!/usr/bin/env python

import subprocess
from subprocess import check_output, STDOUT
import sys
import time
job = sys.argv[1]
cmd = f"ampel job --config  /afs/ifh.de/user/e/eakshay/scratch/Ampel-HU-astro/ampel_conf.yaml --secrets /afs/ifh.de/user/e/eakshay/scratch/vault/vault.yaml --schema {job}"
print(cmd)

while(True):

    try:
        cmd_stdout = check_output(cmd, stderr=STDOUT, shell=True).decode()
        print('Run Success', cmd_stdout)
        break
    except subprocess.CalledProcessError as e:
        log = e.output.decode()
        if log.split('\n')[-2].split()[1] == '404':
            print('...retrying...& sleeping...')
            time.sleep(1)
        else:
            print('Run Failed', log)

    
    
```
