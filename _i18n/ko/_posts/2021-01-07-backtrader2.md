---
title: "Backtrader(7): Sell & commision"
categories:
  - Backtrader
tags:
  - Backtrader
  - Quant
---

## Sell & Commision

이전의 코드는 주식을 매수(BUY)만 할 뿐, 매도(SELL)하지는 않았습니다. 그래서 이번에는 매도하는 기능을 추가해보도록 하겠습니다. BUY 전략은 동일하되, 매도가 일어난 뒤 5일 뒤에 매도하는 전략을 사용해보겠습니다.  

또한, 현재는 주식을 거래할 때 수수료가 붇지 않는데요, 실제 트레이딩에서는 수수료가 붙기 마련입니다. 이 역시 backtrader에서 간단하게 적용할 수 있으므로 수수료를 설정해보도록 하겠습니다.  

마지막으로 이전의 코드에서 log를 이용해 Buy order를 기록했었는데요, 사실 이 log는 정확하지 않습니다. 이유는 order가 '발생'하는 시간과 '실행'되는 시간이 다르기 때문인데요, `buy `와 `sell` 메쏘드를 기본값으로 실행하게 되면 시장이 열려 있을 때 중 가장 빠른 타이밍에 거래를 진행하게 됩니다. 현재 우리는 close(종가)를 이용해 전략을 세웠으므로, 거래 타이밍은 다음 날 주식시장이 열리는 시각이 됩니다. 따라서, buy order를 생성하면 다음 날 open 가격에 주식을 살 수 있습니다.  

아래는 코드와 그 결과입니다.

~~~python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import datetime  # 시간, 날짜 데이터 타입이 있는 라이브러리입니다
import os.path  # 파일의 path를 관리하기 위한 라이브러리입니다
import sys  # system 전반적인 컨트롤을 위한 라이브러리로, 여기서는 script의 이름(argv[0])을 읽기 위해 사용됩니다

# backtrader 플랫폼을 import합니다
import backtrader as bt

# 금융 데이터 수집 라이브러리입니다
# https://financedata.github.io/index.html
import FinanceDataReader as fdr

# Strategy를 수립합니다
class TestStrategy(bt.Strategy):

    def log(self, txt, dt=None):
        # Strategy가 잘 작동하고 있는지 기록하는 함수입니다
        # 앞으로 대다수의 로그는 이 함수를 이용해 기록될 예정입니다
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 과거 주식 데이터가 self.datas에 저장되어있고,
        # close(종가)를 추적하기 위해 멤버 변수를 설정했습니다
        self.dataclose = self.datas[0].close
        
        # 보류중인 order를 추적합니다
        self.order = None
        
        # 매수가와 수수료를 기록합니다
        self.buyprice = None
        self.buycomm = None        
        
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # Bu/Sell order가 broker에 의해 접수되었을 때입니다
            # 아무것도 하지 않고 반환합니다
            return

        # order가 완료되었는지 체크합니다
        # 이 때, broker가 충분한 cash를 보유하고 있지 않으면 거절할 수 있습니다
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    'BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:  # Sell
                self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))

            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order Canceled/Margin/Rejected')

        # self.order에 현재 보류중인 order가 없음을 기록합니다
        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return

        self.log('OPERATION PROFIT, GROSS %.2f, NET %.2f' %
                 (trade.pnl, trade.pnlcomm))
        
    def next(self):
        # 현재 시점에서 close를 기록합니다
        self.log('Close, %.2f' % self.dataclose[0])

        # 만약 order가 보류중인 경우 연속해서 order를 보내지 않습니다
        if self.order:
            return
        
        # 현재 진입했는지 체크합니다
        # 진입하지 않은 경우에 매수를 진행하고,
        # 진입한 경우에 매도를 진행해야 할 것입니다
        if not self.position:        
        
            if self.dataclose[0] < self.dataclose[-1]:
                # 현재 close가 전날 close보다 작은 상황입니다

                if self.dataclose[-1] < self.dataclose[-2]:
                    # 또한 전날 close가 전전날 close보다 작은 상황입니다 

                    # 주식을 매수합니다. 다만 현재에는 기본 파라미터만을 사용하여 매수를 진행합니다
                    # 파라미터를 어떻게 설정할 수 있는지는 추후에 보도록 하겠습니다
                    self.log('BUY CREATE, %.2f' % self.dataclose[0])
                    
                    # order가 발생한 경우 이를 기록합니다
                    self.order = self.buy()
                    
        else:

            # 이미 진입한 경우 주식을 매도합니다
            if len(self) >= (self.bar_executed + 5):
                # Buy와 마찬가지로 기본값으로 매수합니다
                self.log('SELL CREATE, %.2f' % self.dataclose[0])

                # 역시 order가 발생한 경우 이를 기록합니다
                self.order = self.sell()        


