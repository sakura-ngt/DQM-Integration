# DQM-Integration

Repository to store input files for the CMSSW [DQM/Integration](https://github.com/cms-sw/cmssw/tree/master/DQM/Integration) package.

## Streamer Files
For the moment this repository is used to store streamer files that are needed as input for the unitTest.

Currently the unitTest that make use of input streamer files are:
* the `onlinebeammonitor_dqm_sourceclient_cfg.py` client, it reads the `streamDQM` (prepared such that only the hltOnlineBeamSpot is present in the file) streamer files regenerated from run 381594 (from 2024E pp run [OMS link](https://cmsoms.cern.ch/cms/runs/report?cms_run=381594&cms_run_sequence=GLOBAL-RUN)):
   ```
   run381594/run381594_ls1000_streamDQM_pid1388480.dat
   run381594/run381594_ls1000_streamDQM_pid1388480.jsn
   ```
* the `beamhlt_dqm_sourceclient-live_cfg.py` client, it reads the `streamDQMOnlineBeamspot` streamer files regenerated from run 381594 (from 2024E pp run [OMS link](https://cmsoms.cern.ch/cms/runs/report?cms_run=381594&cms_run_sequence=GLOBAL-RUN)):
   ```
   run381594/run381594_ls1000_streamDQMOnlineBeamspot_pid1752643.dat
   run381594/run381594_ls1000_streamDQMOnlineBeamspot_pid1752643.jsn
   ```
* the `ecalgpu_dqm_sourceclient-live_cfg.py`, `hcalgpu_dqm_sourceclient-live_cfg.py`, `pixelgpu_dqm_sourceclient-live_cfg.py` and `pfgpu_dqm_sourceclient-live_cfg.py` read the `streamDQMGPUvsCPU` streamer files regenerated from run 398183  (from Run2025G pp run [OMS link](https://cmsoms.cern.ch/cms/runs/report?cms_run=398183&cms_run_sequence=GLOBAL-RUN)):
   ```
   run398183_ls0142_streamDQMGPUvsCPU_pid3119228.dat
   run398183_ls0142_streamDQMGPUvsCPU_pid3119228.jsn
   ```
* the `sistrip_approx_dqm_sourceclient-live_cfg.py` reads the `streamDQM` streamer files regenerated from run 362321 (from 2022 HI run, [OMS link](https://cmsoms.cern.ch/cms/runs/report?cms_run=362321&cms_run_sequence=GLOBAL-RUN), though they have been re-HLT'ed, see for more details at [CMSHLT-2884](https://its.cern.ch/jira/browse/CMSHLT-2884)):
   ```
   run362321/run362321_ls0231_streamHIDQM_pid276864.dat
   run362321/run362321_ls0231_streamHIDQM_pid276864.jsn
   ```
* the `scouting_dqm_sourceclient-live_cfg.py` reads the `streamDQMOnlineScouting` streamer files generated from run 398183 (from Run2025G pp run, [OMS link](https://cmsoms.cern.ch/cms/runs/report?cms_run=398183&cms_run_sequence=GLOBAL-RUN)):
   ```
   run398183/run398183_ls0142_streamDQMOnlineScouting_pid2737183.dat
   run398183/run398183_ls0142_streamDQMOnlineScouting_pid2737183.jsn
   ```
* the `ngt_dqm_sourceclient-live_cfg.py` reads the `streamDQMTestDataScouting` streamer files generated from run 402360 (from Run2026B pp run, [OMS link](https://cmsoms.cern.ch/cms/runs/report?cms_run=402360&cms_run_sequence=GLOBAL-RUN)):
   ```
   run402360/run402360_ls0193_streamDQMTestDataScouting_pid285513.dat
   run402360/run402360_ls0193_streamDQMTestDataScouting_pid285513.jsn	
   ```

## Recipe to regenerate Streamer files (when streamer layout gets broken)

In repsonse to issue https://github.com/cms-sw/cmssw/issues/45224, streamer files have been regenerated in a release that contains https://github.com/cms-sw/cmssw/pull/44978 (in this case `CMSSW_14_0_9_MULTIARCHS`.

### Recipe for `streamDQM`

This was done using the following script for the pp data:

```bash
#!/bin/bash -ex

RUNNUMBER=381594
LUMISECTION=1000

# cmsrel CMSSW_14_0_9_MULTIARCHS
# cd CMSSW_14_0_9_MULTIARCHS/src
# cmsenv
# scram b

INPUTFILE=root://eoscms.cern.ch//store/express/Run2024E/ExpressPhysics/FEVT/Express-v1/000/381/594/00000/1e2c895f-a250-45be-a7ff-ee95e636a6e9.root
rm -rf run${RUNNUMBER}*

# run on 300 events of LS 1000, with 300 events per input file
convertToRaw -f 300 -l 300 -r ${RUNNUMBER}:${LUMISECTION} -o . -- "${INPUTFILE}"

tmpfile=$(mktemp)
hltConfigFromDB --runNumber "${RUNNUMBER}" > "${tmpfile}"
cat <<@EOF >> "${tmpfile}"
process.load("run${RUNNUMBER}_cff")
process.hltOnlineBeamSpotESProducer.timeThreshold = int(1e6)

# to run without any HLT prescales
del process.PrescaleService
del process.MessageLogger
process.load('FWCore.MessageLogger.MessageLogger_cfi')

process.options.numberOfThreads = 32
process.options.numberOfStreams = 32

process.options.wantSummary = True
# # to run using the same HLT prescales as used online in LS 1000
# process.PrescaleService.forceDefault = True
@EOF
edmConfigDump "${tmpfile}" > hlt.py

cmsRun hlt.py &> hlt.log
```

### Recipe for `streamHIDQM`

While the following script for the HIon data:
```bash
#!/bin/bash -ex

# cmsrel CMSSW_16_1_X_2026-03-03-2300
# cd CMSSW_16_1_X_2026-03-03-2300/src
# cmsenv
# scram b

# run 362321, LSs 231-232

RUNNUMBER=362321
LUMISECTION=231

INPUTFILE=root://eoscms.cern.ch//eos/cms/store/user/cmsbuild//store/hidata/HIRun2022A/HITestRaw0/RAW/v1/000/362/321/00000/f467ee64-fc64-47a6-9d8a-7ca73ebca2bd.root

HLTMENU=/dev/CMSSW_16_0_0/HIon/V31

rm -rf run${RUNNUMBER}*

# run on 100 events of LS 231, with 100 events per input file
convertToRaw -f 100 -l 100 -r ${RUNNUMBER}:${LUMISECTION} -s rawDataRepacker -o . -- "${INPUTFILE}"

tmpfile=tmp.py
hltConfigFromDB --configName "${HLTMENU}" > "${tmpfile}"
sed -i 's|process = cms.Process( "HLT" )|from Configuration.Eras.Era_Run3_cff import Run3\nprocess = cms.Process( "HLT", Run3 )|g' "${tmpfile}"
cat <<@EOF >> "${tmpfile}"
process.load("run${RUNNUMBER}_cff")
#process.hltOnlineBeamSpotESProducer.timeThreshold = int(1e6)

# override the GlobalTag, connection string and pfnPrefix
from Configuration.AlCa.GlobalTag import GlobalTag as customiseGlobalTag
process.GlobalTag = customiseGlobalTag(
    process.GlobalTag,
    globaltag = "160X_dataRun3_HLT_v1",
    conditions = "L1Menu_CollisionsHeavyIons2025_v1_0_3_xml,L1TUtmTriggerMenuRcd,frontier://FrontierProd/CMS_CONDITIONS,,9999-12-31 23:59:59.000"
)

# run the Full L1T emulator, then repack the data into a new RAW collection, to be used by the HLT
from HLTrigger.Configuration.CustomConfigs import L1REPACK
process = L1REPACK(process, "uGT")

# to run without any HLT prescales
del process.PrescaleService

# # to run using the same HLT prescales as used online in LS 231
# process.PrescaleService.forceDefault = True
@EOF
edmConfigDump "${tmpfile}" > hlt.py

bash -c 'echo $$ > cmsrun.pid; exec cmsRun hlt.py &> hlt.log'
job_pid=$(cat cmsrun.pid)
echo "cmsRun is running with PID: $job_pid"

# remove input files to save space
rm -f run${RUNNUMBER}/run${RUNNUMBER}_ls0*_index*.*

# prepare the files by concatenating the .ini and .dat files
mkdir -p prepared
cat run${RUNNUMBER}/run${RUNNUMBER}_ls0000_streamHIDQM_pid${job_pid}.ini run${RUNNUMBER}/run${RUNNUMBER}_ls0${LUMISECTION}_streamHIDQM_pid${job_pid}.dat > prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamHIDQM_pid${job_pid}.dat
cp run${RUNNUMBER}/run${RUNNUMBER}_ls0${LUMISECTION}_streamHIDQM_pid${job_pid}.jsn prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamHIDQM_pid${job_pid}_prep.jsn

# now remove the extra 0
input="prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamHIDQM_pid${job_pid}_prep.jsn"
output="prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamHIDQM_pid${job_pid}.jsn"

jq '
  .data as $d |
  .data = (
    reduce range(0; $d|length) as $i ([];
      if ($i > 0 and .[-1] == "0" and $d[$i] == "0")
      then .
      else . + [$d[$i]]
      end
    )
  )
' "$input" > "$output"

rm -fr $input
rm -fr run${RUNNUMBER}* hlt.* cmsrun.pid dump.py __pycache__
mv prepared run${RUNNUMBER}
```

### Recipe for `streamDQMOnlineScouting`

The streamer files for the `streamDQMOnlineScouting` were prepared using the scouting specific menu:
```bash
#!/bin/bash -ex

# cmsrel CMSSW_16_1_X_2026-01-07-2300
# cd CMSSW_16_1_X_2026-01-07-2300/src
# cmsenv
# scram b

RUNNUMBER=398183 # 2025 EphemeralHLTPhysics
LUMISECTION=142

# LS 174 
INPUTFILES="root://eoscms.cern.ch//store/data/Run2025G/EphemeralHLTPhysics0/RAW/v1/000/398/183/00000/002bbd0c-b9ed-4758-b7a6-e2e13149ca34.root"
rm -rf run${RUNNUMBER}*

# run on 5000 events of given LS, with 1000 event limits per input file
convertToRaw -l 5000 -f 1000 -r ${RUNNUMBER}:${LUMISECTION} -o . -- ${INPUTFILES}

tmpfile=$(mktemp)

hltConfigFromDB --configName /users/jprendi/ScoutingOnlineDQM/Test0/HLT/V7 > dump.py

cat <<@EOF >> dump.py
process.load("run${RUNNUMBER}_cff")
del process.PrescaleService
del process.MessageLogger
process.load('FWCore.MessageLogger.MessageLogger_cfi')

process.GlobalTag.globaltag = cms.string('150X_dataRun3_HLT_v1')

process.options.numberOfThreads = 32
process.options.numberOfStreams = 32

process.options.wantSummary = True
@EOF

edmConfigDump dump.py > hlt.py

bash -c 'echo $$ > cmsrun.pid; exec cmsRun hlt.py &> hlt.log'
job_pid=$(cat cmsrun.pid)
echo "cmsRun is running with PID: $job_pid"

# remove input files to save space
rm -f run${RUNNUMBER}/run${RUNNUMBER}_ls0*_index*.*

# prepare the files by concatenating the .ini and .dat files
mkdir -p prepared
cat run${RUNNUMBER}/run${RUNNUMBER}_ls0000_streamDQMOnlineScouting_pid${job_pid}.ini run${RUNNUMBER}/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMOnlineScouting_pid${job_pid}.dat > prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMOnlineScouting_pid${job_pid}.dat
cp run${RUNNUMBER}/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMOnlineScouting_pid${job_pid}.jsn prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMOnlineScouting_pid${job_pid}_prep.jsn

# now remove the extra 0
input="prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMOnlineScouting_pid${job_pid}_prep.jsn"
output="prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMOnlineScouting_pid${job_pid}.jsn"

jq '
  .data as $d |
  .data = (
    reduce range(0; $d|length) as $i ([];
      if ($i > 0 and .[-1] == "0" and $d[$i] == "0")
      then .
      else . + [$d[$i]]
      end
    )
  )
' "$input" > "$output"

rm -fr $input
rm -fr run${RUNNUMBER}* hlt.* cmsrun.pid dump.py __pycache__
mv prepared run${RUNNUMBER}
```

### Recipe for `streamDQMTestDataScouting`

The streamer files for the `streamDQMTestDataScouting` were prepared using the following script:
```bash
#!/bin/bash -ex

# cmsrel CMSSW_16_0_6_patch1
# cd CMSSW_16_0_6_patch1/src
# cmsenv

RUNNUMBER=402360
LUMISECTION=193

hltLabel=testAddTriggerEvents
hltLabel1="${hltLabel}_hlt1"
hltLabel2="${hltLabel}_hlt2"

INPUTFILE="root://eoscms.cern.ch//store/data/Run2026B/EphemeralHLTPhysics0/RAW/v1/000/402/360/00000/aa48b021-759e-4592-86e5-75eee46b88fc.root"
MENU=/online/collisions/2026/2e34/v1.2/HLT/V2

########################################
# Helpers
########################################

run_cms() {
    local cfg=$1
    local log=$2

    bash -c "echo \$\$ > cmsrun.pid; exec cmsRun ${cfg} >& ${log}"
    job_pid=$(cat cmsrun.pid)
    echo "cmsRun is running with PID: $job_pid"
}

prepare_output() {
    local tag=$1   # old_runXXX or new_runXXX

    rm -f run${RUNNUMBER}/run${RUNNUMBER}_ls0*_index*.*

    mkdir -p prepared

    local base="run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMTestDataScouting_pid${job_pid}"

    cat run${RUNNUMBER}/run${RUNNUMBER}_ls0000_streamDQMTestDataScouting_pid${job_pid}.ini \
        run${RUNNUMBER}/${base}.dat \
        > prepared/${base}.dat

    cp run${RUNNUMBER}/${base}.jsn \
       prepared/${base}_prep.jsn

    jq '
      .data as $d |
      .data = (
        reduce range(0; $d|length) as $i ([];
          if ($i > 0 and .[-1] == "0" and $d[$i] == "0")
          then .
          else . + [$d[$i]]
          end
        )
      )
    ' "prepared/${base}_prep.jsn" > "prepared/${base}.jsn"

    rm -f prepared/${base}_prep.jsn
    rm -rf run${RUNNUMBER}* hlt.* cmsrun.pid dump.py __pycache__

    mv prepared ${tag}
}

convert_input() {
    convertToRaw -f 1000 -l 1000 -r ${RUNNUMBER}:${LUMISECTION} -o . -- "${INPUTFILE}"
}

########################################
# Setup
########################################

rm -rf run${RUNNUMBER}*

hltConfigFromDB --configName ${MENU} > ${hltLabel1}.py

cat <<@EOF >> ${hltLabel1}.py
process.load("run${RUNNUMBER}_cff")

del process.PrescaleService

streamPaths = [foo for foo in process.endpaths_() if foo.endswith('Output')]
streamPaths.remove('DQMTestDataScoutingOutput')
for foo in streamPaths:
    process.__delattr__(foo)
@EOF

########################################
# First run
########################################

convert_input
run_cms "${hltLabel1}.py" "${hltLabel1}.log"
prepare_output "${RUNNUMBER}"
```

### Recipe for `streamDQMGPUvsCPU`

The streamer files for the `streamDQMGPUVsCPU` were prepared using the following script:
```bash
#!/bin/bash -ex
RUNNUMBER=398183 # 2025 EphemeralHLTPhysics
LUMISECTION=142

# cmsrel CMSSW_16_1_X_2026-01-21-2300 
# cd CMSSW_16_1_X_2026-01-21-2300/src/
# cmsenv

INPUTFILE="root://eoscms.cern.ch//store/data/Run2025G/EphemeralHLTPhysics0/RAW/v1/000/398/183/00000/002bbd0c-b9ed-4758-b7a6-e2e13149ca34.root"
rm -rf run${RUNNUMBER}*

# run on 500 events of LS, with 500 events per input file
convertToRaw -f 25 -l 25 -r ${RUNNUMBER}:${LUMISECTION} -o . -- "${INPUTFILE}"

tmpfile=$(mktemp)
hltConfigFromDB --configName /dev/CMSSW_16_0_0/GRun/V7  > "${tmpfile}"
cat <<@EOF >> "${tmpfile}"
process.load("run${RUNNUMBER}_cff")

# to run without any HLT prescales
del process.PrescaleService
del process.MessageLogger
process.load('FWCore.MessageLogger.MessageLogger_cfi')

process.options.numberOfThreads = 32
process.options.numberOfStreams = 32

process.options.wantSummary = True
process.GlobalTag.globaltag = cms.string( "150X_dataRun3_HLT_v1" )
# # to run using the same HLT prescales as used online in LS 1000
# process.PrescaleService.forceDefault = True

# customization for the menu
from HLTrigger.Configuration.customizeHLTforCMSSW import *
process = customizeHLTfor49799(process)
process = customizeHLTfor49852(process)

## just output the GPU vs CPU output
streamPaths = [foo for foo in process.endpaths_() if foo.endswith('Output')]
streamPaths.remove('DQMGPUvsCPUOutput')
for foo in streamPaths:
    process.__delattr__(foo)
@EOF
edmConfigDump "${tmpfile}" > hlt.py

cmsRun hlt.py &> hlt.log

bash -c 'echo $$ > cmsrun.pid; exec cmsRun hlt.py &> hlt.log'
job_pid=$(cat cmsrun.pid)
echo "cmsRun is running with PID: $job_pid"

# remove input files to save space
rm -f run${RUNNUMBER}/run${RUNNUMBER}_ls0*_index*.*

# prepare the files by concatenating the .ini and .dat files
mkdir -p prepared

cat run${RUNNUMBER}/run${RUNNUMBER}_ls0000_streamDQMGPUvsCPU_pid${job_pid}.ini run${RUNNUMBER}/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMGPUvsCPU_pid${job_pid}.dat > prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMGPUvsCPU_pid${job_pid}.dat
cp run${RUNNUMBER}/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMGPUvsCPU_pid${job_pid}.jsn prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMGPUvsCPU_pid${job_pid}_prep.jsn

# now remove the extra 0

input="prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMGPUvsCPU_pid${job_pid}_prep.jsn"
output="prepared/run${RUNNUMBER}_ls0${LUMISECTION}_streamDQMGPUvsCPU_pid${job_pid}.jsn"

jq '
  .data as $d |
  .data = (
    reduce range(0; $d|length) as $i ([];
      if ($i > 0 and .[-1] == "0" and $d[$i] == "0")
      then .
      else . + [$d[$i]]
      end
    )
  )
' "$input" > "$output"

rm -fr $input
rm -fr run${RUNNUMBER}* hlt.* cmsrun.pid dump.py __pycache__
mv prepared run${RUNNUMBER}
```

## Possible extenstions

There are two more clients for which the unitTets could be activated in the future, namely:
```
 <!-- streamDQMCalibration is required -->
 <!-- <test name="TestDQMOnlineClient-ecalcalib_dqm_sourceclient" command="runtest.sh ecalcalib_dqm_sourceclient-live_cfg.py" /> -->
 <!-- streamDQMCalibration is required -->
 <!-- <test name="TestDQMOnlineClient-hcalcalib_dqm_sourceclient" command="runtest.sh hcalcalib_dqm_sourceclient-live_cfg.py" /> -->
```
