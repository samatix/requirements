# Benchmark Revamping

## Requirements 

* The benchmark data retrieval should be separated from zipline 
* The benchmark definition should enable the usage of different benchmarks \(Indexes, ETFs, different geographies\) 
* No performance regression 

## Solution

The option to make the benchmark optional is coherent with the fact that zipline is a specialised backtester \(it won't impact your need at Quantopian to see the performance results in realtime\). If the end user wants to analyse his strategy's performance, he can use Alphalens \(compare the returns to the returns from a specific benchmark instrument or factor\). 

I can do the following: 

1. For tomorrow evening, I can write a simple WA to show the end users how to download the data from IEX and add them in Zipline without the need to amend the benchmark.py by manually saving the returns in zipline data root folder via simple pd transformations. I'll push this change to the readme file after it's accepted
2. For this weekend, I can add an option to zipline to ask the end users to provide the benchmark closing prices in Json/CSV \(in OHLCV\) format with the path to the file. We can provide the end users with sample code on how to download the data from IEX to match the format required as a way to download the data 

### Backtesting with Benchmark Data in CSV/JSON 

* Add a main command to use a custom benchmark file that contains at least \(datetime, close\)

```python
@main.command()
@click.option(
    '-bf',
    '--benchmarkfile',
    default=None,
    type=click.File('r'),
    help='The csv file that contains the benchmark closing prices',
)
```

* Allow as well to use a benchmark from the ingested data via its SID \(or equivalent\)
* If the benchmark is not given, try to retrieve the default data already saved in data\_root that has been saved. 
* If the benchmarks are empty or forced to None move to the next section 

### Backtesting without Benchmark

* Pass a zero dataframe to not change zipline's core functionalities 
* Raise a warning on the metrics valuation 

## Validation Steps

1. Test with a strategy that buys and holds SPY until today \(and until one week ago\) with the benchmark SPY **-&gt;** Validate the metrics values 
2. Test with a strategy that buys AAPL until today with the benchmark SPY until today \(and until one week ago\) **-&gt;** Validate the metrics values 
3. Test with a strategy that buys SX5E until today with the benchmark SX5E until today \(and until one week ago\) **-&gt;** Validate the metrics values 
4. Test with a strategy that buys and holds FP until today with the benchmark SX5E \(different data bundle as different calendar\) **-&gt;** Validate the metrics values 

