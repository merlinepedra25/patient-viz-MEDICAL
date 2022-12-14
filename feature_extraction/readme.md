# Feature Extraction

A binary feature matrix can be extracted using the `extract.py` script.
The output is a CSV table with columns `id` and named after the event types:
`${group_id}__${type_id}` where `${group_id}` and `${type_id}` are as defined
[in the specification](../spec.md).
Additionally, if a tagged cohort (as returned by `./merge.py`) is used there are
two more columns `outcome` (`1` indicates a case) and `test` (`1` indicates that
this patient is used for model verification). Normally, you want to define
patient cohorts first and then extract features as mentioned in the chapters below
(Note: all commands in this file need to be run from this folder).
However, if you really want to create feature vectors for all patients you can run
the following command:

```bash
./extract.py -w cohort.txt --age-time 20100101 --to 20100101 -o output.csv -f ../format.json -c ../config.txt -- ../cms
```

Be aware that the run time is approximately ~1h.

TODO Currently medication is ignored -> needs a command line way to specify age bin size and ignored groups

For more information about arguments call `./extract.py -h`.
You can use `./build_dictionary.py -c config.txt --lookup ${column_name...}`
from the root folder to look up proper names for the columns.
Alternatively you can dump all column names with:
`head -n 1 feature_extraction/output.csv | sed "s/,/ /g" | ./build_dictionary.py -o feature_extraction/headers.json -c config.txt --lookup -`

# Feature Extraction using header specification

You can define aggregation and columns using a header specification. The command
then looks like:

```bash
./extract.py --age-time 20100101 --to 20100101 -o output_large.csv -f ../format.json -c ../config.txt -s header_spec.json -- ../cms/
```

The header spec defines aggregation methods and types.

# Feature Extraction using Cohorts

In order to get meaningful feature vectors you need to define two cohorts: *cases* and *controls*.
This can be done using the `./cohort.py` query script. The syntax for queries
can be found [here](cohort.py#L105-118). After that you need to merge the
cohorts using `./merge.py` in order to get a tagged cohort. This is needed to
guarantee matching columns in the feature vector for both cohorts and
also to split a test set for verification purposes (be sure to set a seed
in order to get reproducible results). The full commands are as follows
assuming the cohort queries are in `cases.txt` and `control.txt`
respectively.

```bash
./cohort.py --query-file cases.txt -f ../format.json -c ../config.txt -o cohort_cases.txt -- ../cms
./cohort.py --query-file control.txt -f ../format.json -c ../config.txt -o cohort_control.txt -- ../cms
./merge.py --cases cohort_cases.txt --control cohort_control.txt -o cohort.txt --test 30 --seed 0
./extract.py -w cohort.txt --age-time 20100101 --to 20100101 -o output.csv -f ../format.json -c ../config.txt -- ../cms
cd ..
head -n 1 feature_extraction/output.csv | sed "s/,/ /g" | ./build_dictionary.py -o feature_extraction/headers.json -c config.txt --lookup -
cd -
```

This creates the feature vectors in `output.csv` and readable column headers in `headers.json`.
The column headers can be mapped to the csv and the output size can be limited via the `process_csv.py` command.

```bash
./process_csv.py -c ../config.txt -i output.csv -n headers.json -l 1000 -o output_small.csv --seed 0
```

# Feature Extraction with Shelve Input

Feature extraction using a shelve db is the same as above except you need to pipe
the patient data into your scripts. The following commands assume you updated
your `config.txt` (absolute file paths are recommended; relative paths are from
the location of the config file) and `format_shelve.json` has the correct headers.

```bash
../shelve_access.py --all -c ../config.txt | ./cohort.py --query-file cases.txt -f ../format_shelve.json -c ../config.txt -o cohort_cases.txt -- -
../shelve_access.py --all -c ../config.txt | ./cohort.py --query-file control.txt -f ../format_shelve.json -c ../config.txt -o cohort_control.txt -- -
./merge.py --cases cohort_cases.txt --control cohort_control.txt -o cohort.txt --test 30 --seed 0
../shelve_access.py --all -c ../config.txt | ./extract.py -w cohort.txt --age-time 20100101 --to 20100101 -o output.csv -f ../format_shelve.json -c ../config.txt -- -
cd ..
head -n 1 feature_extraction/output.csv | sed "s/,/ /g" | ./build_dictionary.py -o feature_extraction/headers.json -c config.txt --lookup -
cd -
```

# Training Models

Using the previously created feature vectors we can now use them to train models.
The following commands assume you created the feature vectors as described above
and stored them as `output.csv`. Also, it is assumed that you ran `../setup.sh --pip`
in order to install the required python libraries.

```bash
./train.py --in output.csv --out model --seed 0 --model reg -v 20
```

For the linear regression model the output consists of 3 files.
`reg_model_scklearn.pkl` contains the model as python pickle.
`reg_model_bias.txt` contains the bias of the model and
`reg_model_weights.txt` contains the feature weights as a CSV table
where the column names equal the feature names.
The prediction for one patient can be computed by the following formula

```javascript
1 / (1 + Math.exp( -( bias + w_a + w_b + w_e + w_h + ??? ) )) > 0.5
```

Where `bias` is the bias of the model and `w_a` is the weight of feature `a`
and so on. Features are only included in the sum if the patient had an occurrence of them.
