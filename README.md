# Accelerometer data

The accelerometer data provided here is sourced from the paper [Gait Monitoring and Analysis: A Mathematical Approach](https://www.mdpi.com/1424-8220/23/18/7743) where experiments were conducted to collect and analyze this dataset. In the paper, detailed descriptions of the experiments are available, including the specific activities in which the data was gathered. The data originates from two distinct groups of individuals: one comprising elderly patients with chronic diseases and the other consisting of individuals diagnosed with Parkinson's disease. Patients with chronic conditions used Android smartwatches, while those with Parkinson's disease utilized wristbands equipped with BLE (Bluetooth Low Energy) technology. The participants were encouraged to use these devices for extended periods, capturing a wide range of daily activities. In a few instances, participants were instructed to wear the devices exclusively during walking activities.

The data is available at the following links:

| Project                             | Sampling frequency | Unique users | Labels | Dataset Size | Data format |                           Download                           |
| :---------------------------------- | :----------------: | :----------: | :----: | :----------: | :---------: | :----------------------------------------------------------: |
| Parkinson                           |        25Hz        |      20      |   NA   |    ~17GB     |   MongoDB   | [LINK](https://drive.google.com/drive/folders/1CnoWaaMFOacgJaCZfJkprRYTsGSBB2po?usp=sharing) |
| Chronic deaseses                    |        25Hz        |      30      |   NA   |    ~25GB     |   MongoDB   | [LINK](https://drive.google.com/drive/folders/1RtDDlfDJzx-0YLW1hm0htt2oQdeuYB5T?usp=sharing) |
| Single Parkinson user (walks only*) |        25Hz        |      1       |   NA   |    ~450MB    |     CSV     | [LINK](https://drive.google.com/drive/folders/1wfsUucekV6ix0HOGfnF-LpLpztrbCluY?usp=sharing) |
| Single Parkinson user 2 (walks only*) |        25Hz        |      1       |   NA   |    ~9MB    |     CSV     | [LINK](https://drive.google.com/drive/folders/13hxamYeQVoo21K7ex1w-pCG53DN5tnTv?usp=sharing) |

\* The data was collected during the users walks but contains some noise.

## HAR classification model training
Given the primarily unsupervised nature of the available data, we addressed the task of building a classifier to identify instances when patients were walking by leveraging the [Capture24](https://www.nature.com/articles/s41598-018-26174-1) dataset. Subsequently, we applied preprocessing techniques to align this dataset with our own and trained an XGBoost classifier model. The training procedure closely resembles the one outlined by [Walmsley et al.](https://www.medrxiv.org/content/10.1101/2020.11.10.20227769v3).

## Restoring data

1. Install MongoDB

2. Move the DB to restore to the machine

3. Execute the following command:
   ` mongorestore --drop --numParallelCollections=8 mongodb://localhost/GaitLogs -d GaitLogs --gzip all_dumps7/GaitLogs/`

   Where:

   - `--drop` is used to indicate that if you already got a that DB it will replace all the collections with the new ones.

   - `--numParallelCollections` indicates the number of parallel workers that will execute the task. Ideally it has to match the number of CPUs of the machine (given that you have enough memory).
   - `mongodb://localhost/GaitLogs` the url to the DB you want to restore.
   - `-d` indicates the DB you want to use
   - `--gzip` indicates that you will restore from a gzip file
   - `all_dumps7/GaitLogs/` is the path to the DB you want to restore

The process can take a while to complete. At the end you will have a clone of the DB locally.

## Querying the DB

In the following, an example of how to query the TED DB using python mongoDB APIs:

```python
from datetime import datetime, timedelta
from pymongo import MongoClient
import pandas as pd

DB_ted = "GaitLogs"

def get_24h_data(collection, date):
    '''Gets data from the collection from a starting date + 24h'''
    # Connect to the MongoDB server
    client = MongoClient('mongodb://localhost:27017/')

    # Select the database and collection
    db = client[DB_ted]
    collection = db[collection]

    # Define the date range (24 hours)
    start_date = datetime.combine(date, datetime.min.time())
    end_date = start_date + timedelta(days=1)
    
    # Define the query
    query = {
        'timestamp': {
            '$gte': start_date,
            '$lt': end_date
        }
    }

    # Execute the query and retrieve the results
    results = collection.find(query)

    # Execute the query and retrieve the results
    results = list(collection.find(query))
    
    return results

def get_data_range(collection, start_date, end_date):
    '''Yelds all the data from a start to an end date 1 dataframe per day'''
    # Iterate through the date range
    for date in pd.date_range(start_date, end_date):
        
        # Get the data for the current date
        cursor = get_24h_data(collection, date)

        # Convert cursor to pandas dataframe
        df = pd.DataFrame(list(cursor))
        
        yield df

        
def parser_ted(df):
    '''Parses data coming from the TED DB'''
    df = df.drop_duplicates("timestamp")
    df = df.rename(columns={"AccX [mg]": "x", "AccY [mg]": "y", "AccZ [mg]": "z"})
    df["timestamp"] = pd.to_datetime(df.timestamp)
    df = df.set_index("timestamp", drop=False)
    df = df[["timestamp", "x", "y", "z"]]

    df[["x", "y", "z"]] = df[["x", "y", "z"]] / 1000

    return df
```

A toy example of how to use this functions:

```python
from datetime import datetime, timedelta
import pandas as pd
import numpy as np

SAVE_DIR = "processed_data_ted/"
COLLECTION = "GaitLogs-Moko-pA8kkKl2PKa1InbgX0IEu0PvN603" # user 14

def process_one(collection=COLLECTION, 
                start_date=datetime(2022,8,1), 
                end_date=datetime(2022,9,1)):

    all_dfs = []
    for df in get_data_range(collection, start_date, end_date):
        if(len(df) > 0):
            df = parser_ted(df)
            all_dfs.append(df)
            #print("LEN DFS:", len(all_dfs))
        else:
            print(f"{collection}: No data")
    
    print(f"{collection} num dfs: {len(all_dfs)}")
    if(len(all_dfs) > 0):
        df_out = pd.concat(all_dfs)
        df_out = df_out.sort_values("timestamp").reset_index(drop=True)
        
        fname = os.path.join(SAVE_DIR, collection)
        df_out.to_csv(f"{fname}.csv.gz", index=False, compression="gzip")

    return len(all_dfs)
```

## Citation
If you use this data please cite our [work](https://www.mdpi.com/1424-8220/23/18/7743):

```latex
@Article{s23187743,
AUTHOR = {Canonico, Massimo and Desimoni, Francesco and Ferrero, Alberto and Grassi, Pietro Antonio and Irwin, Christopher and Campani, Daiana and Dal Molin, Alberto and Panella, Massimiliano and Magistrelli, Luca},
TITLE = {Gait Monitoring and Analysis: A Mathematical Approach},
JOURNAL = {Sensors},
VOLUME = {23},
YEAR = {2023},
NUMBER = {18},
ARTICLE-NUMBER = {7743},
URL = {https://www.mdpi.com/1424-8220/23/18/7743},
ISSN = {1424-8220},
ABSTRACT = {Gait abnormalities are common in the elderly and individuals diagnosed with Parkinson&rsquo;s, often leading to reduced mobility and increased fall risk. Monitoring and assessing gait patterns in these populations play a crucial role in understanding disease progression, early detection of motor impairments, and developing personalized rehabilitation strategies. In particular, by identifying gait irregularities at an early stage, healthcare professionals can implement timely interventions and personalized therapeutic approaches, potentially delaying the onset of severe motor symptoms and improving overall patient outcomes. In this paper, we studied older adults affected by chronic diseases and/or Parkinson&rsquo;s disease by monitoring their gait due to wearable devices that can accurately detect a person&rsquo;s movements. In our study, about 50 people were involved in the trial (20 with Parkinson&rsquo;s disease and 30 people with chronic diseases) who have worn our device for at least 6 months. During the experimentation, each device collected 25 samples from the accelerometer sensor for each second. By analyzing those data, we propose a metric for the &ldquo;gait quality&rdquo; based on the measure of entropy obtained by applying the Fourier transform.},
DOI = {10.3390/s23187743}
}
```
