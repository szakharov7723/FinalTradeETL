# FinalTradeETL
## The task of this project
The task of this project is to collect various market data for bitcoin cash cryptocurrency and structure it in one source for furter analysis and visualization.

## First let's look at our data sources and how they are getting built and updated

### Cosmos DB (NoSQL)
  1) BCHSentimentDatabase --contains 2 containers one for RSS data and another for Reddit data
    b) RSS data for news analysis is parsed every day and loaded to RSSparsedBCHnews table you can view the code here https://github.com/szakharov7723/Googlenews_RSS_parser
    c) Reddit data for community perception is parsed and loaded to APIparsedBCHreddit every day you can view the code here https://github.com/szakharov7723/Reddit_parser plese note currently PushshiftAPI is under development, so the results are unstable. For this document we will assume the data still comes in.
  2) BCHrealtimePrice -- contains 1 container for real time price tick it triggers every 5 minutes to collect data almost real time. While in our curent project we won't analyze data by minutes this data can be used for various data science tasks to create real-time prediction models. You can view the code here https://github.com/szakharov7723/Bitcoin_cash-price-tick
 
While I do realize this is very small vartiety of data source for price analysis and opportunities for prediction, I don't have that much free credits to parse and store data from Twitter, Facebook, influencial financial publishers and other influencers and perception data sources.


### Calendar

This is simple calendar data which so far consists of only date and datetime columns with autogenerated id. We will need this table for incremental data loading


## Now we shall review our data pipeline
This is how pipeline looks like 

![alt text](https://github.com/szakharov7723/FinalTradeETL/blob/main/ETL.PNG "ETL pipeline")
 
 First we get daily data from Reddit and RSS, then we run compiling data flow
 
 The data flow looks like this
 
 ![alt text](https://github.com/szakharov7723/FinalTradeETL/blob/main/ETL_data_flow.PNG "ETL data flow")
 

 Calendar table has the following schema for daily incremental load

```
SELECT  * FROM [dbo].[Calendar]
WHERE  CalendarDate = convert(date, getdate(), 1)
```

RSS data has the following schema for data in case we decide to store variant data in our NoSQL, we also avoid null data, while it is impotant to keep track of nulls in our data, this is not a part of our task  

```
SELECT RSSparsedBCHnews.date,
RSSparsedBCHnews["sentiment score"] 

FROM RSSparsedBCHnews
WHERE RSSparsedBCHnews.date  != null
```
This is the schema for Reddit data using the same logic as for RSS data

```
SELECT APIparsedBCHnews.Date,
APIparsedBCHnews["title sentiment score"],
APIparsedBCHnews["body sentiment score"]
FROM APIparsedBCHnews
WHERE APIparsedBCHnews.Date  != null
```

And this schema is for Price
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

