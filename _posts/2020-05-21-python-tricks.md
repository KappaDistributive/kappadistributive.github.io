---
title: Python Tricks For Data Scientist 
layout: single
mathjax: true
category: Data-Science 
tags: [data science, python]
--- 
Recently, I've found myself gravitating toward a small set of 'tricks' in
Python over and over again and thought it would be fun to share and discuss
a few of them with other data scientists. So here they go:

## Cached Data Loader 

Jupyter notebooks are great for exploratory data analysis for many reasons.
One major of which is their persistent state between code execution: You
can load your data once and then iteratively perform all sorts of
modifications / visualizations without having to reload your data over the
network.

Sometimes, however, that's still not enough: If your network connection is
slow or the dataset you're trying to analyze particular large or you need
to restart your kernel frequently for whatever reason, data loading can
become a real pain.

To cope with this, especially now that I'm working remotely on a slow
internet connection, I've found myself using cached data loading more 
frequently. Here's how this may look like in practice:

```python
def _cached_data_loader(
    connector: StorageConnector,
    query: str,
    query_parameters: List[QueryParameter],
    cache_location: Path,
    force_update: bool, 
) -> pandas.DataFrame:
    if force_update or not os.path.isfile(cache_location):
        data = connector.query_to_dataframe(
            query,
            query_parameters
        )
        data.to_pickle(cache_location)
    else:
        data = pd.from_pickle(cache_location)

    return data
    
def load_marketing_data(
    query_parameters: List[QueryParameter] = [],
    cache_location: Path = Path("cache/marketing.pkl"),
    force_update: bool = False
) -> pandas.DataFrame:
    connector = ...
    query = "..."
    return = _cached_data_loader(
        connector,
        query,
        query_parameters,
        cache_location,
        force_update
    )
```

After downloading marketing data over the network, any subsequent call to this
function will load a cached version from disk instead. However, by setting
`force_update = True` you can still refresh your (cached) data at any time.

This can probably be implemented much more elegantly as a decorator. And if you
do, I'd love to hear about it!

## Debugging with Upturn

Since many bugs in data science code only occur on certain input data, debugging
can be challenging and simple debugging tools often seem too restrictive,
whereas more feature complete debugging tools can be intimidating. What I've found
to be useful at times is to abuse IPython as a debugger.
Say, for instance, that a certain pre-processing step in my data pipeline yields
unexpected results. Instead of setting a breakpoint and attempting to understand
what's going on with a debugger, I might add something like this:

```python
def preprocessing(raw_data: pd.DataFrame) -> np.ndarray:
    ...
    buggy code
    ...

    import IPython
    IPython.embed()
    exit(1)

    ...
    return data
```

When reaching `IPython.embed()`, this opens an interactive Python session in
the terminal that allows me to inspect all relevant variables, execute
arbitrary code, experiment with possible solutions, ... When you're done, 
simply exit IPython which will then also exit the program in the following line.

## Diffing and Merging Jupyter Notebooks

Working in close collaboration on challenging problems is easily my favorite
aspect of being a data scientist. Resolving merge conflicts in Jupyter
notebooks, however, that close collaboration inevitably leads to, has been a
rather dreadful experience for me. That is, until I found 
[nbdime](https://github.com/jupyter/nbdime). It's a super handy tool to diff
and merge Jupyter notebooks in your terminal or via their web-based GUI. 

<img src="/images/diagrams/nbmerge-web.png" alt="merging notebooks with nbdime"/>

## Speeding Up BigQuery

Transforming large amounts of data from 
[BigQuery](https://cloud.google.com/bigquery) to Pandas DataFrames, even from
within Google's Compute Engine, can be incredibly slow. Fortunately, Google 
offers an alternative that I've personally experienced to shorten load times
by two orders of magnitude -- BigQuery's Storage API.
Here's how to use it (assuming that this API has been enabled in your project 
and your service account has permissions to use it):

```python
from google.cloud import bigquery, bigquery_storagev1beta1

credentials, project_id = ...

bq_client = bigquery.Client(
    credentials=credentials,
    project=project_id
)

bq_storage_client = bigquery_storagev1beta1.BigQueryStorageClient(
    credentials=credentials
)

query = "..."

data = (
    bq_client.query(query)
    .result()
    .to_dataframe(bq_storage_client=bq_storage_client)
)
```

Keep in mind, however, that this will result in additional costs.
