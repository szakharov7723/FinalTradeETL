# FinalTradeETL
## The object of this project
This project shows a possible data architecture for price analysis and prediction. It uses cryptocurrency as an example, but it can be also applied to various forecasting solutions involving different data sources. 
## The task of this project
The task of this project is to collect market data for bitcoin cash cryptocurrency and structure it in one source for forecasting machine learning and visualization.





## First let's look at our data sources and how they are getting built and updated

### Cosmos DB (NoSQL)
1. BCHSentimentDatabase —contains 2 containers one for RSS data and another for Reddit data
    1. RSS data for news analysis is parsed and loaded to RSSparsedBCHnews table  every day you can view [data collection script here](https://github.com/szakharov7723/Googlenews_RSS_parser/blob/main/NewsParse%26Load.ipynb) 
    2. Reddit data for community perception is parsed and loaded to APIparsedBCHreddit table every day you can view [data collection script here](https://github.com/szakharov7723/Reddit_parser/blob/main/ForumParse%26Load.ipynb) please note currently PushshiftAPI is under development, so the results are unstable. For this document we will assume the data still comes in.
2. BCHrealtimePrice — contains 1 container for real time price tick it triggers [data collection script](https://github.com/szakharov7723/Bitcoin_cash-price-tick/blob/main/BCH_price_streaming.ipynb) every 5 minutes to collect data almost real time. While in our curent project we won't analyze data by minutes this data can be used for various data science tasks to create real-time prediction models.


While I do realize this is very small vartiety of data source for price analysis and opportunities for prediction, I don't have that much free credits to parse and store data from Twitter, Facebook, influencial financial publishers and other influencers and crypto metrics data sources.


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




### This pipeline runs every day
