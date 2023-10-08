# 动量策略代码说明-实验课1

***

**目录**

[toc]

***

## 前言……

老师和助教同学好！

我使用`Python`代码来复现动量策略，，最终代码文件为`Final_Momentum_Strategy.ipynb`。下面的部分按照“策略构建”、“结果展示”和“结果分析”三部分展开叙述。

这份代码我花费了很多时间，但我知道当中仍然存在一些问题（甚至可能相当致命）。如果您有兴趣的话，可以与我联系，告诉我您对于这份代码的想法，请不吝赐教！

> **姓名：** 
>
> **学号：** 2021310430
>
> **联系方式： ** 

## 策略构建

### 数据探索

基于A股市场月个股回报率数据构建策略，首先需要对数据进行探索，必要时进行清洗工作。

#### 数据结构

数据文件为`TRD_Mnth.csv`，为长面板数据，其中个体标识符为'Stkcd'，时间标识符为'Trdmnt'，下面的表格分析了各字段在读入`Python`后的数据类型以及含义

|     字段     | 数据类型 | 含义                               |
| :----------: | :------: | ---------------------------------- |
|   `Stkcd`    |  int64   | 证券代码                           |
|   `Trdmnt`   |  object  | 交易月份                           |
|  `Msmvosd`   | float64  | 月个股流通市值                     |
|   `Mretwd`   | float64  | 考虑现金红利再投资的月个股回报率   |
| `Markettype` |  int64   | 市场类型，在本次策略构建中可以忽略 |

#### 数据特征

月个股回报率数据时间跨度为1990-12至2022-08，但其中并非所有的个股都对应了这其中的所有月份。

观察可以得到，每只股票第一个月的收益率为空值。

同时，运行代码检查`Stkcd`对应的月份是否连续，代码如下：

```python
# 创建data_result的副本
data_result_copy = data_result.copy()

# 提取Trdmnt中的月份
data_result_copy['month'] = data_result_copy['Trdmnt'].dt.month

# 定义一个函数，检查月份是否连续
def is_month_consecutive(group):
    sorted_months = group.sort_values()
    diff = np.diff(sorted_months)
    return np.all(diff == 1) or np.all(diff == -11)

# 按照Stkcd进行分组，在组内检查month是否连续
consecutive_check = data_result_copy.groupby('Stkcd')['month'].apply(is_month_consecutive)

print(consecutive_check)

# 筛选出月份连续的股票代码
consecutive_stkcd = consecutive_check[consecutive_check].index
print(consecutive_stkcd)

# 保留月份连续的股票代码的数据
data_1 = data_result_copy[data_result_copy['Stkcd'].isin(consecutive_stkcd)]

# data_1现在包含了符合条件的且月份连续的股票代码的数据
data_1.head()
```

结果data_1中没有任何数据，说明没有任何一只股票对应的月份是完全连续的。

但在之后的代码处理中，月份不连续并不实际影响收益率的计算，前提是将收益率计算时有缺失值的数据剔除。

### 代码部分

下面进行策略的构建。

#### 工作路径设定

建立工作区文件夹，文件夹下建立子文件夹“1_Rawdata”,“2_Clean”,"Code","3_Output"，其中代码放置于"Code"，月个股回报率文件`TRD_Mnth.csv`放置于“1_Rawdata”。

```python
import os

# 获取当前工作目录
current_dir = os.getcwd()
print(f"Current working directory: {current_dir}")

# 使用相对路径定位到目标目录
target_dir = os.path.join(current_dir, '..', '1_Rawdata')

# 更改工作目录
os.chdir(target_dir)
print(f"Target working directory: {os.getcwd()}")
```

#### 读取数据并导入相关模块

```python
import pandas as pd
import numpy as np
import datetime as dt
import matplotlib.pyplot as plt
import seaborn as sns

# 读取数据
data = pd.read_csv('TRD_Mnth.csv')
data = data.drop(['Markettype'], axis=1)
```

