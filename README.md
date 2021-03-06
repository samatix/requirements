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

## Validation

### Configuration

Let us consider two instruments **A** \(a\) and instrument **B** \(b\) to be used as a benchmark : 

#### Data file for the instrument A a.csv:

```text
date,open,high,low,close,adj_close,volume
2020-01-02 00:00:00+00:00,100,100,100,100,100,10000
2020-01-03 00:00:00+00:00,120,120,120,120,120,12000
2020-01-06 00:00:00+00:00,100,100,100,100,100,10000
2020-01-07 00:00:00+00:00,160,160,160,160,160,16000
2020-01-08 00:00:00+00:00,180,180,180,180,180,18000
2020-01-09 00:00:00+00:00,200,200,200,200,200,20000
```

**Data file for the benchmark B b.csv:**

```python
date,open,high,low,close,adj_close,volume
2020-01-02 00:00:00+00:00,100,100,100,100,100,10000
2020-01-03 00:00:00+00:00,90,90,90,90,90,9000
2020-01-06 00:00:00+00:00,120,120,120,120,120,10000
2020-01-07 00:00:00+00:00,140,140,140,140,140,14000
2020-01-08 00:00:00+00:00,160,160,160,160,160,16000
2020-01-09 00:00:00+00:00,180,180,180,1180,180,18000
```

#### Ingestion Configuration

```python
import pandas as pd

from zipline.data.bundles import register
from zipline.data.bundles.csvdir import csvdir_equities

start_session2 = pd.Timestamp('2020-01-02', tz='utc')
end_session2 = pd.Timestamp('2020-01-09', tz='utc')

register(
    'csv-xpar-sample',
    csvdir_equities(
        ['daily'],
        '/Users/aennassiri/opensource/zipline',
    ),
    calendar_name='XPAR',
    start_session=start_session2,
    end_session=end_session2
)
```

#### Sample Algorithm

The algorithm is supposed to order at the beginning of the backtesting period 1000 **A** stocks 

```python
from zipline.api import order, symbol
from zipline.finance import commission, slippage


def initialize(context):
    context.stocks = symbol('a')
    context.has_ordered = False
    context.set_commission(commission.NoCommission())
    context.set_slippage(slippage.NoSlippage())


def handle_data(context, data):
    if not context.has_ordered:
        order(context.stocks, 1000)
        context.has_ordered = True
```

###  Tests

#### Run without any benchmark option 

**Command:**

```python
run -f TEST_FOLDER/test_benchmark3.py -b csv-xpar-sample -s 01/01/2020 -e 01/09/2020
```

**Result** 

```bash
Warning: Neither a benchmark file nor a benchmark symbol is provided. Trying to use the default benchmark loader. To use zero as a benchmark, use the flag --no-benchmark
...
ValueError: Please set your IEX_API_KEY environment variable and retry.
Please note that this feature will be deprecated
```

**Comment**

If no benchmark setting is used, we use the default benchmark data loader after raising a warning. The latter checks if data already exists in the zipline data folder, if not it tries to download it from IEX as before. I've made a change in the code to enable users who set an environment variable with the IEX\_API\_KEY code. 

This option is kept as to not break the backward compatibility for users who put directly the benchmark data in the zipline data folder. 

**TODO:**

I'm going to update the error message: 

* Advise the end user to prefer using the explicit benchmark options provided 
* Tell the end user that it is possible to directly put the benchmark data in the data folder
* Inform the end user that it is possible to download the data from IEX, if the env variable IEX\_API\_KEY code though warn him that the benchmark SPY is going to be used 

#### Run with benchmark file option 

**Command:**

```python
run -f TEST_FOLDER/test_benchmark3.py -b csv-xpar-sample -s 01/01/2020 -e 01/09/2020 --benchmark-file TEST_FOLDER/data/daily/b.csv --trading-calendar XPAR
```

**Result** 

```bash
[2020-02-07 10:19:55.904112] INFO: zipline.finance.metrics.tracker: Simulated 6 trading days
first open: 2020-01-02 08:01:00+00:00
last close: 2020-01-09 16:30:00+00:00
                           algo_volatility  algorithm_period_return     alpha  \
2020-01-02 16:30:00+00:00              NaN                    0.000       NaN   
2020-01-03 16:30:00+00:00         0.000000                    0.000  0.000000   
2020-01-06 16:30:00+00:00         0.018330                   -0.002 -0.070705   
2020-01-07 16:30:00+00:00         0.055083                    0.004  0.268001   
2020-01-08 16:30:00+00:00         0.048217                    0.006  0.310527   
2020-01-09 16:30:00+00:00         0.043427                    0.008  0.299573   
...
```

