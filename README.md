# MIMICExtractEasy

MIMICExtractEasy is a set of forks of several repos that include
enhancments that make it much easier to produce and use a MIMIC-Extract dataset

## What is MIMIC-Extract?

The [paper on MIMIC-Extract](https://arxiv.org/abs/1907.08322) gives all the detail on what MIMIC-Extract provides beyond the raw MIMIC-III data, but in a nutshell it is a data cleaning/preprocessing/cohort selection tool. Out of the box, it:

* Provides some basic cohort selection filters (e.g. minimum age, population size)
* Unifies/converts units for a suite of labs/vitals events
* Extracts some data from notes
* Unifies ITEMIDs for the same or similar items (based on medical ontology knowledge, at one of two levels of grouping)
* Eliminates extreme outliers and clamps outliers to valid ranges

As a particular enhancement, it includes an interventions table which includes features representing several classes of ICU interventions, which the authors used to train models for onset/wean prediction tasks.

The [code for MIMIC-Extract is available on GitHub](https://github.com/MLforHealth/MIMIC_Extract). The authors built it to be extensible, allowing users to extend the pipeline to include additional elements or customize cohort selection logic.

## What's not easy already?

MIMIC-Extract's code runs with some relatively old libraries, including Pandas 0.24 and spaCy 2, and Python 3.6. The authors helpfully provide an Anaconda environment YAML file, which makes it relatively easy to install all the original dependencies, but dthat solution only goes so far (conda cannot
solve the dependency set on recent macOS versions, for instance).

Also, MIMIC-Extract utilizes the [mimic-code](https://github.com/MIT-LCP/mimic-code) database representation of the MIMIC-III dataset and its excellent collection of [concepts](https://github.com/MIT-LCP/mimic-code/tree/main/mimic-iii/concepts)—but this requires the user to set up PostgreSQL and setup the MIMIC-III database as a prerequisite, which adds complexity and is a barrier to entry.

## How does this project make it easier?

Here are the forks and what they bring to the table:

* [SphtKr/mimic-code](https://github.com/SphtKr/mimic-code)
  * _Track [upstream PR #1529](https://github.com/MIT-LCP/mimic-code/pull/1529)!_
  * Adds support for DuckDB—an embedded database—to build the database and concepts, which can be accomplished with a single scripted command
* [SphtKr/MIMIC_Extract](https://github.com/SphtKr/MIMIC_Extract)
  * _Upstream PR coming soon_
  * Updates MIMIC-Extract with modernized dependencies that does not need an Anaconda environment
  * Adds support to MIMIC-Extract for reading the DuckDB version of the database instead of PostgreSQL
* [SphtKr/PyHealth](https://github.com/SphtKr/PyHealth/)
  * _Track [upstream PR #136](https://github.com/sunlabuiuc/PyHealth/pull/136)!_
  * Adds a basic dataset class to the PyHealth library supporting the MIMIC-Extract output, so you can use the MIMIC-Extract output with PyHealth's easy-to-use APIs for model development and analysis.

 See ["Why DuckDB"](https://duckdb.org/why_duckdb) for a discussion of how DuckDB may be faster for your MIMIC dataset analysis needs, but for our purposes it is used because it is an "embedded" database (like [SQLite](https://www.sqlite.org/index.html)) and is therefore much easier to set up and use compared to a "client-server" database like PostgreSQL—it also has a number of features beyond SQLite that made porting the mimic-code concepts possible.

 ## Can you make it _even easier?_

 Yes! Simply follow the instructions below...

 > NOTE: While easy, this will take a lot of space—from 25-60GB

0. (optional) Create and/or activate a Python virtual environment 
```sh
# Create a venv...
python3 -m venv ./venv
# Activate it...
. ./venv/bin/activate
```

1. Check out this repository and its submodules
```sh
git clone https://github.com/SphtKr/MIMICExtractEasy
# cd into the directory...
cd ./MIMICExtractEasy
# init the submodules - this checks out the forks listed above...
git submodule update --init --recursive
```

2. Download a MIMIC-III dataset
```sh
# move to the `data` directory...
cd ./data
# The full dataset, if you have PhysioNet access (about 7GB)...
wget -r -N -c -np -nH --cut-dirs=1 --user YOURUSERNAME --ask-password https://physionet.org/files/mimiciii/1.4/
# OR the demo dataset...
wget -r -N -c -np https://physionet.org/files/mimiciii-demo/1.4/
# We'll put these locations in some variables for later...
DATA_DIR=$(readlink -f .)
MIMIC_DATA_DIR=$(readlink -f ./physionet.org/files/mimiciii*/1.4/)
```

3. Build the MIMIC-III DuckDB database with concepts
```sh
# move to the DuckDB build directory...
cd ../mimic-code/mimic-iii/buildmimic/duckdb/
# install the requirements...
pip3 install -r ./requirements.txt
# Build the database!
python3 import_duckdb.py ${MIMIC_DATA_DIR} ${DATA_DIR}/mimic3.db --skip-indexes
# Make the concepts (you can do this all in one step but it might run out of memory)...
python3 import_duckdb.py ${MIMIC_DATA_DIR} ${DATA_DIR}/mimic3.db --skip-tables --make-concepts
```
The above may take some time, it depends a great deal on your hardware.

4. Create a MIMIC-Extract dataset
```sh
# Move to the MIMIC_Extract directory...
cd ../../../../MIMIC_Extract/
# Install the requirements...
pip3 install -r ./requirements.txt
# Make a destiation directory for the output...
mkdir ${DATA_DIR}/extract
# Run mimic_direct_extract.py (see MIMIC-Extract docs for additional options)...
python3 mimic_direct_extract.py \
  --duckdb_database=${DATA_DIR}/mimic3.db \
  --duckdb_schema=main \
  --resource_path=./resources \
  --plot_hist=0 \
  --out_path=${DATA_DIR}/extract
```
This may take several hours. If you don't want the notes output, adding the `--extract_notes=0` option can save a lot of time.

The resulting extract dataset should be in `${DATA_DIR}/extract`

5. Use PyHealth
```sh
# install the PyHealth fork
cd ../
pip install ./PyHealth
```
Now you can use the `MIMICExtractDataset` in your Python code:
```Python
from pyhealth.datasets import MIMICExtractDataset

dataset = MIMICExtractDataset(
    root="./data/extract",
    tables=[
        "C",
        "vitals_labs",
        "interventions",
    ],
    dev=True,
    refresh_cache=True,
    itemid_to_variable_map='./MIMIC_Extract/resources/itemid_to_variable_map.csv'
)
dataset.stat()
dataset.info()
```

## What else?

You can use the MIMIC-Extract authors' original [Jupyter notebooks](https://github.com/SphtKr/MIMIC_Extract/tree/duckdb-support/notebooks)
to run some benchmark ML models. These two are good ones to try and only required the "LEVEL2"
grouped version of the dataset generated with the commands above:

* [Baselines for Intervention Prediction - Vasopressor.ipynb](https://github.com/SphtKr/MIMIC_Extract/tree/duckdb-support/notebooks/Baselines%20for%20Intervention%20Prediction%20-%20Vasopressor.ipynb)
* [Baselines for Intervention Prediction - Mechanical Ventilation.ipynb](https://github.com/SphtKr/MIMIC_Extract/tree/duckdb-support/notebooks/Baselines%20for%20Intervention%20Prediction%20-%20Mechanical%20Ventilation.ipynb)

You will need to update the `DATAFILE` variable in the notebook to point to the `all_hourly_data.h5`
file in your generated dataset.

Note that the sklearn noteboooks require both the "LEVEL2" grouping and the "raw" version, which you can 
generate by adding the `--no_group_by_level2` argument to the above call to `mimic_direct_extract.py`.
