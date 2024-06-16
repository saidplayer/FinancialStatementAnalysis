# [10-K financial statements: analyzing companies’ reports with SEC EDGAR API](https://drsoli.com/index.php/2023/09/18/10-k-financial-statements-analyzing-companies-reports-with-sec-edgar-api/)

Here we access and analyze financial statements, such as 10-k and 10-Q. Then we will calculate principal financial ratios: Current ratio, Leverage, P/E and Inventory turnover.

You know EDGAR, the API maintained by US SEC for accessing financial reports of
– US public companies,
– ETFs, and
– variable annuities.

EDGAR stands for “Electronic Data Gathering, Analysis, and Retrieval system” and you can easily connect to it, with any programming language to read the data you need. Here we will look for 10-K annual reports of some big corporations. The 10-K report includes
– Balance sheet,
– Income statement, and
– Cash flow statement.

*Note that EDGAR, being an American system, uses GAAP and not IFRS standard for reporting.*

Let’s start! Based on EDGAR’s API documentation, the API for company facts, is the one that returns a list of all the reports. In order to access company facts for any company, one would need its CIK (Central Index Key) code. There are different places where you can find a list of all CIK codes for all US public companies, but basically you can use EDGAR CIK lookup tool for that. Here I have an array with the ticker and CIK code of some corporations.

```Python
import requests
import json
from yfinance import download

CIK_list = [
  ["aapl",320193 ],
  ["msft",789019],
  ["xom",34088],
  ["nvda",1045810],
  ["tsla",1318605],
  ["pfe",78003],
  ["baba",1577552],
  ["csco",858877],
  ["adbe",796343],
  ["pypl",1633917],
  ["ibm",51143],
  ["nflx",1065280],
  ["ba",12927],
  ["wmt", 104169],
  ["jpm", 19617],
  ["orcl", 1341439],
  ["nke", 320187],
  ["amd", 2488],
  ["tgt", 27419],
  ["ge", 40545],
  ["googl", 1652044],
  ["amzn", 1018724],
  ["f", 37996]
 ]
```

Now let’s define a function that calls the API. Note that the CIK code that we pass to the API must be a 10-digit string, with possible zeros on the left. Taking Ford as an example, its CIK code is 37996 and it needs 00000 on its left to reach 10-digits. This is what the for loop in the beginning of the following function does. It is also important to pass good headers with your HTTP get request, since EDGAR requires it.

```Python
def get_raw_data(CIK):
  CIK = str(CIK)
  for i in range(10 - len(CIK)):
    CIK = "0" + CIK
  my_headers = {"user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36" }
  data = requests.get("https://data.sec.gov/api/xbrl/companyfacts/CIK" + CIK + ".json", headers = my_headers)
  data = json.loads(data.text)
  return data
Now the following few lines of code, calls the API for all the companies in our list.

for [Ticker, CIK] in CIK_list:
  data = get_raw_data(CIK)
But wait! Do not run this code because the volume of data is so large that printing it to the output is not useful. Explaining the structure of this data is a long story, you can explore it using the keys() method. Long story short, to check our data, lets run the following and check its output right after it.

for [Ticker, CIK] in CIK_list:
  data = get_raw_data(CIK)
  print(data["facts"]["us-gaap"]["Assets"]["units"]["USD"][-1]["val"])
335038000000
411976000000
363248000000
49555000000
90591000000
220168000000
255263000000
101852000000
27838000000
74579000000
132213000000
50817473000
134774000000
255121000000
3868240000000
136662000000
37531000000
67967000000
53206000000
163006000000
383044000000
477607000000
265991000000
```

As you might have found out, here we printed “Assets” value from the balance sheet. The [-1] index means we want the last report in the list, which is the latest one. Therefore, to access each item of the latest 10-K filing of any company, we can use the following value and replace the KEYNAME with the GAAP parameter name we are looking for, e.g., Assets in the previous case.

