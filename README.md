# Verification of Election Shapefiles

![PyPI - License](https://img.shields.io/pypi/l/op-verification)
![PyPI](https://img.shields.io/pypi/v/op-verification)

This verification script generates a report that compares a **Precinct-Level Election Shapefile** with expected election results and geometries.

## Installation

This repository is availibe as a package on [PyPI](https://pypi.org/project/op-verification/).

To install `op-verification` from PyPI, run `pip install op-verification`. 

## Sources

* Expected election results (for 2016 Data) are sourced from [MIT Election Data + Science Lab (MEDSL)](https://electionlab.mit.edu/data)
* Expected geometries are sourced from [The United States Census Bureau](https://www.census.gov/) and the [Alaska Division of Elections](http://www.elections.alaska.gov/doc/info/2013-HD-ProclamationPlan.zip)

## Usage

Checkout [`Verification Example Notebook.ipynb`](https://github.com/OpenPrecincts/verification/blob/master/examples/Verification%20Example%20Notebook.ipynb) for examples.

When importing, use:

```python
import op_verification
```

rather than

```python
import op-verification
```

The latter option won't work.

### Inputs and Outputs

**Input:**

* `state_prec_gdf` (GeoDataFrame) containing precinct geometries and election results.
* `state_abbreviation` (str) e.g. 'MA' for Massachusetts
* `source` (str) person or organization that made the 'state_prec_gdf' e.g 'VEST'
* `county_level_results_df` (DataFrame) containing official county-level election results
* `office` (str) office to be evaluated in vote validation e.g. 'U.S. Senate'
* `year` (str) 'YYYY' indicating the year the election took place e.g. '2016'
* `d_col` (str) denotes the column for democratic vote counts in each precinct
* `r_col` (str) denotes the column for republican vote counts in each precinct
* `path` (str) filepath to which the report should be saved (if None it won't be saved)

**Output:**

* `state_report` (StateReport) for the `state_prec_gdf`
* `county_reports` ((CountyReport) list) for the `state_prec_gdf`

#### Schemas

Pay close attention to the name and [`dtype`](https://numpy.org/doc/stable/reference/generated/numpy.dtype.html) of your columns to ensure the script works correctly. It's permissible to include more columns than those listed here (they will just be ignored by the script).

`state_prec_gdf`:

| Column Name | dtype    | example                                           |
|-------------|----------|---------------------------------------------------|
| `d_col`     | int      | 5936                                              |
| `r_col`     | int      | 6395                                              |
| geometry    | geometry | POLYGON ((-71.99365 44.49649, -71.99262 44.496... |
| GEOID       | object   | '01001'                                           |

GEOID is optional for `state_prec_gdf`, but strongly reccomended. [Learn more...](https://github.com/OpenPrecincts/verification#geoid-county-assignment-for-each-precinct)

`county_level_results_df`:

| Column Name | dtype  | example                    |
|-------------|--------|----------------------------|
| county      | object | 'Essex County'             |
| GEOID       | object | '01001'                    |
| party       | object | 'democrat' or 'republican' |
| votes       | int    | 5936                       |

The `county_level_results_df` DataFrame should only contain results for the `office` that's passed as an input.

#### GEOID/County assignment for each precinct

[Wikipedia link](https://en.wikipedia.org/wiki/FIPS_county_code)

This script compares precinct level election results from the `state_prec_gdf` with the expected election results from official state election data records at the state and county level. In both cases the precinct level results are aggregated up to their state and county respectively and then compared to the expected results. Likewise, the precinct geometries are aggregated up to the county level and compared with the county shapefiles from the US Census Bureau.

In order to do the comparisons detailed above, the script needs to know about the makeup of `state_prec_gdf`. Specifically, it needs to know the county (or equivalent) for each precinct and which columns correspond with the votes for the Democratic and Republican candidate for each precinct.

The precincts need to be assigned a county in the form of the county's 5 digit GEOID code described below:

##### GEOID SPEC

Elements of the GEOID column are 5 character strings. The first 2 characters
are the StateFP code and the last 3 characters are the CountyFP code. e.g.

* Massachusetts' StateFP = '25'
* Essex County's CountyFP = '009'
* Essex County, Massachusetts' GEODID = '25009'

If either code has fewer digits than are allocated, the string representation should
be zero-padded from the left. e.g. Alaska (StateFP = 2) should be '02'.

The GEOID may be given for each precinct in `state_prec_gdf` file. In this case, the column must conform to the spec above and be named `'GEOID'`. If the GEOID column is missing then the script will attempt to create it using the [MAUP package](https://github.com/mggg/maup#assigning-precincts-to-districts) to assign each precinct to the county which contains it. Omission of the GEOID label in the input file and failure to assign counties with MAUP (e.g. script throws an exception) will result in the report skipping county level metrics (denoted with -1 metric values).

#### Candidate vote counts

The script needs to know which column contains votes for the Democratic and Republican candidate(s) being reviewed by the script. They can be manually entered as arguments:

* `d_col` denotes the column for Democratic vote counts in each precinct
* `r_col` denotes the column for Republican vote counts in each precinct.

Without those arguments, the script will guess based on the expected number of votes for each candidate.

### Election Year reccomendations

#### 2016 Precinct-Level Election Shapefiles

The `verify.verify_state_2016(...)` function will call `verify.verify_state(...)` and automatically apply 2016 specific defaults:

* Uses Official County Results from the 2016 Presidential Election already in this repository
* Sets year to '2016'
* Sets office to 'President'

Using this funciton for a 2016 Precinct-Level Election Shapefile has the benefit of standardizing 2016 reports. Moreover, it saves you the time of finding official county level results and conforming the data to the expected schema for input data.

#### Non-2016 Precinct-Level Election Shapefile

For non-2016 Precinct-Level Election Shapefiles, you do need to supply a schema-conforming `county_level_results_df` to `verify.verify_state(...)`. I reccomend looking on the Department of State website for the state being validated. For example, I found official county results for Pennsylannia's 2018 election [here](https://electionreturns.pa.gov/)

Checkout [`Verification Example Notebook.ipynb`](https://github.com/OpenPrecincts/verification/blob/master/examples/Verification%20Example%20Notebook.ipynb) for examples of both cases.

## Verification Report Breakdown

### Quality Scores

#### Vote Score

Compute the ratio of votes observed in `state_prec_gdf` to the votes expected (based on official state election data records in `county_level_results_df`) for the democratic and republican candidate. Then the Vote Score is the weighted average of these ratios. [Python Implementation](https://github.com/OpenPrecincts/verification/blob/master/op_verification/verify.py#L71)

* Ideally Vote Score = 1
* A Vote Score above 1 indicates that the **Input** contains more recorded votes than the official state election data
* A Vote Score below 1 indicates that the **Input** contains fewer recorded votes than the official state election data.

#### County Vote Score Dispersion

For each county, compute the square of the difference between the expected number of votes for the democratic and republican candidate. Then, County Vote Score Dispersion is the average of the square difference across all the counties in the state. [Python Implementation](https://github.com/OpenPrecincts/verification/blob/master/op_verification/verify.py#L452)

* Ideally County Vote Score Dispersion = 0
* As the County Vote Score Dispersion increases, so does the degree to which the **Input** differs with respect to official state election data records about the county-level results.

#### Area Difference Score

Compute the symmetric difference between the **Input's** geometries and the expected geometries for that state from the Census Bureau. Then Area Difference Score is the ratio of the symmetric difference's area to the area of the precinct shapefiles. [Python Implementation](https://github.com/OpenPrecincts/verification/blob/master/op_verification/verify.py#L184)

* Ideally Area Difference Score is 0
* As the Area Difference Score increases it indicates a greater geometric difference between the observed geometry in the **Input** and the expected geometry.
* An Area Difference Score of -1 indicates an error was encountered when attempting to compute the metric. Therefore, it is the worst value possible for the Area Difference score. 

### Library Compatibility

Check the **Input** for compatibility with libraries and packages that we hope our end users will be able to apply to the map.

* can_use_maup: `(boolean)` Can use [MAUP](https://github.com/mggg/maup), a geospatial toolkit for redistricting data.
* can_use_gerrychain: `(boolean)` Can use [Gerrychain](https://github.com/mggg/GerryChain) which is useful for applying sensitivity testing via Markov chain Monte Carlo sampling.

### Raw Data

* n_votes_democrat_expected: `(int)` number of votes for the democratic candidate in MEDSL dataset
* n_votes_republican_expected: `(int)` number of votes for the republican candidate in MEDSL dataset
* n_two_party_votes_expected: `(int)` n_votes_republican_expected + n_votes_republican_expected
* n_votes_democrat_observed: `(int)`  number of votes for the democratic candidate in the **Input**
* n_votes_republican_observed: `(int)`  number of votes for the republican candidate in the **Input**
* n_two_party_votes_observed: `(int)`  n_votes_democrat_observed + n_votes_republican_observed
* all_precincts_have_a_geometry: `(int)`  every precinct has a valid geometry
