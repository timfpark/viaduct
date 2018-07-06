# Bergen

Bergen makes manage data pipeline configuration easier and cleaner. It enables you to replace code that looks like this:

```
futbol_bs =  BlockBlobService(
    account_name='futbol',
    account_key='rgFEEDDEADBEEFAPwbfxgdFSW6vKQEAarHua2HSFYWTU1DbR6pFy3EQ=='
)

byte_data = futbol_bs.get_blob_to_bytes(
    'livedata',
    'sampleGameData/ball_player.csv'
)

ball_df = pd.read_csv(
    StringIO(
        byte_data.content.decode('utf-8')
    ),
    error_bad_lines=False
)
```

With code abstracted to work against different datasets and that adheres to [12 factor engineering standards](https://12factor.net/) like this:

```
import os

futbol_bs =  BlockBlobService(
    account_name=os.environ['CSE_ML_FUTBOL_ACCOUNTNAME'],
    account_key=os.environ['CSE_ML_FUTBOL_KEY']
)

byte_data = futbol_bs.get_blob_to_bytes(
    os.environ['CSE_ML_FUTBOL_CONTAINER'],
    os.environ['CSE_ML_FUTBOL_PATH']
)

ball_df = pd.read_csv(
    StringIO(
        byte_data.content.decode('utf-8')
    ),
    error_bad_lines=False
)
```

Bergen accomplishes this by storing dataset and store configuration details in a catalog that can be managed centrally:

```
$ bergen catalog add https://cse-ml-catalog/
$ bergen catalogs list

CATALOG			DESCRIPTION
cse-ml			CSE ML Group Datasets
```

It also makes it easier to create, discover, and consume datasets. For example, we can find the dataset we need for the previous data science project using:

```
$ bergen datasets list
CATALOG		DATASET			DESCRIPTION
...
cse-ml		futbol			Football sports performance captures.
nyu			minst-images	MINST images
nyu			minst-labels	MINST image labels
```

Let's say we want to use the `futbol` dataset with a jupyter notebook to do some data science work on it. Instead of having hand copy credentials painfully and insecurely into the source code or managing brittle configuration details in local environmental variables, we can do this:

```
$ bergen dataset loadenv futbol --catalog cse-ml
$ jupyter notebook
```

Under the covers, the `bergen` tool fetches the `futbol` dataset connection details from the catalog at `https://cse-ml-catalog` and populates environment variables with all of the configuration and authentication details you need to consume the dataset:

```
$ printenv | grep 'CSE_ML_FUTBOL'
CSE_ML_FUTBOL_ACCOUNTNAME=futbol
CSE_ML_FUTBOL_KEY=rglhUbRG/557nmOGwdzFEEDDEADBEEFAPwbfxgdxndzuvpk6nFSW6vKQEAarHua2HSFYWTU1DbR6pFy3EQ==
CSE_ML_FUTBOL_CONTAINER=livedata
CSE_ML_FUTBOL_PATH=sampleGameData/ball_player.csv
```

When you start `jupyter`, it automatically imports these environmental variables, so doing:

```
import os

os.environ['CSE_ML_FUTBOL_ACCOUNTNAME']
```

will yield

```
'futbol'
```

This means we can execute the generalized version of the data science code we wrote at the beginning of this and this configuration will automatically be injected from the environment variables.