#### 设定动量策略的参数

需要设定的参数为：

- `forming_month_j`：形成期长度
- `holding_month_k`：持有期长度
- `trade_begin`与`trade_end`：回测期的开始与结束日期

```python
forming_month_j = [1, 3, 6]
holding_month_k = [1, 3, 6]

trade_begin = '2020-01-01'
trade_end = '2020-12-31'
```

#### 数据预处理

见代码注释

```python
# 确保数据按股票代码和日期排序
data = data.sort_values(['Stkcd', 'Trdmnt'])
data['Trdmnt'] = pd.to_datetime(data['Trdmnt'])

# 设置回测期
data_trade = data[(data['Trdmnt'] >= trade_begin) & (data['Trdmnt'] <= trade_end)]

# 将Msmvosd缺失的数据剔除
data = data[data['Msmvosd'].notnull()]

# 初始化一个DataFrame来保存不同策略的收益
result = pd.DataFrame(index=forming_month_j, columns=holding_month_k)
result_weighted = pd.DataFrame(index=forming_month_j, columns=holding_month_k)
```

#### 循环体部分——策略构建

首先获得回测期当中的每个月份，作为接下来循环中的循环变量。

```python
# 获取唯一的年月，作为回测的月份点
unique_year_month = pd.PeriodIndex(data_trade['Trdmnt'].dt.to_period('M').unique())
```

接下来，首先附上循环体的完整代码，然后对循环体的每个部分进行说明。