Order at trading day 2 : 

```python
[{'price': 120.0, 'amount': 1000, 'dt': Timestamp('2020-01-03 16:30:00+0000', tz='UTC'), 'sid': Equity(0 [A]), 'order_id': '8b3d018994cf43db960e2943b59f7ef0', 'commission': None}]
```

| File | algo\_volatility | algorithm\_period\_return | alpha | benchmark\_period\_return | benchmark\_volatility | beta | capital\_used |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 2020-01-02 16:30:00+00:00 |  | 0.0 |  | 0.0 |  |  | 0.0 |
| 2020-01-03 16:30:00+00:00 | 0.0 | 0.0 | 0.0 | -0.09999999999999998 | 1.1224972160321822 | 0.0 | -120000.0 |
| 2020-01-06 16:30:00+00:00 | 0.018330302779823376 | -0.0020000000000000018 | -0.07070503597122309 | 0.19999999999999996 | 3.6018513757973594 | -0.004964028776978422 | 0.0 |
| 2020-01-07 16:30:00+00:00 | 0.05508274964812368 | 0.0040000000000000036 | 0.26800057257371374 | 0.40000000000000013 | 3.0243456592570017 | -0.0006048832358595375 | 0.0 |
| 2020-01-08 16:30:00+00:00 | 0.048217028714915476 | 0.006000000000000005 | 0.3105268451522164 | 0.6000000000000001 | 2.6367729194171097 | -0.0002895623813476541 | 0.0 |
| 2020-01-09 16:30:00+00:00 | 0.043427366958536835 | 0.008000000000000007 | 0.34104117177313514 | 0.8 | 2.3608034346685574 | -0.0001915086325657816 | 0.0 |

**Comment**

The benchmark data from the provided file is correctly loaded. 

#### Run with benchmark symbol option 

**Command:**

```python
run -f TEST_FOLDER/test_benchmark3.py -b csv-xpar-sample -s 01/01/2020 -e 01/09/2020 --benchmark-symbol b --trading-calendar XPAR
```

**Result** 

```bash
[2020-02-07 10:28:00.235496] INFO: zipline.finance.metrics.tracker: Simulated 6 trading days
first open: 2020-01-02 08:01:00+00:00
last close: 2020-01-09 16:30:00+00:00
                           algo_volatility  algorithm_period_return     alpha  \
2020-01-02 16:30:00+00:00              NaN                    0.000       NaN   
2020-01-03 16:30:00+00:00         0.000000                    0.000  0.000000   
2020-01-06 16:30:00+00:00         0.018330                   -0.002 -0.070705   
2020-01-07 16:30:00+00:00         0.055083                    0.004  0.268001   
2020-01-08 16:30:00+00:00         0.048217                    0.006  0.310527   
2020-01-09 16:30:00+00:00         0.043427                    0.008  0.341041   
```

Order at trading day 2 : 

```python
[{'amount': 1000, 'sid': Equity(0 [A]), 'dt': Timestamp('2020-01-03 16:30:00+0000', tz='UTC'), 'price': 120.0, 'order_id': '18d3e8ab70be4cf392b2f8e044e3680d', 'commission': None}]
```

| Symbol | algo\_volatility | algorithm\_period\_return | alpha | benchmark\_period\_return | benchmark\_volatility | beta | capital\_used |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 2020-01-02 16:30:00+00:00 |  | 0.0 |  | 0.0 |  |  | 0.0 |
| 2020-01-03 16:30:00+00:00 | 0.0 | 0.0 | 0.0 | -0.09999999999999998 | 1.1224972160321822 | 0.0 | -120000.0 |
| 2020-01-06 16:30:00+00:00 | 0.018330302779823376 | -0.0020000000000000018 | -0.07070503597122309 | 0.19999999999999996 | 3.6018513757973594 | -0.004964028776978422 | 0.0 |
| 2020-01-07 16:30:00+00:00 | 0.05508274964812368 | 0.0040000000000000036 | 0.26800057257371374 | 0.40000000000000013 | 3.0243456592570017 | -0.0006048832358595375 | 0.0 |
| 2020-01-08 16:30:00+00:00 | 0.048217028714915476 | 0.006000000000000005 | 0.3105268451522164 | 0.6000000000000001 | 2.6367729194171097 | -0.0002895623813476541 | 0.0 |
| 2020-01-09 16:30:00+00:00 | 0.043427366958536835 | 0.008000000000000007 | 0.34104117177313514 | 0.8 | 2.3608034346685574 | -0.0001915086325657816 | 0.0 |

