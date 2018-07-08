# Bergen

Bergen makes manage data pipeline configuration easier and cleaner. It enables you to replace code that looks like this:

```
sports_blob_service =  BlockBlobService(
    account_name='sports',
    account_key='rgFEEDDEADBEEFAPwbfxgdFSW6vKQEAarHua2HSFYWTU1DbR6pFy3EQ=='
)

byte_data = sports_blob_service.get_blob_to_bytes(
    'livedata',
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

byte_data = futbol_blob_service.get_blob_to_bytes(
    os.environ['CSE_ML_SOCCER_CONTAINER'],
    os.environ['CSE_ML_SOCCER_PATH']
)
```

Bergen accomplishes this by storing configuration details in a centrally managed catalog:

```
$ bergen catalog add https://cse-ml-catalog/
$ bergen catalogs list

CATALOG         DESCRIPTION
cse-ml          CSE ML Group
nyu             NYU's canonical ML datasets
```

It also makes it easier to create, discover, and consume datasets. For example, we can find the dataset we need for the previous data science project using:

```
$ bergen datasets list
CATALOG     DATASET         DESCRIPTION
...
cse-ml      soccer          Soccer sports performance captures.
nyu         minst-images    MINST images
nyu         minst-labels    MINST image labels
```

Let's say we want to use the `futbol` dataset with a jupyter notebook to do some data science work on it. Instead of having hand copy credentials painfully and insecurely into the source code or managing brittle configuration details in local environmental variables, we can do this:

```
$ bergen dataset loadenv soccer --catalog cse-ml
$ jupyter notebook
```

Under the covers, the `bergen` tool fetches the `futbol` dataset connection details from the catalog at `https://cse-ml-catalog` and populates environment variables with all of the configuration and authentication details you need to consume the dataset:

```
$ printenv | grep 'CSE_ML_FUTBOL'
CSE_ML_SPORTS_ACCOUNTNAME=futbol
CSE_ML_SPORTS_KEY=rglhUbRG/557nmOGwdzFEEDDEADBEEFAPwbfxgdxndzuvpk6nFSW6vKQEAarHua2HSFYWTU1DbR6pFy3EQ==
CSE_ML_SOCCER_CONTAINER=soccer
CSE_ML_SOCCER_PATH=livedata
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

In the previous example we had a dataset that is backed by Azure Storage Account, so the first thing we need to do is add this to the Bergen catalog as a store.

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

Adding this storage account to the Bergen catalog is then as easy as:

```
$ bergen store create sports -f sports.json
```

With our backing Azure Storage added, we can now add our dataset. We do that in an analogous manner to the store, defining the dataset in JSON and then adding it.

```
{
    store: 'sports',
    config: {
        container: 'soccer',
        path: 'livedata'
    }
}
```

```
$ bergen dataset create soccer -f soccer.json
```