```python
# 开始循环：不同形成期和持有期
for form_period in forming_month_j:
    for hold_period in holding_month_k:

        all_returns = []
        all_returns_winners = []
        all_returns_losers = []
        past_winners = []
        past_losers = []

        all_returns_weighted = []
        all_returns_winners_weighted = []
        all_returns_losers_weighted = []

        # 根据hold_period生成额外的月份
        extra_months = pd.PeriodIndex([unique_year_month[-1] + i for i in range(1, hold_period)], freq='M')
        full_year_month = unique_year_month.union(extra_months)

        # 遍历每一个唯一的年月，包括额外的月份
        for current_ym in full_year_month:
            current_ym_str = str(current_ym)
            current_ym_datetime = pd.to_datetime(current_ym.to_timestamp()).to_pydatetime()
            if current_ym_datetime <= unique_year_month.max().to_timestamp():
                
                end_date = pd.to_datetime(current_ym.to_timestamp())
                start_date = end_date - pd.DateOffset(months=form_period)

                mask = (data['Trdmnt'] > start_date) & (data['Trdmnt'] <= end_date)
                form_returns = data.loc[mask].groupby('Stkcd')['Mretwd'].apply(lambda x: (1 + x).prod() - 1)

                available_stocks = data.loc[data['Trdmnt'] > end_date]['Stkcd'].unique()
                form_returns = form_returns[form_returns.index.isin(available_stocks)]

                #将股票按form_returns大小分为5组
                groups_form_returns = pd.qcut(form_returns, 5, labels=False)

                #赢家股票为form_returns最高的1组
                winners = form_returns[groups_form_returns == 4].index.tolist()

                #输家股票为form_returns最低的1组
                losers = form_returns[groups_form_returns == 0].index.tolist()

                past_winners.append(winners)
                past_losers.append(losers)

                # 如果我们的列表变得过长，就移除旧的元素
                if len(past_winners) > hold_period:
                    past_winners.pop(0)
                    past_losers.pop(0)
            else:
                # 如果不在交易期内，则继续移除旧的元素
                past_winners.pop(0)
                past_losers.pop(0)
            
            # 计算当月所有past_winners和past_losers的平均收益
            mask = (data['Trdmnt'] == str(current_ym))
            current_winner_returns = [
                data.loc[mask & data['Stkcd'].isin(winners_month)]['Mretwd'].mean() 
                for winners_month in past_winners
            ]
            current_loser_returns = [
                data.loc[mask & data['Stkcd'].isin(losers_month)]['Mretwd'].mean() 
                for losers_month in past_losers
            ]
            
            # 取算数平均
            current_winner_return = np.mean(current_winner_returns)
            current_loser_return = np.mean(current_loser_returns)

            # 取加权平均
            current_winner_returns_weighted = [
                np.average(
                    data.loc[mask & data['Stkcd'].isin(winners_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Mretwd'], 
                    weights=data.loc[mask & data['Stkcd'].isin(winners_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Msmvosd']
                )
                for winners_month in past_winners
            ]
            current_loser_returns_weighted = [
                np.average(
                    data.loc[mask & data['Stkcd'].isin(losers_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Mretwd'], 
                    weights=data.loc[mask & data['Stkcd'].isin(losers_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Msmvosd']
                )
                for losers_month in past_losers
            ]

            # 取加权平均
            current_winner_return_weighted = np.nanmean(current_winner_returns_weighted)
            current_loser_return_weighted = np.nanmean(current_loser_returns_weighted)

            # 计算策略收益：赢家收益 - 输家收益
            #strategy_return = current_winner_return - current_loser_return
            all_returns_winners.append(current_winner_return)
            all_returns_losers.append(current_loser_return)

            all_returns_winners_weighted.append(current_winner_return_weighted)
            all_returns_losers_weighted.append(current_loser_return_weighted)

            all_returns.append(current_winner_return - current_loser_return)
            all_returns_weighted.append(current_winner_return_weighted - current_loser_return_weighted)

        # def calculate_annual_returns(all_returns):
        #     all_returns_clean = [x for x in all_returns if not np.isnan(x)]
        #     cumulative_returns = [x + 1 for x in all_returns_clean]
        #     geometric_mean = np.prod(cumulative_returns)**(1/len(cumulative_returns)) - 1
        #     annual_returns = (geometric_mean + 1) ** 12 - 1
        #     return annual_returns
        
        def calculate_geometric_mean_returns(all_returns):
            all_returns_clean = [x for x in all_returns if not np.isnan(x)]
            cumulative_returns = [x + 1 for x in all_returns_clean]
            geometric_mean_returns = np.prod(cumulative_returns)**(1/len(cumulative_returns)) - 1
            return geometric_mean_returns

        # annual_returns = calculate_annual_returns(all_returns)
        # annual_returns_weighted = calculate_annual_returns(all_returns_weighted)

        geometric_mean_returns = calculate_geometric_mean_returns(all_returns)
        geometric_mean_returns_weighted = calculate_geometric_mean_returns(all_returns_weighted)

        result.loc[form_period, hold_period] = geometric_mean_returns
        result_weighted.loc[form_period, hold_period] = geometric_mean_returns_weighted



```

##### 外层：选择不同形成期

```python
# 开始循环：不同形成期和持有期
for form_period in forming_month_j:
```

对`forming_month_j`中不同的形成期进行循环，用于构建不同形成期的动量策略。

##### 中间层：选择不同持有期

```python
    for hold_period in holding_month_k:
```

对`hold_period`中不同的持有期进行循环，用于构建不同持有期的动量策略。

##### 内层：动量策略构建

```python
        all_returns = []
        all_returns_winners = []
        all_returns_losers = []
        past_winners = []
        past_losers = []

        all_returns_weighted = []
        all_returns_winners_weighted = []
        all_returns_losers_weighted = []
```

首先定义一些空列表对象，用来储存后面计算结果。

```python
        # 根据hold_period生成额外的月份
        extra_months = pd.PeriodIndex([unique_year_month[-1] + i for i in range(1, hold_period)], freq='M')
        full_year_month = unique_year_month.union(extra_months)
```

将收益率计算的时间范围定义为：回测期加上回测期之外`hold_period - 1`个月，这是由于在回测期之外`hold_period - 1`个月的每一个月中，所持有的股票数量会不断减少，即赢家和输家股票会不断减少，但是仍然会继续有收益率，直到回测期之外`hold_period - 1`个月。

