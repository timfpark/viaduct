# Viaduct

Viaduct makes managing datasets easier for the developer. It enables you to replace code that looks like this:

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

With code generalized to with a dataset specified via [12 factor engineering standards](https://12factor.net/) like this:

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

Viaduct accomplishes this by storing these configuration details in a central catalog:

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

Let's say our data science team wants to use the `soccer` dataset with a jupyter notebook. Instead of having hand copy credentials painfully and insecurely into the source code or managing brittle configuration details in local environmental variables, we can do this:

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

Perhaps obviously, you need to add a dataset to the catalog before you can reference it.

In the previous example we had a dataset that is backed by Azure Storage Account, so the first thing we need to do is add this store to the Viaduct catalog with a definition file like this:

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

Let's say in the previous example that our soccer training data is being collected and labeled by a data engineering team seperate from a machine learning team that is experimenting with different algorithms to best model the data.

Let's also say, that the data engineering team is continually adding more labeled training data, keeps versioned copies of the dataset, and that the machine learning team always wants to train their models off the latest version of this data.

One way we could do that is to continously communicate to the machine learning team that there is an updated dataset using a manual process -- but that is brittle, requires orchestration, and prone to human failure.

A second approach would be to just replace the entire dataset with the updated one such that the downstream system can address it in exactly the same manner -- but that doesn't allow you to maintain immutable versions of the data such that you can later audit what data was used to train the model.

A third approach, and the one that Viaduct provides, is to introduce indirection between the addressing of a dataset and the physical location of the dataset. In this approach, when the data engineering team produces a new version of a dataset, they tag it with a timestamp, and then update a second tag, 'latest' to point at this new version.

The machine learning team can then always address the dataset using the latest tag, and be assured they are using the latest training dataset, while still preserving the dataset used for training if it is necessary to refer to it later to understand an issue with the model in production.

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

Our data engineering team can then produce new immutable versions of the dataset, say at captures/2018-07-12 and then add a tag for the latest dataset that points to it with:

```
$ viaduct tag soccer set 2018-07-12 2018-07-12
$ viaduct tag soccer set latest 2018-07-12
```

Then, in the data scientist notebook example above, we can use the latest tag to always select the latest version of the dataset.

```
$ viaduct dataset loadenv soccer:latest --catalog cse-ml
$ jupyter notebook
...
```