**Comment**

The benchmark data from the provided file is correctly loaded and matches the results from the test with benchmark\_file.

If the benchmark data symbol is not found, a warning is raised and the default loader is used as a contingency plan:

```python
/Users/aennassiri/opensource/zipline/zipline/utils/run_algo.py:116: UserWarning: Symbol c as a benchmark not found in this bundle. Proceedig with default benchmark loader
  "loader" % benchmark_symbol)
[2020-02-07 10:32:49.049269] INFO: Loader: Cache at /Users/aennassiri/.zipline/data/SPY_benchmark.csv does not have data from 1990-01-02 00:00:00+00:00 to 2020-02-07 00:00:00+00:00.

[2020-02-07 10:32:49.049469] INFO: Loader: Downloading benchmark data for 'SPY' from 1989-12-29 00:00:00+00:00 to 2020-02-07 00:00:00+00:00
```

**TODO:** 

* Correct the warning message 
* I did the validation with the Paris Calendar and wanted to check how the system behaves when the trading calendar is given in the ingested data and is different from the one used for running the algorithm.  I need to review how the system booked an order at 21:30 knowing that the ingested data is from a different calendar. This issue is independent from the benchmark validation 

#### Run with --no-benchmark option 

**Command:**

```python
run -f TEST_FOLDER/test_benchmark3.py -b csv-xpar-sample -s 01/01/2020 -e 01/09/2020 --no-benchmark
```

**Result** 

```bash
Warning: Using zero returns as a benchmark. The risk metrics that requires benchmark returns will not be calculated.
[2020-02-07 10:38:07.174387] INFO: zipline.finance.metrics.tracker: Simulated 6 trading days
first open: 2020-01-02 14:31:00+00:00
last close: 2020-01-09 21:00:00+00:00
                           algo_volatility  algorithm_period_return alpha  \
2020-01-02 21:00:00+00:00              NaN                    0.000  None   
2020-01-03 21:00:00+00:00         0.000000                    0.000  None   
2020-01-06 21:00:00+00:00         0.018330                   -0.002  None   
2020-01-07 21:00:00+00:00         0.055083                    0.004  None   
2020-01-08 21:00:00+00:00         0.048217                    0.006  None   
2020-01-09 21:00:00+00:00         0.043427                    0.008  None   

                           benchmark_period_return benchmark_volatility  beta  \
2020-01-02 21:00:00+00:00                      0.0                 None  None   
2020-01-03 21:00:00+00:00                      0.0                [0.0]  None   
2020-01-06 21:00:00+00:00                      0.0                [0.0]  None   
2020-01-07 21:00:00+00:00                      0.0                [0.0]  None   
2020-01-08 21:00:00+00:00                      0.0                [0.0]  None   
2020-01-09 21:00:00+00:00                      0.0                [0.0]  None   
```

```python
[{'price': 120.0, 'order_id': 'b607afab7c674f22a303a0a483f00a31', 'amount': 1000, 'sid': Equity(0 [A]), 'commission': None, 'dt': Timestamp('2020-01-03 21:00:00+0000', tz='UTC')}]
```

**Comment**

* A warning states that the system is using zero returns as a benchmark. 
* All the results are the same except between the last three runs except for Alpha \(None\), Beta\(None\), benchmark\_returns \(Zero\),  benchmark\_volatility\(Zero\). 

### **Appendix**

The comparison and reconciliation of the returns, volatility, alpha, beta can be found in this [sheet](https://docs.google.com/spreadsheets/d/1-Zl8fYPAH6k9dvhAUJ2iAZaKTTFm62Hybffp1t8cSfQ/edit?usp=sharing)