```python
        # 遍历每一个唯一的年月，包括额外的月份
        for current_ym in full_year_month:
            current_ym_str = str(current_ym)
            current_ym_datetime = pd.to_datetime(current_ym.to_timestamp()).to_pydatetime()
```

遍历收益率计算时间范围内的所有月份，接下来计算收益率。

```python
            if current_ym_datetime <= unique_year_month.max().to_timestamp():
        		......
            else:
                # 如果不在交易期内，则继续移除旧的元素
                past_winners.pop(0)
                past_losers.pop(0)
```

如果超出回测期范围，则不再增加赢家和输家股票，并每次循环减少最开始的一组赢家和输家股票。

```python
                end_date = pd.to_datetime(current_ym.to_timestamp())
                start_date = end_date - pd.DateOffset(months=form_period)
```

设定形成期的开始日期和结束日期，为计算该时间范围内股票收益率，从而筛选得到winners和losers股票。

```python
                mask = (data['Trdmnt'] > start_date) & (data['Trdmnt'] <= end_date)
                form_returns = data.loc[mask].groupby('Stkcd')['Mretwd'].apply(lambda x: (1 + x).prod() - 1)

                available_stocks = data.loc[data['Trdmnt'] > end_date]['Stkcd'].unique()
                form_returns = form_returns[form_returns.index.isin(available_stocks)]
```

使用布尔掩码计算形成期内个股的总收益率，然后使用持有期内有收益率数据的股票列表`available_stocks`进行筛选。

```python
                #将股票按form_returns大小分为5组
                groups_form_returns = pd.qcut(form_returns, 5, labels=False)

                #赢家股票为form_returns最高的1组
                winners = form_returns[groups_form_returns == 4].index.tolist()

                #输家股票为form_returns最低的1组
                losers = form_returns[groups_form_returns == 0].index.tolist()

                past_winners.append(winners)
                past_losers.append(losers)
```

对股票按形成期总收益率进行排序分组，得到winners和losers，并将该月的赢家和输家股票加入所有的赢家或输家股票。

```python
                # 如果我们的列表变得过长，就移除旧的元素
                if len(past_winners) > hold_period:
                    past_winners.pop(0)
                    past_losers.pop(0)
```

当列表变得过长，就移除旧的元素，保证当月计算的收益率对应的股票都处在持有期内。

```python
            # 计算当月所有past_winners和past_losers的平均收益
            mask = (data['Trdmnt'] == str(current_ym))
            current_winner_returns = [
                data.loc[mask & data['Stkcd'].isin(winners_month)]['Mretwd'].mean() 
                for winners_month in past_winners
            ]
            current_loser_returns = [
                data.loc[mask & data['Stkcd'].isin(losers_month)]['Mretwd'].mean() 
                for losers_month in past_losers
            ]
            
            # 取算数平均
            current_winner_return = np.mean(current_winner_returns)
            current_loser_return = np.mean(current_loser_returns)
```

计算当月所有处于持有期内股票的收益率，得到该月的收益率，此处同一月份做多做空的股票在计算收益率时采取非加权的平均数。

```python
            # 取加权平均
            current_winner_returns_weighted = [
                np.average(
                    data.loc[mask & data['Stkcd'].isin(winners_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Mretwd'], 
                    weights=data.loc[mask & data['Stkcd'].isin(winners_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Msmvosd']
                )
                for winners_month in past_winners
            ]
            current_loser_returns_weighted = [
                np.average(
                    data.loc[mask & data['Stkcd'].isin(losers_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Mretwd'], 
                    weights=data.loc[mask & data['Stkcd'].isin(losers_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Msmvosd']
                )
                for losers_month in past_losers
            ]

            # 取加权平均
            current_winner_return_weighted = np.nanmean(current_winner_returns_weighted)
            current_loser_return_weighted = np.nanmean(current_loser_returns_weighted)
```