if __name__ == '__main__':
    cerebro = bt.Cerebro()

    # cerebro에 전략을 추가해줍니다
    cerebro.addstrategy(TestStrategy)

    # 신라젠(215600)의 2018년 데이터를 불러옵니다
    df = fdr.DataReader('215600', '2018')
    data = bt.feeds.PandasData(dataname=df)

    # cerebro에 data feed를 추가해줍니다
    cerebro.adddata(data)

    # 초기 자본을 설정합니다
    cerebro.broker.setcash(100000.0)
    
    # 0.1%의 수수료를 설정합니다
    cerebro.broker.setcommission(commission=0.001)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
~~~

~~~
Starting Portfolio Value: 1000000.00
2018-01-02, Close, 102500.00
2018-01-03, Close, 103000.00
2018-01-04, Close, 92200.00
2018-01-05, Close, 100000.00
2018-01-08, Close, 93800.00
2018-01-09, Close, 109000.00
2018-01-10, Close, 98000.00
2018-01-11, Close, 96700.00
2018-01-11, BUY CREATE, 96700.00
2018-01-12, BUY EXECUTED, Price: 99800.00, Cost: 99800.00, Comm 99.80
2018-01-12, Close, 98100.00
2018-01-15, Close, 103900.00
2018-01-16, Close, 102900.00
2018-01-17, Close, 106900.00
2018-01-18, Close, 105200.00
2018-01-19, Close, 103000.00
2018-01-19, SELL CREATE, 103000.00
2018-01-22, SELL EXECUTED, Price: 102600.00, Cost: 99800.00, Comm 102.60
2018-01-22, OPERATION PROFIT, GROSS 2800.00, NET 2597.60
2018-01-22, Close, 104100.00
2018-01-23, Close, 115000.00
...
...
...
Final Portfolio Value: 979142.30
~~~

코드에 대한 설명은 주석으로 달아놓았습니다. 전략이 단순한 만큼 엄청난 손해가 발생했습니다.  

## Parameters & sizers

현재는 strategy가 하드코딩되어있습니다. 이 말은 strategy가 코드로 고정되어 있다는 말로, 저희가 쉽게 전략을 바꿔가면서 테스트해볼 수 없습니다. 예를 들어 우리가 매도하는 타이밍을 매수햐고 5일 뒤가 아닌, 매수하고 10일 뒤로 변경해보고 싶을 때, 직접 관련되 코드를 찾아 바꿔줘야 합니다. 현재는 전략이 간단하여 쉽게 바꿀 수 있지만, 앞으로 복잡한 전략을 테스트할 경우 이런 점들이 상당히 불편할 것입니다.  

다행히도 backtrader에서는 이러한 parameter들을 지정할 수 있는 기능을 가지고 있습니다. 먼저 strategy class 내에 

~~~python
params = (('myparam', 27), ('exitbars', 5),)
~~~

이와 같은 `params` 멤버 변수를 설정해주고, `cerebro.addstrategy`를 아래와 같이 변경해주면 됩니다.

~~~python
# Add a strategy
cerebro.addstrategy(TestStrategy, myparam=20, exitbars=7)
~~~

또한, 주식을 매수할 때 얼마나 살 것인지 결정하는 기능으로 sizer가 있습니다. 이에 대해서는 자세히 알아보도록 하고, 만약 현재의 10배 단위로 주식을 거래하고 싶으면 아래와 같은 코드를 추가하면 됩니다.

~~~python
cerebro.addsizer(bt.sizers.FixedSize, stake=10)
~~~

아래는 전체 코드입니다. `params` 멤버 변수와 `sell()`를 실행하는 부분 위주로 보시길 바랍니다.

~~~python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import datetime  # 시간, 날짜 데이터 타입이 있는 라이브러리입니다
import os.path  # 파일의 path를 관리하기 위한 라이브러리입니다
import sys  # system 전반적인 컨트롤을 위한 라이브러리로, 여기서는 script의 이름(argv[0])을 읽기 위해 사용됩니다

# backtrader 플랫폼을 import합니다
import backtrader as bt

# 금융 데이터 수집 라이브러리입니다
# https://financedata.github.io/index.html
import FinanceDataReader as fdr

