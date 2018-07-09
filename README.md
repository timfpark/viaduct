# Viaduct

Viaduct makes manage data pipeline configuration easier and cleaner. It enables you to replace code that looks like this:

```
sports_blob_service =  BlockBlobService(
    account_name='sports',
    account_key='rgFEEDDEADBEEFAPwbfxgdFSW6vKQEAarHua2HSFYWTU1DbR6pFy3EQ=='
)

byte_data = sports_blob_service.get_blob_to_bytes(
    'captures',
    'soccer'
)
```

With code generalized to work with the configured dataset via [12 factor engineering standards](https://12factor.net/) like this:

```
import os

sports_blob_service =  BlockBlobService(
    account_name=os.environ['CSE_ML_SPORTS_ACCOUNTNAME'],
    account_key=os.environ['CSE_ML_SPORTS_KEY']
)

byte_data = sports_blob_service.get_blob_to_bytes(
    os.environ['CSE_ML_SOCCER_CONTAINER'],
    os.environ['CSE_ML_SOCCER_PATH']
)
```

Viaduct accomplishes this by storing configuration details in a centrally managed catalog:

```
$ viaduct catalog add https://cse-ml-catalog/
$ viaduct catalogs list

CATALOG         DESCRIPTION
cse-ml          CSE ML Group
nyu             NYU's canonical ML datasets
```

It also makes it easier to create, discover, and consume datasets. For example, we can find the dataset we need for the previous data science project using:

```
$ viaduct datasets list
CATALOG     DATASET         DESCRIPTION
...
cse-ml      soccer          Soccer sports performance captures.
nyu         minst-images    MINST images
nyu         minst-labels    MINST image labels
```

Let's say we want to use the `soccer` dataset with a jupyter notebook to do some data science work on it. Instead of having hand copy credentials painfully and insecurely into the source code or managing brittle configuration details in local environmental variables, we can do this:

```
$ viaduct dataset loadenv soccer --catalog cse-ml
$ jupyter notebook
```

Under the covers, the `viaduct` tool fetches the `soccer` dataset connection details from the catalog at `https://cse-ml-catalog` and populates environment variables with all of the configuration and authentication details you need to consume the dataset:

```
$ printenv | grep 'CSE_ML'
CSE_ML_SPORTS_ACCOUNTNAME=sports
CSE_ML_SPORTS_KEY=rglhUbRG/557nmOGwdzFEEDDEADBEEFAPwbfxgdxndzuvpk6nFSW6vKQEAarHua2HSFYWTU1DbR6pFy3EQ==
CSE_ML_SOCCER_CONTAINER=soccer
CSE_ML_SOCCER_PATH=captures
```

When you start `jupyter`, it automatically imports these environmental variables, so our generalized version of the code above will now execute.

```
import os

sports_blob_service =  BlockBlobService(
    account_name=os.environ['CSE_ML_SPORTS_ACCOUNTNAME'],
    account_key=os.environ['CSE_ML_SPORTS_KEY']
)

byte_data = sports_blob_service.get_blob_to_bytes(
    os.environ['CSE_ML_SOCCER_CONTAINER'],
    os.environ['CSE_ML_SOCCER_PATH']
)
```

## Adding Datasets to a Catalog

Obviously, before you can reference a dataset, you need to add it to the catalog.

In the previous example we had a dataset that is backed by Azure Storage Account, so the first thing we need to do is add this to the Viaduct catalog as a store.

We do that by creating a definition file that looks like this:

```
{
    type: 'azure-storage',
    config: {
        accountName: 'sports',
        key: 'rglhUbRG<snip>YWTU1DbR6pFy3EQ=='
    }
}
```

Every store definition file has the type of storage, in this case Azure Storage, and any configuration that backing storage requires.

Adding this storage account to the Viaduct catalog is then as easy as:

```
$ viaduct store create sports -f sports.json
```

With our backing Azure Storage added, we can now add our dataset. We do that in an analogous manner to the store, defining the dataset in JSON and then adding it.

```
{
    store: 'sports',
    config: {
        container: 'soccer',
        path: 'captures'
    }
}
```

```
$ viaduct dataset create soccer -f soccer.json
```

## Tagging Datasets

Most data pipelines involve multiple teams and systems working together in concert to produce an end result.

Let's say in the previous example that our soccer training data is being collected and labeled by a data engineering seperate from the machine learning team that is experimenting with different algorithms to best model the data.

Let's also say, that the data engineering team is continually refining the data to add more training data and keeps versioned copies of it and the machine learning team always wants to train their models off the latest version of this data.

One way we could do that is to set up a process to continously communicate to the second team that there is an updated dataset using a manual process -- but that is brittle, requires orchestration, and prone to human failure.

A second approach would be to just replace the entire dataset with the updated one such that the downstream system can address it the same -- but that doesn't allow you to maintain immutable versions of the data such that you can later audit what data was used to train the model.

A third approach is to introduce indirection between the addressing of a dataset and the actual dataset. In this model, when the data engineering team produces a new version of a dataset, they tag it with a timestamp, and then update a second tag, 'latest' to point at this new version.

The machine learning team can then always address the dataset using the latest tag and be assured they are using the latest training dataset.

In Viaduct, you can accomplish this with tagged datasets. Viaduct is not opinionated on how versions of the dataset are physically arranged in storage, instead we express this in the definition of the dataset. For our soccer data example above, we do a one time adjustment our definition of our logical dataset to this:

```
{
    store: 'sports',
    config: {
        container: 'soccer',
        path: 'captures/{{tag}}'
    }
}
```

As our data engineering team adds versions of the dataset, they then add a tag for the latest dataset:

```
$ viaduct tag soccer set 20180712T00000 20180712T00000
$ viaduct tag soccer set latest 20180712T00000
```

Then, in the data scientist notebook example above, we can use the latest tag to always select the latest version of the dataset.

```
$ viaduct dataset loadenv soccer:latest --catalog cse-ml
$ jupyter notebook
...
```