计算当月所有处于持有期内股票的收益率，得到该月的收益率，此处同一月份做多做空的股票在计算收益率时采取按照个股市值加权的平均数。在计算加权平均数时，将市值缺失的数据进行了提前剔除。

```python
            # 计算策略收益：赢家收益 - 输家收益
            #strategy_return = current_winner_return - current_loser_return
            all_returns_winners.append(current_winner_return)
            all_returns_losers.append(current_loser_return)

            all_returns_winners_weighted.append(current_winner_return_weighted)
            all_returns_losers_weighted.append(current_loser_return_weighted)

            all_returns.append(current_winner_return - current_loser_return)
            all_returns_weighted.append(current_winner_return_weighted - current_loser_return_weighted)
```

将该月的收益率数据放入先前定义的列表中。

```python
        def calculate_geometric_mean_returns(all_returns):
            all_returns_clean = [x for x in all_returns if not np.isnan(x)]
            cumulative_returns = [x + 1 for x in all_returns_clean]
            geometric_mean_returns = np.prod(cumulative_returns)**(1/len(cumulative_returns)) - 1
            return geometric_mean_returns

        # annual_returns = calculate_annual_returns(all_returns)
        # annual_returns_weighted = calculate_annual_returns(all_returns_weighted)

        geometric_mean_returns = calculate_geometric_mean_returns(all_returns)
        geometric_mean_returns_weighted = calculate_geometric_mean_returns(all_returns_weighted)
```

对于给定形成期和持有期情况下，通过定义计算几何平均数函数来计算所有产生收益率月份对应收益率的几何平均数，包括非加权和加权的情况。

```python
        result.loc[form_period, hold_period] = geometric_mean_returns
        result_weighted.loc[form_period, hold_period] = geometric_mean_returns_weighted
```

将给定形成期和持有期情况下得到的策略收益率，按照给定的行与列放入结果数据框中，包括非加权和加权的情况。注意该收益率为月收益率。

#### 结果展示

```python
print(result)
print(result_weighted)
```

打印对于不同形成期与持有期，动量策略的收益率结果数据框。

```python
import matplotlib.pyplot as plt
import seaborn as sns

# 绘制热力图
sns.heatmap(result.astype(float), annot=True, fmt=".2%", cmap="vlag", center=0)
title_line1 = "Momentum Strategy Returns (Jegadeesh and Titman, 1993)"
title_line2 = "Unweighted Average"
title_line3 = trade_begin + "~" + trade_end
plt.title(f"{title_line1}\n{title_line2}\n{title_line3}")
plt.xlabel("Holding Period (months)")
plt.ylabel("Formation Period (months)")

# 保存图像到'3_Output'文件夹
output_pic_file_name = "Momentum_Strategy_heatmap_" + trade_begin + "~" + trade_end + ".png"
trading_period = trade_begin + "~" + trade_end
output_folder = f'../3_Output/{trading_period}'
os.makedirs(output_folder, exist_ok=True)
output_pic_file_name = "Momentum_Strategy_heatmap_" + trading_period + ".png"
plt.savefig(f'{output_folder}/{output_pic_file_name}')

# 显示图像
plt.show()
```

非加权结果：绘制热力图，色阶以0为界，大于0为红色，小于0为蓝色。对热力图进行命名，将其保存于代码文件夹同目录下的结果数据文件夹'3_Output'。

