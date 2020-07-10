# Using clinicaltrials.gov API for extracting information on COVID-19 antibodies

# Background

1. We will use the "full-study" API to extract results. The API returns a max of 100 hits each time. So we need to iterate through all hits, 100 each time, until the end

2. We're interested in extracting the following data:

   Inclusion criteria:
   - COVID-19 indication
   - Antibody treatments, combination treatments involving antibodies

   Exclusion criteria:
   - Entries describing preclinical or clinical development of diagnostic antibodies, polyclonal antibodies, convalescent plasma therapies, immune globulin intravenous therapies (IGIV), vaccines, small molecules, and recombinant proteins other than immunoglobin (Ig), Ig fragments, and Ig fusion proteins were removed from our collection. Studies and clinical trials without explicitly stating COVID-19 or SARS-CoV-2 as their indication or target were also eliminated


3. Shown below are some useful resources on the JSON data from "full-study" queries. **Note that NOT ALL of these fields will be available for an unique study, so it is recommended to use `try...except` to avoid extracting data from a non-existent field**

   - [List of study fields](https://clinicaltrials.gov/api/info/study_fields_list)
   - [Empty structure of JSON returned from a full-study query](https://clinicaltrials.gov/api/info/study_structure)
   - [Search areas](https://clinicaltrials.gov/api/info/search_areas) : Note that regardless of the hierarchy of the fields in JSON, you can use the search areas directly in the `expr` param (see code)

4. Caveat: Manual inspection of the returned results from the script is strongly recommended to ensure relevancy.


# How to format your query

There are 2 ways:


## 1. Single field search

```python
url="https://ClinicalTrials.gov/api/query/full_studies"
params={
    "expr": "NCT04320615",
    "field": "NCTId",
    "min_rnk": 1,
    "max_rnk": 1,
    "fmt": "JSON"
}

data=requests.get(url, params=params).json()
```

Parameters:
- `expr`: value for the field that you want to earch
- `field`: what field to search. Can only be a single field
- `fields`: fields to return. Only applies to "Study Fields" queries. "Full-Study" queries return all fields,  therefore is NOT affected by this parameter
- `min_rnk` and `max_rnk`: When you submit a query, hits are numbered from 1 to xxxx (the last hit). For full-studies, each time you can only retrieve 100 hits. Use this parameter specifify the "index" of the hits that you want to retrieve. E.g. 1-100, 101-200, ....Use these parameters to iterate through hits to extract them all in to sqlite database.
- `fmt`: format, specify as `"JSON"`



## 2. Multiple Field Search (RECOMMENDED)

A barebone multi-field search request looks like this:

```python
url="https://ClinicalTrials.gov/api/query/full_studies"
params={
    "expr": "AREA[NCTId]NCT04320615",
    "min_rnk": 1,
    "max_rnk": 1,
    "fmt": "JSON"
}

data=requests.get(url, params=params).json()
```

Parameters:
- `expr`: use an expression string. For how to structure the string, see [ref on logical operators](https://clinicaltrials.gov/api/gui/ref/expr).

  A simple example of the string query:
  ```
  AREA[InterventionDescription]antibody  NOT AREA[InterventionType]diagnostic
  ```

  Note that regardless of the hierachy of the field in JSON, you can search the field directly using the string query above. The API also appears to have some abilities to match synonymous words - e.g. COVID-19 will automatically cover covid19, etc.


## 3. How many hits are there in my query?

The number of hits matching your query can be found in the `data['FullStudiesResponse']['NStudiesFound']` parameter. For example:

```python

url="https://ClinicalTrials.gov/api/query/full_studies"

params={
    "expr": "AREA[Condition]COVID-19",
    "fmt": "JSON",
    "min_rnk": 1,
    "max_rnk": 1

}
data=requests.get(url, params=params).json()

# total number of trials in the database
print("Total number of studies: ", data['FullStudiesResponse']['NStudiesAvail'])

# total number of trials found matching your query
print("Total number of trials mathcing query: ", data['FullStudiesResponse']['NStudiesFound'])

```
