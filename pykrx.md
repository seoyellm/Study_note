# pykrx

``` python
from pykrx import stock
from pykrx import bond

# ---------------------------------------------------------------
# stock 모듈
# ---------------------------------------------------------------

#-------market data
stock.get_market_ticker_list(start,end,market='KOSPI') #KOSPI, KOSDAQ, KONEX
stock.get_market_ticker_name(ticker_code)
stock.get_market_ohlcv(start,end,ticker_code)
stock.get_market_cap(start,end,ticker_code)

#-------business day
stock.get_business_days(year,month)

#-------인덱스 조회 API
# 1001 코스피
# 1028 코스피 200
# 1034 코스피 100
# 1035 코스피 50
stock.get_index_ticker_list(date, market='KOSDAQ')
stock.get_index_ticker_name(index_code)

stock.get_index_fundamental(start,end,index_code )   # 지정된 기간 동안 PER/PBR/배당수익률을 조회


# ---------------------------------------------------------------
# bond 모듈
# ---------------------------------------------------------------
bond.get_otc_treasury_yields(start, end, 개별종목)  # 장외 채권수익률을 출력

```



# FinanceDataReader



``` python
import FinanceDataReader as fdr

fdr.DataReader(ticker_code,start,end, exchange='KRX')
# 'KS11'(코스피지수), 'KQ11'(코스닥지수), 'DJI'(다우존스지수), 'IXIC'(나스닥지수), 'US500'(S&P 500지수)

# ----------- stock symbol list
fdr.StockListing('KRX')
# KOSPI, KOSDAQ, KONEX
# NYSE, NASDAQ, AMEX
# S&P500
# SSE, SZSE, HKEX, TSE, HOSE
# KRX-DELISTING : 상장폐지 종목
# KRX-ADMINISTRATIVE : 관리 종목

# ----------- FRED 데이터
m2 = fdr.DataReader('M2', data_source='fred') #  M2통화량
```



# pandas_datareader

``` python
from pandas_datareader import data as pdr
import yfinance as yf

yf.pdr_override()
pdr.get_data_yahoo(ticker_code,start, end)
```