```python
import matplotlib.pyplot as plt
import seaborn as sns

# 绘制热力图
sns.heatmap(result_weighted.astype(float), annot=True, fmt=".2%", cmap="vlag", center=0)
title_line1 = "Momentum Strategy Returns (Jegadeesh and Titman, 1993)"
title_line2 = "Weighted Average"
title_line3 = trade_begin + "~" + trade_end
plt.title(f"{title_line1}\n{title_line2}\n{title_line3}")
plt.xlabel("Holding Period (months)")
plt.ylabel("Formation Period (months)")

# 保存图像到'3_Output'文件夹
output_pic_file_name = "Momentum_Strategy_heatmap_weighted_" + trade_begin + "~" + trade_end + ".png"
trading_period = trade_begin + "~" + trade_end
output_folder = f'../3_Output/{trading_period}'
os.makedirs(output_folder, exist_ok=True)
output_pic_file_name = "Momentum_Strategy_heatmap_weighted_" + trading_period + ".png"
plt.savefig(f'{output_folder}/{output_pic_file_name}')

# 显示图像
plt.show()

```

加权结果：绘制热力图，色阶以0为界，大于0为红色，小于0为蓝色。对热力图进行命名，将其保存于代码文件夹同目录下的结果数据文件夹'3_Output'。

### 完整代码

