---
title: "Backtrader(1): Introduction"
categories:
  - Backtrader
tags:
  - Backtrader
  - Quant
---

안녕하세요. 오늘은 파이썬의 백 테스팅 프레임워크인 backtrader에 대해 공부해보겠습니다.  
본 포스트는 https://www.backtrader.com/docu/ 를 참고했음을 미리 밝힙니다.  

먼저 Docu의 Introduction 항목을 보면 backtrader의 두 가지 목표가 아래와 같이 나와있습니다. ("베스트 키드"에서 미야기의 룰을 인용했다고 밝히기도 합니다.)

1. 손쉬운 사용
2. 1번으로 돌아갈 것

이처럼 사용의 용이성을 목표로 만들었지만, 퀀트와 백테스팅 개념 자체가 익숙치 않은 만큼 곧바로 사용하기는 어려웠습니다. 그래서 관련된 자료를 찾아보던 중 한국어로 된 자료가 많지 않아 앞으로 공부한 내용을 정리해보려 합니다.

## Basic Setup & Setting the Cash

먼저 기본 코드를 살펴보겠습니다.

~~~Python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.broker.setcash(1000000.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
~~~

~~~
Starting Portfolio Value: 1000000.00
Final Portfolio Value: 1000000.00
~~~


첫 줄의 from __future__ import ... 부분은 Python 2와 3의 호환을 위한 부분으로, Python 2에서 Python 3의 문법을 사용하기 위해 사용됩니다. 현재 대부분의 유저가 Python 3을 사용하고 있으니 신경쓰지 않아도 되겠습니다. 그리고 backtrader를 import 합니다.  
main 부분에서는 Cerebro engine을 초기화하는데요, 이 Cerebro는 backtrader의 초석<sup>[1](#footnote_1)</sup> 같은 존재입니다. backtrader의 중심이 되는 부분으로 Data Feeds, strategy, observers, analyzers, writers 등을 저장하고, backtesting이나 trading을 진행한다고 합니다. 앞으로 대부분의 과정이 이 Cerebro를 조작한다고 보시면 될 것 같습니다.  
다음으로 `cerebro.broker.setcash(1000000.0)`이 있는데요, broker는 트레이딩을 진행하는 브로커, setcash는 초기 자본을 설정하는 부분입니다.  
마지막으로 `cerebro.run()`을 이용해 cerebro engine을 수행합니다. 본 문서에서는 백테스팅을 실행하는 부분이 되겠군요. 아직 Data feeds나 Strategy를 지정하지 않았으므로 `cerebro.broker.getvalue()`를 이용해 잔고를 출력했을 때, 그대로임을 볼 수 있습니다.

## Adding a Data Feed & First strategy

다음으로 Data feed를 전달하고, 간단한 트레이딩 전략을 사용하여 백테스팅을 수행해보겠습니다.

~~~Python
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

    def next(self):
        # 현재 시점에서 close를 기록합니다
        self.log('Close, %.2f' % self.dataclose[0])

        if self.dataclose[0] < self.dataclose[-1]:
            # 현재 close가 전날 close보다 작은 상황입니다

            if self.dataclose[-1] < self.dataclose[-2]:
                # 또한 전날 close가 전전날 close보다 작은 상황입니다 

                # 주식을 매수합니다. 다만 현재에는 기본 파라미터만을 사용하여 매수를 진행합니다
                # 파라미터를 어떻게 설정할 수 있는지는 추후에 보도록 하겠습니다
                self.log('BUY CREATE, %.2f' % self.dataclose[0])
                self.buy()


if __name__ == '__main__':
    cerebro = bt.Cerebro()

    # cerebro에 전략을 추가해줍니다
    cerebro.addstrategy(TestStrategy)

    # 신라젠(215600)의 2018년 데이터를 불러옵니다
    data = fdr.DataReader('215600', '2018')

    # cerebro에 data feed를 추가해줍니다
    cerebro.adddata(data)

    # 초기 자본을 설정합니다
    cerebro.broker.setcash(100000.0)

    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    cerebro.run()

    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
~~~

코드에 대한 설명은 주석으로 달아놓았습니다. 다만, 살펴보아야 하는 부분들이 있는데요, 첫번째로 FinanceDataReader에 대한 부분입니다. 이 라이브러리는 금융 데이터를 수집하기 위해 개발된 라이브러리인데요, 손쉽게 사용 가능하고, 한글로 된 설명서가 있어 추천드리고 싶습니다. 관심있으신 분은 [FinanceDataReader 사용자 안내서](https://financedata.github.io/posts/finance-data-reader-users-guide.html)를 참고하시면 좋겠습니다. 다음으로 Strategy를 수립하는 부분입니다. 현재 Strategy는 Close가 두번 연속으로 하락했을 때 매수를 진행하는데요, 이를 위해 `self.dataclose`를 사용했습니다. backtrader에서 index 0은 대개 현재 날짜를 의미합니다. 따라서, 전날 데이터를 조회하고 싶을 때는 -1, 전전날 데이터를 조회하고 싶을 때는 -2를 사용합니다. 마지막으로 현재 코드에서는 주식을 "얼마나" 매수할 것인지 나와있지 않은데요, 이후에 이를 설정하는 방법에 대해 살펴보겠습니다.

<a name="footnote_1">1</a>: https://www.backtrader.com/docu/cerebro/