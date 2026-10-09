# German Wind Generation Forecasting

This project builds an hourly model of German wind power generation from public generation and weather data. It tests how much a machine learning model improves on simple baselines, and it checks where the model falls short.

## Question

Can wind speed at a few locations, plus recent generation levels, predict hourly German wind output better than a naive forecast, when the model is tested on later years than it was trained on?

## Data

- Generation: Open Power System Data, time series package, 60 minute single index file. I used the column DE_wind_generation_actual.
- Weather: Open-Meteo Historical Weather API (ERA5 reanalysis). I used hourly wind speed at 100 m, in m/s and UTC, at three points: North Sea coast (54.0, 8.0), Baltic coast (54.3, 12.0) and inland north (52.5, 9.5).
- Period: 1 January 2015 to 30 September 2020, which gives 50,400 hourly rows.

Please credit both sources if you reuse this work, and check each source's licence terms. Open-Meteo data is licensed under CC BY 4.0.

## Method

1. Data preparation. I joined generation and weather on a UTC hourly index and checked for gaps, zeros and negative values. Generation had 74 missing hours in four gaps: one of 2 hours and three of 24 hours. I filled the 2 hour gap by linear interpolation and left the 24 hour gaps empty, so those hours were excluded.
2. Exploration. I looked at mean generation by month and hour, and at generation against wind speed. Correlation with generation was 0.86 for the inland north point, 0.81 for the North Sea and 0.74 for the Baltic.
3. Features. I built 16 features: wind speed at the three points, each lagged by 1 and 24 hours, generation lagged by 24 and 168 hours, a 7 day and a 30 day trailing mean of generation (both shifted by 24 hours), and hour, month and day of week. Rows with missing values after lagging were removed (384 rows, 0.76%, before adding the rolling features).
4. Split by time. Training uses 2015 to 2018. Testing uses January 2019 to September 2020 (15,336 hours). The data was never shuffled.
5. Baselines. Same hour yesterday, and the average of the same hour over the previous seven days.
6. Models. Random Forest and gradient boosting (scikit-learn HistGradientBoostingRegressor, with early stopping off so the training data stays in time order). I then added the rolling generation features to the gradient boosting model.

## Results

Mean generation in the test period is 14,152 MW. Mean error is actual minus predicted, so a positive value means the model predicts too low on average.

| Model | MAE (MW) | RMSE (MW) | Mean error (MW) | MAE as % of mean |
|---|---|---|---|---|
| Same hour yesterday | 6,890 | 9,174 | -8 | 48.7 |
| 7 day same hour average | 7,220 | 9,269 | -57 | 51.0 |
| Random Forest | 3,081 | 4,233 | 2,354 | 21.8 |
| Gradient boosting | 2,871 | 3,920 | 1,986 | 20.3 |
| Gradient boosting with rolling level features | 2,632 | 3,560 | 1,478 | 18.6 |

The final model has 61.8% lower MAE than the best baseline.

## Findings and limits

- The models under-predict in the test years. For gradient boosting without the rolling features, mean error was 1,774 MW in 2019 and 2,269 MW in 2020. Mean generation in the training years rose from 8,710 MW in 2015 to 12,393 MW in 2018, and the test years averaged about 14,150 MW. This is consistent with growth in installed wind capacity, but the data cannot rule out windier weather as a factor.
- The rolling level features cut the mean error by about a quarter but did not remove it. The final model still under-predicts by about 10% of mean test generation.
- The Random Forest and the first gradient boosting model predicted lower values than the highest actual output (largest prediction 40,935 MW and 40,506 MW against a test maximum of 46,064 MW).
- The weather inputs are observed reanalysis values, not forecasts available a day ahead. Scores here are better than a real day-ahead setup would give. The generation lags and rolling means use only data known 24 hours earlier.
- Three weather points do not cover the whole country, and the model has no data on installed capacity.

## How to run

1. Install the packages: pandas, numpy, matplotlib, scikit-learn and requests.
2. Open the notebook and run the cells in order. The notebook downloads the generation file and calls the Open-Meteo API, so no keys or accounts are needed.

## Possible next steps

- Use archived weather forecasts instead of reanalysis for a true day-ahead test.
- Add installed capacity as a feature.
- Add more weather points and variables such as air density.