```python
import os

# 获取当前工作目录
current_dir = os.getcwd()
print(f"Current working directory: {current_dir}")

# 使用相对路径定位到目标目录
target_dir = os.path.join(current_dir, '..', '1_Rawdata')

# 更改工作目录
os.chdir(target_dir)
print(f"Target working directory: {os.getcwd()}")

import pandas as pd
import numpy as np
import datetime as dt
import matplotlib.pyplot as plt
import seaborn as sns

# 读取数据
data = pd.read_csv('TRD_Mnth.csv')
data = data.drop(['Markettype'], axis=1)

forming_month_j = [1, 3, 6]
holding_month_k = [1, 3, 6]

trade_begin = '2000-01-01'
trade_end = '2010-12-31'

# 确保数据按股票代码和日期排序
data = data.sort_values(['Stkcd', 'Trdmnt'])
data['Trdmnt'] = pd.to_datetime(data['Trdmnt'])

# 设置回测期
data_trade = data[(data['Trdmnt'] >= trade_begin) & (data['Trdmnt'] <= trade_end)]

# 将Msmvosd缺失的数据剔除
data = data[data['Msmvosd'].notnull()]

# 初始化一个DataFrame来保存不同策略的收益
result = pd.DataFrame(index=forming_month_j, columns=holding_month_k)
result_weighted = pd.DataFrame(index=forming_month_j, columns=holding_month_k)

# 获取唯一的年月，作为回测的月份点
unique_year_month = pd.PeriodIndex(data_trade['Trdmnt'].dt.to_period('M').unique())

# 开始循环：不同形成期和持有期
for form_period in forming_month_j:
    for hold_period in holding_month_k:

        all_returns = []
        all_returns_winners = []
        all_returns_losers = []
        past_winners = []
        past_losers = []

        all_returns_weighted = []
        all_returns_winners_weighted = []
        all_returns_losers_weighted = []

        # 根据hold_period生成额外的月份
        extra_months = pd.PeriodIndex([unique_year_month[-1] + i for i in range(1, hold_period)], freq='M')
        full_year_month = unique_year_month.union(extra_months)

        # 遍历每一个唯一的年月，包括额外的月份
        for current_ym in full_year_month:
            current_ym_str = str(current_ym)
            current_ym_datetime = pd.to_datetime(current_ym.to_timestamp()).to_pydatetime()
            if current_ym_datetime <= unique_year_month.max().to_timestamp():
                
                end_date = pd.to_datetime(current_ym.to_timestamp())
                start_date = end_date - pd.DateOffset(months=form_period)

                mask = (data['Trdmnt'] > start_date) & (data['Trdmnt'] <= end_date)
                form_returns = data.loc[mask].groupby('Stkcd')['Mretwd'].apply(lambda x: (1 + x).prod() - 1)

                available_stocks = data.loc[data['Trdmnt'] > end_date]['Stkcd'].unique()
                form_returns = form_returns[form_returns.index.isin(available_stocks)]

                #将股票按form_returns大小分为5组
                groups_form_returns = pd.qcut(form_returns, 5, labels=False)

                #赢家股票为form_returns最高的1组
                winners = form_returns[groups_form_returns == 4].index.tolist()

                #输家股票为form_returns最低的1组
                losers = form_returns[groups_form_returns == 0].index.tolist()

                past_winners.append(winners)
                past_losers.append(losers)

                # 如果我们的列表变得过长，就移除旧的元素
                if len(past_winners) > hold_period:
                    past_winners.pop(0)
                    past_losers.pop(0)
            else:
                # 如果不在交易期内，则继续移除旧的元素
                past_winners.pop(0)
                past_losers.pop(0)
            
            # 计算当月所有past_winners和past_losers的平均收益
            mask = (data['Trdmnt'] == str(current_ym))
            current_winner_returns = [
                data.loc[mask & data['Stkcd'].isin(winners_month)]['Mretwd'].mean() 
                for winners_month in past_winners
            ]
            current_loser_returns = [
                data.loc[mask & data['Stkcd'].isin(losers_month)]['Mretwd'].mean() 
                for losers_month in past_losers
            ]
            
            # 取算数平均
            current_winner_return = np.mean(current_winner_returns)
            current_loser_return = np.mean(current_loser_returns)

            # 取加权平均
            current_winner_returns_weighted = [
                np.average(
                    data.loc[mask & data['Stkcd'].isin(winners_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Mretwd'], 
                    weights=data.loc[mask & data['Stkcd'].isin(winners_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Msmvosd']
                )
                for winners_month in past_winners
            ]
            current_loser_returns_weighted = [
                np.average(
                    data.loc[mask & data['Stkcd'].isin(losers_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Mretwd'], 
                    weights=data.loc[mask & data['Stkcd'].isin(losers_month) & data['Mretwd'].notna() & data['Msmvosd'].notna()]['Msmvosd']
                )
                for losers_month in past_losers
            ]

            # 取加权平均
            current_winner_return_weighted = np.nanmean(current_winner_returns_weighted)
            current_loser_return_weighted = np.nanmean(current_loser_returns_weighted)

            # 计算策略收益：赢家收益 - 输家收益
            #strategy_return = current_winner_return - current_loser_return
            all_returns_winners.append(current_winner_return)
            all_returns_losers.append(current_loser_return)

            all_returns_winners_weighted.append(current_winner_return_weighted)
            all_returns_losers_weighted.append(current_loser_return_weighted)

            all_returns.append(current_winner_return - current_loser_return)
            all_returns_weighted.append(current_winner_return_weighted - current_loser_return_weighted)

        # def calculate_annual_returns(all_returns):
        #     all_returns_clean = [x for x in all_returns if not np.isnan(x)]
        #     cumulative_returns = [x + 1 for x in all_returns_clean]
        #     geometric_mean = np.prod(cumulative_returns)**(1/len(cumulative_returns)) - 1
        #     annual_returns = (geometric_mean + 1) ** 12 - 1
        #     return annual_returns
        
        def calculate_geometric_mean_returns(all_returns):
            all_returns_clean = [x for x in all_returns if not np.isnan(x)]
            cumulative_returns = [x + 1 for x in all_returns_clean]
            geometric_mean_returns = np.prod(cumulative_returns)**(1/len(cumulative_returns)) - 1
            return geometric_mean_returns

        # annual_returns = calculate_annual_returns(all_returns)
        # annual_returns_weighted = calculate_annual_returns(all_returns_weighted)

        geometric_mean_returns = calculate_geometric_mean_returns(all_returns)
        geometric_mean_returns_weighted = calculate_geometric_mean_returns(all_returns_weighted)

        result.loc[form_period, hold_period] = geometric_mean_returns
        result_weighted.loc[form_period, hold_period] = geometric_mean_returns_weighted

print(result)
print(result_weighted)

import matplotlib.pyplot as plt
import seaborn as sns

# 绘制热力图
sns.heatmap(result.astype(float), annot=True, fmt=".2%", cmap="vlag", center=0)
title_line1 = "Momentum Strategy Returns (Jegadeesh and Titman, 1993)"
title_line2 = "Unweighted Average"
title_line3 = trade_begin + "~" + trade_end
plt.title(f"{title_line1}\n{title_line2}\n{title_line3}")
plt.xlabel("Holding Period (months)")
plt.ylabel("Formation Period (months)")

# 保存图像到'3_Output'文件夹
output_pic_file_name = "Momentum_Strategy_heatmap_" + trade_begin + "~" + trade_end + ".png"
trading_period = trade_begin + "~" + trade_end
output_folder = f'../3_Output/{trading_period}'
os.makedirs(output_folder, exist_ok=True)
output_pic_file_name = "Momentum_Strategy_heatmap_" + trading_period + ".png"
plt.savefig(f'{output_folder}/{output_pic_file_name}')

# 显示图像
plt.show()

import matplotlib.pyplot as plt
import seaborn as sns

# 绘制热力图
sns.heatmap(result_weighted.astype(float), annot=True, fmt=".2%", cmap="vlag", center=0)
title_line1 = "Momentum Strategy Returns (Jegadeesh and Titman, 1993)"
title_line2 = "Weighted Average"
title_line3 = trade_begin + "~" + trade_end
plt.title(f"{title_line1}\n{title_line2}\n{title_line3}")
plt.xlabel("Holding Period (months)")
plt.ylabel("Formation Period (months)")

# 保存图像到'3_Output'文件夹
output_pic_file_name = "Momentum_Strategy_heatmap_weighted_" + trade_begin + "~" + trade_end + ".png"
trading_period = trade_begin + "~" + trade_end
output_folder = f'../3_Output/{trading_period}'
os.makedirs(output_folder, exist_ok=True)
output_pic_file_name = "Momentum_Strategy_heatmap_weighted_" + trading_period + ".png"
plt.savefig(f'{output_folder}/{output_pic_file_name}')

# 显示图像
plt.show()
```

