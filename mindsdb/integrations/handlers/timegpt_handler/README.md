---
title: Nixtla's TimeGPT Integration with MindsDB
sidebarTitle: TimeGPT
---

TimeGPT by Nixtla is a generative pre-trained model specifically designed for predicting time series data. TimeGPT takes time series data as input and produces forecasted outputs. TimeGPT can be effectively employed in various applications, including demand forecasting, anomaly detection, financial prediction, and more.

You can learn more about its features [here](https://nixtla.github.io/nixtla/).

## How to bring TimeGPT Models to MindsDB

Before creating a model, you will need to create an ML engine for TimeGPT using the `CREATE ML_ENGINE` statement and providing the TimeGPT API key:

```sql
CREATE ML_ENGINE timegpt
FROM timegpt
USING
    timegpt_api_key = 'timegpt_api_key';
```

Once the ML engine is created, we use the `CREATE MODEL` statement to create the TimeGPT model in MindsDB.

```sql
CREATE MODEL model_name
FROM data_source
  (SELECT * FROM table_name)
PREDICT column_to_be_predicted
GROUP BY column_name, column_name, ...
ORDER BY date_column
HORIZON 3 -- model forecasts the next 3 rows
USING ENGINE = 'timegpt';
```

To ensure that the model is created based on the TimeGPT engine, include the `USING` clause at the end, which defines the `engine` and lists all parameters used with time-series models, including `GROUP BY`, `ORDER BY`, `HORIZON`.

What's different about the TimeGPT engine is that it does not expose the `WINDOW` parameter in its API, so as a user you need to send a payload with at least N rows, where N depends on the model and the frequency of the series. This is automatically handled by MindsDB in the [TimeGPT handler code](https://github.com/mindsdb/mindsdb/tree/main/mindsdb/integrations/handlers/timegpt_handler).

## Example

Nixtla's TimeGPT model can be used to obtain real-time forecasts of the trading data from Binance.

<Tip>
Follow [this link](https://www.youtube.com/watch?v=8LfpFocdyEo&list=PLq3sJIV6w5BoHJ9gFSedwtb_pqk--4K89&index=3) to watch a video on integrating TimeGPT model with Binance data.
</Tip>

First, connect to Binance from MindsDB executing this command:

```sql
CREATE DATABASE my_binance
WITH ENGINE = 'binance';
```

Please note that before using the TimeGPT engine, you should create it from the MindsDB editor, or other clients through which you interact with MindsDB, with the below command:

```sql
CREATE ML_ENGINE timegpt
FROM timegpt
USING
    timegpt_api_key = 'timegpt_api_key';
```

You can check the available engines with this command:

```sql
SHOW ML_ENGINES;
```

If you see the TimeGPT engine on the list, you are ready to follow the tutorials.

Now let's create a TimeGPT model and train it with data from Binance.

```sql
CREATE MODEL cryptocurrency_forecast_model
FROM my_binance
  (
    SELECT *
    FROM aggregated_trade_data
    WHERE symbol = 'BTCUSDT'
  )
PREDICT open_price
ORDER BY open_time
HORIZON 10
USING ENGINE = 'timegpt';
```

Use the `CREATE MODEL` statement to create, train, and deploy a model. The `FROM` clause defines the training data used to train the model - here, the latest Binance data is used. The `PREDICT` clause specifies the column to be predicted - here, the open price of the BTC/USDT trading pair is to be forecasted.

As it is a time-series model, you should order the data by a date column - here, it is the open time when the open price takes effect. Finally, the `HORIZON` clause defines how many rows into the future the model will forecast - here, it forecasts the next 10 rows (the next 10 minutes, as the interval between Binance data rows is one minute).

<Warning>
Please note that the TimeGPT engine is sensitive to inconsistent intervals between data rows. Please check your data for missing, duplicated or irregular timestamps to mitigate errors that may arise if the intervals between data rows are inconsistent.

In this example, the intervals between Binance data rows are consistently equal to one minute.
</Warning>

Before proceeding, make sure that the model status reads `complete`.

```sql
DESCRIBE cryptocurrency_forecast_model;
```

To make forecasts, you must save the Binance data into a view:

```sql
CREATE VIEW btcusdt_recent AS (
  SELECT *
  FROM my_binance.aggregated_trade_data
  WHERE symbol = 'BTCUSDT'
);
```

This view is going to be joined with the model to get forecasts:

```sql
SELECT m.open_time ,
       m.open_price
FROM btcusdt_recent AS d
JOIN cryptocurrency_forecast_model AS m
WHERE d.open_time > LATEST;
```