# Strategy를 수립합니다
class TestStrategy(bt.Strategy):
    # parameters를 설정합니다
    params = (
        ('exitbars', 5),
    )
    
    def log(self, txt, dt=None):
        # Strategy가 잘 작동하고 있는지 기록하는 함수입니다
        # 앞으로 대다수의 로그는 이 함수를 이용해 기록될 예정입니다
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # 과거 주식 데이터가 self.datas에 저장되어있고,
        # close(종가)를 추적하기 위해 멤버 변수를 설정했습니다
        self.dataclose = self.datas[0].close
        
        # 보류중인 order를 추적합니다
        self.order = None
        
        # 매수가와 수수료를 기록합니다
        self.buyprice = None
        self.buycomm = None        
        
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            # Bu/Sell order가 broker에 의해 접수되었을 때입니다
            # 아무것도 하지 않고 반환합니다
            return

        # order가 완료되었는지 체크합니다
        # 이 때, broker가 충분한 cash를 보유하고 있지 않으면 거절할 수 있습니다
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    'BUY EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))

                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:  # Sell
                self.log('SELL EXECUTED, Price: %.2f, Cost: %.2f, Comm %.2f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))

            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order Canceled/Margin/Rejected')

        # self.order에 현재 보류중인 order가 없음을 기록합니다
        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return

        self.log('OPERATION PROFIT, GROSS %.2f, NET %.2f' %
                 (trade.pnl, trade.pnlcomm))
        
    def next(self):
        # 현재 시점에서 close를 기록합니다
        self.log('Close, %.2f' % self.dataclose[0])

        # 만약 order가 보류중인 경우 연속해서 order를 보내지 않습니다
        if self.order:
            return

        # 현재 진입했는지 체크합니다
        # 진입하지 않은 경우에 매수를 진행하고,
        # 진입한 경우에 매도를 진행해야 할 것입니다
        if not self.position:        
        
            if self.dataclose[0] < self.dataclose[-1]:
                # 현재 close가 전날 close보다 작은 상황입니다

                if self.dataclose[-1] < self.dataclose[-2]:
                    # 또한 전날 close가 전전날 close보다 작은 상황입니다 

                    # 주식을 매수합니다. 다만 현재에는 기본 파라미터만을 사용하여 매수를 진행합니다
                    # 파라미터를 어떻게 설정할 수 있는지는 추후에 보도록 하겠습니다
                    self.log('BUY CREATE, %.2f' % self.dataclose[0])
                    
                    # order가 발생한 경우 이를 기록합니다
                    self.order = self.buy()
                    
        else:

            # 이미 진입한 경우 주식을 매도합니다
            if len(self) >= (self.bar_executed + self.params.exitbars):
                # Buy와 마찬가지로 기본값으로 매수합니다
                self.log('SELL CREATE, %.2f' % self.dataclose[0])

                # 역시 order가 발생한 경우 이를 기록합니다
                self.order = self.sell()        


if __name__ == '__main__':
    cerebro = bt.Cerebro()

    # cerebro에 전략을 추가해줍니다
    cerebro.addstrategy(TestStrategy)

    # 신라젠(215600)의 2018년 데이터를 불러옵니다
    df = fdr.DataReader('215600', '2018')
    data = bt.feeds.PandasData(dataname=df)

    # cerebro에 data feed를 추가해줍니다
    cerebro.adddata(data)

    # 초기 자본을 설정합니다
    cerebro.broker.setcash(1000000.0)
    
    # sizer를 추가하되, 주식의 양을 결정하는 stake 인자를 10으로 설정합니다
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    
    # 0.1%의 수수료를 설정합니다
    cerebro.broker.setcommission(commission=0.001)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
~~~

~~~
Starting Portfolio Value: 1000000.00
2018-01-02, Close, 102500.00
2018-01-03, Close, 103000.00
2018-01-04, Close, 92200.00
2018-01-05, Close, 100000.00
2018-01-08, Close, 93800.00
2018-01-09, Close, 109000.00
2018-01-10, Close, 98000.00
2018-01-11, Close, 96700.00
2018-01-11, BUY CREATE, 96700.00
2018-01-12, BUY EXECUTED, Price: 99800.00, Cost: 998000.00, Comm 998.00
2018-01-12, Close, 98100.00
2018-01-15, Close, 103900.00
2018-01-16, Close, 102900.00
2018-01-17, Close, 106900.00
2018-01-18, Close, 105200.00
2018-01-19, Close, 103000.00
2018-01-19, SELL CREATE, 103000.00
2018-01-22, SELL EXECUTED, Price: 102600.00, Cost: 998000.00, Comm 1026.00
2018-01-22, OPERATION PROFIT, GROSS 28000.00, NET 25976.00
2018-01-22, Close, 104100.00
2018-01-23, Close, 115000.00
...
...
...
2021-01-06, Close, 12100.00
2021-01-07, Close, 12100.00
Final Portfolio Value: 791423.00
~~~

stake를 10으로 설정하였더니, 이전보다 이익과 손해 (GROSS) 모두 10씩 곱해진 점을 확인할 수 있습니다.