## 结果展示

设定不同的回测期，会有不同的结果。下面分别设定回测期为 '2000-01-01' ~'2010-12-31'和'2010-01-01'~ '2020-12-31'，得到动量策略收益率结果如下。

### '2000-01-01' ~'2010-12-31'

![Momentum_Strategy_heatmap_2000-01-01~2010-12-31](https://cdn.jsdelivr.net/gh/HenryK39B5/Blogverse@master/img/Momentum_Strategy_heatmap_2000-01-012010-12-31.png)

![Momentum_Strategy_heatmap_weighted_2000-01-01~2010-12-31](https://cdn.jsdelivr.net/gh/HenryK39B5/Blogverse@master/img/Momentum_Strategy_heatmap_weighted_2000-01-012010-12-31.png)

### '2010-01-01'~ '2020-12-31'

![Momentum_Strategy_heatmap_2010-01-01~2020-12-31](https://cdn.jsdelivr.net/gh/HenryK39B5/Blogverse@master/img/Momentum_Strategy_heatmap_2010-01-012020-12-31.png)

![Momentum_Strategy_heatmap_weighted_2010-01-01~2020-12-31](https://cdn.jsdelivr.net/gh/HenryK39B5/Blogverse@master/img/Momentum_Strategy_heatmap_weighted_2010-01-012020-12-31.png)

## 结果分析

从结果来看，中国A股市场存在动量效应，即在此次形成期和持有期设定下，通过动量策略都能够获得正的收益。值得说明的是，这个结果需要进一步进行 Newey-West 标准误差估计才能够说明结果是显著的，但由于时间关系，我未能找到在`Python`实现该回归的方式。

而对于不同形成期和持有期形成的不同动量策略来看，形成期和持有期为1所获得的收益是最大的，说明短期的动量效应是非常明显的，而随着持有期的增加，正收益率明显下降。相比之下，形成期增加同样会使得正收益率下降，但影响程度不如持有期。