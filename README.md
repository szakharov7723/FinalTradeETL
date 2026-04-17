# Financial-Data-ETL-Pipeline-Python-SQL
## Cryptocurrency ETL Pipeline for Price Prediction
This project implements an end-to-end data pipeline for collecting, processing, and storing cryptocurrency data (Bitcoin Cash) for downstream analytics, visualization, and machine learning forecasting.
## Project Goal
The goal of this project is to design and implement a scalable data pipeline that aggregates cryptocurrency market data, news, and social sentiment into a centralized storage system (Azure Cosmos DB) for downstream analytics and predictive modeling.




## Architecture

Data sources:
- RSS feeds (crypto news)
- Reddit API (community sentiment)
- Real-time price API

Pipeline:
RSS / Reddit / Price APIs  
        ↓  
Data Extraction (Python scripts)  
        ↓  
Azure Cosmos DB (NoSQL)  
        ↓  
Data Transformation  
        ↓  
Analytics / Machine Learning

## Pipeline Execution Strategy

This solution is designed as a distributed ETL system.

Each data source is processed by an independent ingestion service:

- [RSS parser (news)](https://github.com/szakharov7723/Googlenews_RSS_parser)  
- [Reddit parser (sentiment)](https://github.com/szakharov7723/Reddit_parser)  
- [Price collector (real-time data)](https://github.com/szakharov7723/Bitcoin_cash-price-tick)  

Key characteristics:

- Modular execution (each service runs independently)  
- Incremental data ingestion  
- Near real-time processing (price updates every 5 minutes)  

In production, orchestration can be implemented using Azure Data Factory or Apache Airflow.

## Tech Stack

- Python (data ingestion & processing)
- Azure Cosmos DB (NoSQL storage)
- REST APIs (data sources)
- Pandas (data transformation)
- Scheduling (time-based ingestion)

## First let's look at our data sources and how they are getting built and updated

### Cosmos DB (NoSQL)
1. BCHSentimentDatabase —contains 2 containers one for RSS data and another for Reddit data
    1. RSS data for news analysis is parsed and loaded to RSSparsedBCHnews table  every day you can view [data collection script here](https://github.com/szakharov7723/Googlenews_RSS_parser/blob/main/NewsParse%26Load.ipynb) 
    2. Reddit data for community perception is parsed and loaded to APIparsedBCHreddit table every day you can view [data collection script here](https://github.com/szakharov7723/Reddit_parser/blob/main/ForumParse%26Load.ipynb) please note currently PushshiftAPI is under development, so the results are unstable. For this document we will assume the data still comes in.
2. BCHrealtimePrice — contains 1 container for real time price tick it triggers [data collection script](https://github.com/szakharov7723/Bitcoin_cash-price-tick/blob/main/BCH_price_streaming.ipynb) every 5 minutes to collect data almost real time. While in our curent project we won't analyze data by minutes this data can be used for various data science tasks to create real-time prediction models.


Current limitations:
- Limited number of external data sources
- No integration with Twitter / financial APIs

Future improvements:
- Add social media sentiment (Twitter, etc.)
- Integrate financial indicators
  

### Calendar

This is simple calendar data which so far consists of only date and datetime columns. We will need this table for incremental data loading

I created it with the following script
```

CREATE TABLE [Calendar]
(
    [CalendarDate] DATE,
	[CalendarDatetime] DATETIME
)

DECLARE @StartDate DATETIME
DECLARE @EndDate DATETIME
SET @StartDate = DATEADD(YEAR, -1, GETDATE())
SET @EndDate = DATEADD(d, 730, @StartDate)

WHILE @StartDate <= @EndDate
      BEGIN
             INSERT INTO [Calendar]
             (
                   CalendarDate,
			 CalendarDatetime
             )
             SELECT
                   @StartDate, @StartDate

             SET @StartDate = DATEADD(dd, 1, @StartDate)
      END
```

## Now we shall review our data pipeline
This is how pipeline looks like 

![alt text](https://github.com/szakharov7723/FinalTradeETL/blob/main/ETL.PNG "ETL pipeline")
 
 First we get daily data from Reddit and RSS, then we run compiling data flow
 
 The data flow looks like this
 
 ![alt text](https://github.com/szakharov7723/FinalTradeETL/blob/main/ETL_data_flow.PNG "ETL data flow")
 

 Calendar table has the following query for daily incremental load

```
SELECT  * FROM [dbo].[Calendar]
WHERE  CalendarDate = convert(date, getdate(), 1)
```

RSS data has the following query for data in case we decide to store variant data in our NoSQL, we also avoid null data, while it is impotant to keep track of nulls in our data, this is not a part of our task  

```
SELECT RSSparsedBCHnews.date,
RSSparsedBCHnews["sentiment score"] 

FROM RSSparsedBCHnews
WHERE RSSparsedBCHnews.date  != null
```
This is the query for Reddit data using the same logic as for RSS data

```
SELECT APIparsedBCHnews.Date,
APIparsedBCHnews["title sentiment score"],
APIparsedBCHnews["body sentiment score"]
FROM APIparsedBCHnews
WHERE APIparsedBCHnews.Date  != null
```

And this query is for Price
```
SELECT TickpriceAPI_BCH.timestamp,
TickpriceAPI_BCH.last_trade_price
FROM TickpriceAPI_BCH
WHERE TickpriceAPI_BCH.timestamp  != null
```
Since we store data in NoSQL , the data will be of string type, so the next step will be changing it to date format

![alt text](https://github.com/szakharov7723/FinalTradeETL/blob/main/RSSscore.PNG "RSS formatting")

We will do the same transformation for RSS and price data source


Then we aggregate data for daily statistics

![alt text](https://github.com/szakharov7723/FinalTradeETL/blob/main/Aggregatetransform.PNG "Reddit Aggregate")



We do the same transformation for Reddit and price data source.

After mentioned transformations, next data sources are joint into single staging table

![alt text](https://github.com/szakharov7723/FinalTradeETL/blob/main/Joins.PNG "Final join")

And the final transformation will be getting our final sentiment column by the following expression
```
({body sentiment score}+{title sentiment score}+{sentiment score})/3
```

And then we feed our data to SQL table
![image](https://user-images.githubusercontent.com/59535392/109377445-6b45d480-7899-11eb-8d64-456c9f811c25.png)

In order to avoid duplicates, and make sure our pipeline loads data as intended we can run the following check and replace SELECT with DELETE, if duplicates are found
```
SELECT
    FROM [dbo].[Finaldata]
    WHERE ID NOT IN
    (
        SELECT MAX(Id)
        FROM [dbo].[Finaldata]
        GROUP BY [TradeDate], 
                 [BCHprice], 
                 [FinalSentiment]
    );
```
### Azure Machine learning flow
After we have collected all data we are now ready to feed it into our [machine learning model](https://github.com/szakharov7723/AzureMLdataops/blob/main/AzureMLcryptofcst.ipynb)  and forecasting report
Since we have pretty few data for high quality model training,  we retrain and apply best model every day to find the best forecasting solution.

In short:
* We use Service principal authentification to keep it running daily
* We created a link to our SQL datastore and use this data for training
* We train our data for forecasting models and use *normalized root mean squared error* as a primary metric
* we forecast daily for 2 weeks and register time series columnn
* We use remote compute cluster to train our model and find the best one
* We create a temporary dataframe consisting of forecast price and date column generateed for 14 days starting from today
* Later we use pyodbc lib to connect to our SQL database, where we erase previous forecast data and load fresh forecast for next 14 days.



### Create Forecasting report in Tableau  

Then we connect Tableau to our SQL server and 14 day Forecating table
![image](https://github.com/szakharov7723/FinalTradeETL/blob/main/Forecast_report.PNG "Forecast report")


## How to Run

### Prerequisites

Make sure you have the following installed:

- Python 3.9+
- pip (Python package manager)
- Access to Azure Cosmos DB (or configured connection string in project files)

```bash

### 1. Clone repositories


git clone https://github.com/szakharov7723/Googlenews_RSS_parser
git clone https://github.com/szakharov7723/Reddit_parser
git clone https://github.com/szakharov7723/Bitcoin_cash-price-tick
git clone https://github.com/szakharov7723/AzureMLdataops

### 2. Configure environment

:'For each project:
Install dependencies
Set up environment variables (API keys, Cosmos DB connection string)
'
## Example:
cd Googlenews_RSS_parser
pip install -r requirements.txt

## Repeat for all repositories.

### 3. Run ingestion services

## Run each data ingestion service separately:

#RSS News
cd Googlenews_RSS_parser
python main.py

#Reddit Data
cd Reddit_parser
python main.py

#Price Data (real-time)
cd Bitcoin_cash-price-tick
python main.py

### 4. Run data processing / ML
cd AzureMLdataops
python main.py

### 5. Expected Results
:'After running the pipeline:

Data is stored in Azure Cosmos DB
You will have:
Cryptocurrency price data
News data (RSS)
Reddit sentiment data
Data is ready for:
Analysis
Visualization
Machine learning models
'

### This pipeline runs every day
```
 ### Example Output

The pipeline produces structured datasets combining:

- Timestamp
- Asset price
- News signals
- Sentiment indicators

These datasets can be used for:
- trend analysis
- correlation studies
- feature engineering for ML models
- start-ups





