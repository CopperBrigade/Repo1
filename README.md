# DroneBot
Extraction queries are stored in Redash, for endpoints 
defined in the metadata API, and a series of steps applies business transformation logic. The
final result are endpoint specific flat files, stored in `./data`.

## Manifest
- `./config/`: bash environment variables (IMPORTANT) and python logging configurations
- `./dependencies/`: application dependencies required for runtime, see the 
[directory](https://github.com/pfizer-rd/DroneBot/tree/main/dependencies) for an overview on 
individual components
- `./env/`: openeye licenses and environment yaml with required python packages
- `ProcessADMEData.py`: entry point for individual endpoint processing, called via bash script to 
facilitate parallel endpoint processing
- `drone_runner.sh`: application runner and the **main entry point**

## Installation

### Python Environment (for Windows using Git Bash)

This project was set up and run using **Git Bash** on **Windows**.

1. Navigate into the project directory:
```bash
cd path/to/project-repo
```

Install dependencies:

#### Virtual Environment & Project Execution
1. Create a Virtual Environment (only once)
```bash
python -m venv <your_env_name>
```
Replace `<your_env_name>` with your preferred name (e.g., venv, env).

2. Activate the Virtual Environment
```bash
source <your_env_name>/Scripts/activate
```
⚠️ If activation fails due to script policy restrictions, run this command in PowerShell as Administrator:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```
✅ Once activated, your terminal prompt will change to indicate that the environment is active.

3. Install Required Packages
Make sure you're inside the virtual environment, then run:
```bash
pip install -r env/requirement.txt
```
✅ This installs all dependencies listed in requirement.txt only for this project, not system-wide.

4. Run the Project
After activating the virtual environment, run:
```bash
source <your_env_name>/Scripts/activate    
./drone_runner.sh                          
```
✅ This runs the project within the virtual environment using the installed dependencies.

### Setting Environment Variables (**IMPORTANT**)
Environment variables are referenced across python and bash scripts. Changing the names of these 
variables may break the application and some values must be set before running, which can be 
configured in `./config/env_vars`. The table below provides an overview of all variables
and denotes entries that must be configured prior to running, which can be set either as session 
variables (via `~/.bashrc`) or manually updated in `env_vars`.

#### OpenEye License Setup

DroneBot requires an **OpenEye Toolkit** for chemical structure processing.  
To use OpenEye modules, a valid OpenEye license file must be provided.

1. **Obtain the license file** (`oe_license.txt` or `oe_license.lic`) from your organization’s OpenEye admin.  
2. **Place the file** in the project under `./env/` (or a secure shared location).  
3. **Set the environment variable** in `./config/env_vars` or your `.env` file:

```bash
   OE_LICENSE=./env/oe_license.txt
```

4. **Confirm the license is detected** before running the pipeline:

```bash
echo $OE_LICENSE
```

#### NOTE:
A `.env` file is required in the root directory.  
System needs "jq" (JSON parser) installed (`apt-get install jq`).

| Variable Name  | Description | Must be Set |
| ------------- | ----------- | ------------ |
| OUTPUT_DATA_DIR | _base location for all generated data files, created if nonexistent_ | NO |
| ALL_RUN_INFO | _location for global run log and historical run times, created if nonexistent_ | NO |
| OE_LICENSE | _openeye license file, required to use python module_ | NO |
| METADATA_API_BASE | _URL of Grindah metadata API_ | NO |
| LOGGING_CONF | _location of YAML config file, sets logging level and output files_ | NO |
| ENABLE_PARALLEL_PROCESSING | _enable/disable parallel endpoint processing_ | NO |
| CONCURRENT_JOBS_ALLOWED | _number of concurrent endpoint jobs allowed_ | NO |
| REDASH_API_KEY  | _redash service account API key_ | YES |
| COUCHBASE_USERNAME  | _couchbase service account username_  | YES |
| COUCHBASE_PASSWORD  | _couchbase service account password_ | YES |
| COUCHBASE_HOST |  _hostname of couchbase service, without port information_ | YES |
| METRIC_DB_USER |  _credential information for metric summary database_ | YES |
| METRIC_DB_HOST |  _credential information for metric summary database_ | YES |
| METRIC_DB_PASSWORD |  _credential information for metric summary database_ | YES |

## Running the Script
From the project root directory, activate the dronebot2 micromamba environment and run the following 
command, to submit extraction jobs for each endpoint configured in the metadata API.
```bash
./drone_runner.sh
```

## Understanding Script Output

### Global Run Information
With the default config options, `./global_run_logs` is created, if it doesn't exist, with 
information about all endpoint runs. For a new run event, `event_categories.csv` and 
`run_timings.csv` are appended. A run hash identifies unique endpoint submissions.

| Output Object | Description |
| ------------- | ------------- |
| `event_categories.csv`  | _summary record event categorization information_  |
| `run_timings.csv`  | _timing information about endpoint runs_  |
| `run.log`  | _global queue log from parallel endpoint processing_  |

### Local Endpoint Information
With the default config options, a directory is created inside `./data/` for each model endpoint
containing execution results and logging information.

```bash
.
└── ELOGD
    ├── adme
    │   ├── elogd_events.csv
    │   ├── elogd.csv
    ├── adme_detail
    │   ├── elogd.csv
    │   └── elogd_events.csv
    ├── logs
    │   ├── error.log
    │   └── run.log
    │   └── timing_ELOGD.json
    └── raw
        └── redash_elogd.csv.gz
```

This table summarizes output artifacts.

| Output Object | Description |
| ------------- | ------------- |
| `error.log`  | _warning and error information from previous run_  |
| `run.log`  | _INFO log entries from previous run_  |
| `elogd_events.csv` | _existing couchbase and new rif data, with entries categorized by event_ | 
| `./adme/elogd.csv`  | _final adme data file with new RIF results_ |
| `./adme_detail/elogd.csv`  | _final adme\_details data file with new RIF results_ |
| `./timing_ELOGD.json` | _Store the processing time of each individual endpoint._ |
| `./raw/` | _raw data from RIF pull without any processing_ |

## Logic Overview#
### Fetching New Data
Data is extracted from the RIF Oracle database using Redash and queries must match the naming
convention _GC. See `fetch_redash()` in `DroneRunner.py` for details.

### Filtering Structures
Applies smile integrity for new structures using a rules engine. See `filter_structs()` in 
`DroneRunner.py` for details.

### Clean-Up 
Verifies if new RIF values match model bounds set in the metadata API. See `clean_structs()` in 
`DroneRunner.py` for details.

### Average Replicates
Applies business logic to combine endpoint values. See `avg_structs()` in `DroneRunner.py` for 
details.

### Event Categorization
Assigns a category label to each record in RIF and Couchbase based on the number of field mismatches
between corresponding entries. See `event_cat_structs()` in `DroneRunner.py` for details.

### Changes from Original
_Draft list of adjustments from the original:_

- API Integration (replacing ConfigManager/ConfigDriver)
- Overhauled Logging (making it easier to track down issues/errors)
- Simplified/Streamlined Logic (removed X% of original modules/dependencies)
- Performance Enhancements
    - Couchbase Indices
    - Redash Refreshes via Time Threshold
    - (TODO) Parallel Processing
- Minor Bug Fixes & Improvements 
    - date formatting problem
    - query sourcing issues
    - outdated dependencies
    - documentation
    - (TODO) testing