```Python
data["facts"]["us-gaap"][KEYNAME]["units"]["USD"][-1]["val"]
You can run the Keys() method and you will get a list of all GAAP parameters that come with the specific report. Check out the following. I have removed major part of the output below to make it short.

data["facts"]["us-gaap"].keys()
__________________________________

dict_keys(['AccountsPayableAndAccruedLiabilitiesCurrent', 'AccountsPayableCurrentAndNoncurrent', 'AccountsPayableRelatedPartiesCurrentAndNoncurrent', 'AccountsReceivableNetCurrent', 'AccountsReceivableRelatedParties', 
..., 
'OtherComprehensiveIncomeLossNetOfTaxPortionAttributableToParent', 'SupplierFinanceProgramObligation', 'SupplierFinanceProgramObligationDecreaseSettlement'])
```

Now let’s do some simple analyses on the data. We will define a function to calculate some principal finance ratios: Current ratio, Leverage ration, P/E and Inventory Turnover. We have an additional function, get_price, which returns the last close of the company’s share price and is used for P/E ratio.

```Python
def calculate_ratios(data, ticker):
  AssetsCurrent = (data["facts"]["us-gaap"]["AssetsCurrent"]["units"]["USD"][-1]["val"])
  LiabilitiesCurrent = (data["facts"]["us-gaap"]["LiabilitiesCurrent"]["units"]["USD"][-1]["val"])
  CurrentRatio = AssetsCurrent / LiabilitiesCurrent

  Assets = (data["facts"]["us-gaap"]["Assets"]["units"]["USD"][-1]["val"])
  StockholdersEquity = (data["facts"]["us-gaap"]["StockholdersEquity"]["units"]["USD"][-1]["val"])
  LeverageRatio = Assets / StockholdersEquity

  EarningsPerShareBasic = (data["facts"]["us-gaap"]["EarningsPerShareBasic"]["units"]["USD/shares"][-1]["val"])
  SharePrice = get_price(ticker)
  PERatio = float(SharePrice) / EarningsPerShareBasic

  InventoryNet = (data["facts"]["us-gaap"]["InventoryNet"]["units"]["USD"][-1]["val"])
  CostOfGoodsSold = (data["facts"]["us-gaap"]["CostOfGoodsSold"]["units"]["USD"][-1]["val"])
  InventoryTurnover = CostOfGoodsSold / InventoryNet

  ratios = [CurrentRatio, LeverageRatio, PERatio, InventoryTurnover]
  return ratios
  
def get_price(ticker):
    data = download(tickers=ticker, period='1d', interval='1h', progress=False)
    share_price = str(data['Close'][0])
    return share_price
```

Running the calculate_ratios function above returns the four financial ratios. However, some companies have their specific way of reporting certain items on the balance sheet, income statement or cashflow statement. Therefore, in some cases some of the key values are missing. You see these missing data marked as __ in the following output results. The code presented in this post was part of the main program that gives the following output. The full code is accessible on my Google Colab.

```Python
      Ticker         Current        Leverage             P/E        Inventory Turnover
     ___________________________________________________________________________________
        aapl           0.982           5.559         139.387                     6.174
        msft           1.769           1.998          33.837                      1.37
         xom           1.484           1.825          60.626                        __
        nvda           2.787           1.802         174.456                     1.454
        tsla            1.59           1.772         313.176                     0.176
         pfe           2.117           2.223          81.878                     0.249
        baba           1.811           1.771          173.98                        __
        csco           1.385           2.296          18.188                     0.995
        adbe           1.157           1.876         186.848                        __
        pypl           1.296           3.793          67.694                        __
         ibm            1.06           5.955          83.672                     4.834
        nflx           1.326           2.226         118.385                        __
          ba           1.167              __          -826.2                        __
         wmt           0.827           3.207          55.952                     2.076
         jpm              __          12.378          31.132                        __
        orcl           0.874          57.663         127.787                     5.987
         nke           2.723            2.68          29.376                     1.941
         amd            2.18           1.233          5054.5                      0.23
         tgt           0.833           4.438          65.718                     1.403
          ge           1.252           5.226          -51.75                     0.845
       googl           2.172           1.434          95.828                    10.182
        amzn           0.948           2.833         213.265                     1.896
           f           1.205            6.09          25.858                     2.117
```

*These values are the result of basic analysis of the raw data from EDGAR and some of them are incorrect because no preprocessing & corrections have been performed.*
