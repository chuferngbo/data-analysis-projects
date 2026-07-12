# Cookie Cats A/B 测试分析流程与 Python 代码注释

## 1. 项目背景与实验问题

Cookie Cats 是一款移动消除游戏。产品团队希望评估：将首次关卡门槛从第 30 关后移至第 40 关，是否能改善用户留存与参与度。

- **对照组（Control）**：`gate_30`，首次关卡门槛位于第 30 关。
- **实验组（Treatment）**：`gate_40`，首次关卡门槛位于第 40 关。
- **核心业务问题**：实验组是否比对照组拥有更高的 7 日留存率？

数据集包含 90,189 名随机分配到两种版本的用户，字段包括用户 ID、实验分组、14 天游戏轮次、次日留存和 7 日留存。

数据来源：[Kaggle - Mobile Games A/B Testing: Cookie Cats](https://www.kaggle.com/datasets/mursideyarkin/mobile-games-ab-testing-cookie-cats)

---

## 2. 分析流程总览

1. 数据检查：确认数据规模、字段类型、缺失值和重复用户。
2. SRM 检查：确认实验分流比例是否异常。
3. 指标对比：比较两组留存率和游戏参与度的描述性结果。
4. 两样本比例 Z 检验：检验 7 日留存率差异是否显著。
5. 置信区间：判断效果大小的合理范围，而不只看 p 值。
6. 护栏指标：检验次日留存，观察是否伤害短期体验。
7. 非参数检验：检验游戏轮次这类偏态指标的差异。
8. 业务决策：综合统计显著性、效应量、SRM 风险提出上线建议。

---

## 3. 数据检查

### 为什么要做

实验分析的前提是数据可用。需要先确认：

- 数据是否完整读取；
- 一名用户是否只出现一次；
- 实验分组字段和留存字段是否可用；
- 是否存在缺失值或明显异常。

如果用户重复出现，留存率会被重复计算；如果字段缺失或类型错误，后续统计检验就没有可信基础。

### 指标含义

| 字段 | 含义 |
|---|---|
| `userid` | 用户唯一标识，应无重复 |
| `version` | 实验分组：`gate_30` 为对照组，`gate_40` 为实验组 |
| `sum_gamerounds` | 用户安装后 14 天内累计游戏轮次 |
| `retention_1` | 用户是否在安装后第 1 天回访，布尔值 |
| `retention_7` | 用户是否在安装后第 7 天回访，布尔值 |

### Python 代码

```python
from pathlib import Path

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

from scipy.stats import chisquare, mannwhitneyu
from statsmodels.stats.proportion import (
    proportions_ztest,
    confint_proportions_2indep,
    proportion_confint,
)

# 设置图表风格；新版 seaborn 支持 set_theme
sns.set_theme(style="whitegrid")
pd.set_option("display.max_columns", None)

# 读取原始数据。Notebook 位于 AB测试 文件夹时可使用相对路径。
data_path = Path("archive (3)") / "cookie_cats.csv"
df = pd.read_csv(data_path)

# 检查数据规模、重复用户和分组样本量
print("数据规模：", df.shape)
print("重复用户数：", df["userid"].duplicated().sum())
print("\n各实验组样本量：")
display(df["version"].value_counts())

# 查看前几行、字段类型和非空数量
display(df.head())
df.info()
```

### 本次结果

- 数据规模：90,189 行、5 列。
- 重复用户数：0。
- 字段无缺失值，留存指标为布尔值。
- `gate_30`：44,700 人；`gate_40`：45,489 人。

数据可以进入下一步实验质量检查。

---

## 4. SRM（样本比例失衡）检查

### 为什么要做

SRM（Sample Ratio Mismatch，样本比例失衡）用于检查实际分组人数是否与实验设定的分流比例一致。

本实验预期为 50% : 50% 分流。如果实际人数差异过大，可能意味着：

- 随机分流逻辑存在问题；
- 某组数据漏采或重复采集；
- 数据抽取范围不一致；
- 用户存在跨组曝光或异常过滤。

存在 SRM 时，后续的留存差异可能混入数据质量问题，因此需要在报告中明确标注风险。

### 检验原理

- **原假设 H0**：两组样本量符合 50% : 50% 的预期比例。
- **备择假设 H1**：实际样本量显著偏离预期比例。
- 使用卡方检验；当 `p < 0.05` 时，认为存在明显 SRM 风险。

### Python 代码

```python
# 固定分组顺序，便于解释结果
group_counts = (
    df["version"]
    .value_counts()
    .reindex(["gate_30", "gate_40"])
)

# 50:50 分流下，每组理论样本量相同
expected_counts = [len(df) / 2, len(df) / 2]

chi2_stat, p_value_srm = chisquare(
    f_obs=group_counts,
    f_exp=expected_counts,
)

print("实际样本量：")
display(group_counts)
print(f"SRM 卡方统计量：{chi2_stat:.4f}")
print(f"SRM p 值：{p_value_srm:.4f}")

if p_value_srm < 0.05:
    print("结论：存在样本比例失衡，正式决策前需排查分流和数据链路。")
else:
    print("结论：未发现明显样本比例失衡，可继续进行实验分析。")
```

### 本次结果与解释

- SRM p 值：`0.0086`。
- 结果小于 0.05，存在样本比例失衡风险。

这不代表后续分析必须停止，但意味着本项目的统计结论应表述为**带 SRM 风险的分析结果**。在真实业务中，应先修复分流/埋点问题，再重新运行实验。

---

## 5. 核心指标描述性对比

### 为什么要做

在做显著性检验之前，先直观看两组指标水平，确认差异方向和业务量级。

### 指标含义

| 指标 | 计算方式 | 业务含义 |
|---|---|---|
| 次日留存率 | `retention_1=True` 用户数 / 组内用户数 | 新用户短期回访与首日体验 |
| 7 日留存率 | `retention_7=True` 用户数 / 组内用户数 | 用户中期留存，作为本实验主指标 |
| 平均游戏轮次 | `sum_gamerounds` 的均值 | 用户参与度与游戏投入程度 |

### Python 代码

```python
# 按实验组汇总用户量、平均轮次、次日和 7 日留存率
group_summary = (
    df
    .groupby("version", as_index=False)
    .agg(
        users=("userid", "nunique"),
        avg_game_rounds=("sum_gamerounds", "mean"),
        retention_1_rate=("retention_1", "mean"),
        retention_7_rate=("retention_7", "mean"),
    )
)

# 布尔值均值等价于 True 的占比，转换为百分比便于展示
for col in ["retention_1_rate", "retention_7_rate"]:
    group_summary[col] = (group_summary[col] * 100).round(2)

display(group_summary)
```

### 本次结果

| 分组 | 平均游戏轮次 | 次日留存率 | 7 日留存率 |
|---|---:|---:|---:|
| `gate_30` 对照组 | 52.46 | 44.82% | 19.02% |
| `gate_40` 实验组 | 51.30 | 44.23% | 18.20% |

实验组在三个指标上均略低于对照组，下一步需要判断差异是否只是随机波动。

---

## 6. 两样本比例 Z 检验：7 日留存率

### 为什么要做

7 日留存是二元结果（留存/未留存），适合使用两样本比例 Z 检验。该检验回答的问题是：

> 实验组与对照组的 7 日留存率差异，是否超出了随机波动范围？

### 假设设定

- **H0**：`gate_30` 与 `gate_40` 的 7 日留存率相同。
- **H1**：两组 7 日留存率不同。
- 显著性水平：`alpha = 0.05`。

### Python 代码

```python
# 拆分对照组和实验组，后续各项检验复用
control = df.loc[df["version"].eq("gate_30")]
treatment = df.loc[df["version"].eq("gate_40")]

# True 求和等于留存成功人数
control_success = control["retention_7"].sum()
treatment_success = treatment["retention_7"].sum()
control_n = len(control)
treatment_n = len(treatment)

# 两样本比例 Z 检验；two-sided 表示检验是否存在任意方向的差异
z_stat, p_value = proportions_ztest(
    count=np.array([control_success, treatment_success]),
    nobs=np.array([control_n, treatment_n]),
    alternative="two-sided",
)

control_rate = control_success / control_n
treatment_rate = treatment_success / treatment_n

# 效应量使用“实验组 - 对照组”，单位为百分点
diff_pp = (treatment_rate - control_rate) * 100

print(f"对照组 7 日留存率：{control_rate:.2%}")
print(f"实验组 7 日留存率：{treatment_rate:.2%}")
print(f"留存率差异（实验组 - 对照组）：{diff_pp:.2f} 个百分点")
print(f"Z 统计量：{z_stat:.4f}")
print(f"p 值：{p_value:.4f}")
```

### 本次结果与解释

- 对照组 7 日留存率：19.02%。
- 实验组 7 日留存率：18.20%。
- 差异：`-0.82` 个百分点。
- p 值：`0.0016 < 0.05`。

拒绝 H0，实验组 7 日留存率显著低于对照组。关卡后移没有提升主指标，反而带来负向影响。

---

## 7. 95% 置信区间：判断效果大小

### 为什么不能只看 p 值

p 值只能说明“差异是否显著”，不能直接说明差异有多大。样本量很大时，极小差异也可能显著。

置信区间给出效果大小的合理范围：

- 如果区间跨过 0，说明效果方向不确定；
- 如果区间完全为正，实验可能带来提升；
- 如果区间完全为负，实验可能带来下降。

### Python 代码

```python
# 计算“实验组留存率 - 对照组留存率”的 95% 置信区间
ci_low, ci_high = confint_proportions_2indep(
    count1=treatment_success,
    nobs1=treatment_n,
    count2=control_success,
    nobs2=control_n,
    compare="diff",
    method="wald",
)

print(f"95% 置信区间：[{ci_low * 100:.2f}, {ci_high * 100:.2f}] 个百分点")
```

### 本次结果与解释

95% 置信区间为 `[-1.33, -0.31]` 个百分点，整个区间都小于 0。

这说明即使考虑抽样误差，`gate_40` 方案大概率仍会降低 7 日留存，而不是带来提升。

### 7 日留存率可视化代码

```python
# 为两组留存率计算 Wilson 95% 置信区间，用误差线展示不确定性
plot_data = pd.DataFrame({
    "group": ["gate_30", "gate_40"],
    "retention_7_rate": [control_rate, treatment_rate],
    "success": [control_success, treatment_success],
    "nobs": [control_n, treatment_n],
})

ci = plot_data.apply(
    lambda row: proportion_confint(
        count=row["success"],
        nobs=row["nobs"],
        method="wilson",
    ),
    axis=1,
)

plot_data["ci_low"] = [item[0] for item in ci]
plot_data["ci_high"] = [item[1] for item in ci]

plt.figure(figsize=(8, 5))

plt.bar(
    plot_data["group"],
    plot_data["retention_7_rate"] * 100,
    color=["#4C78A8", "#E45756"],
)

plt.errorbar(
    x=plot_data["group"],
    y=plot_data["retention_7_rate"] * 100,
    yerr=[
        (plot_data["retention_7_rate"] - plot_data["ci_low"]) * 100,
        (plot_data["ci_high"] - plot_data["retention_7_rate"]) * 100,
    ],
    fmt="none",
    color="black",
    capsize=6,
)

plt.title("7-Day Retention Rate: Control vs Treatment")
plt.xlabel("Experiment Group")
plt.ylabel("7-Day Retention Rate (%)")
plt.ylim(0, 25)
plt.tight_layout()
plt.show()
```

---

## 8. 护栏指标：次日留存率

### 为什么要做

主指标之外，还要检查实验是否损害其他关键体验。这里将次日留存率作为护栏指标：即使产品希望影响中期留存，也不应明显损害新用户的短期回访。

### Python 代码

```python
# 复用两样本比例 Z 检验逻辑，检验次日留存率
control_retention_1 = control["retention_1"].sum()
treatment_retention_1 = treatment["retention_1"].sum()

z_stat_1d, p_value_1d = proportions_ztest(
    count=np.array([control_retention_1, treatment_retention_1]),
    nobs=np.array([control_n, treatment_n]),
    alternative="two-sided",
)

control_rate_1d = control_retention_1 / control_n
treatment_rate_1d = treatment_retention_1 / treatment_n
diff_pp_1d = (treatment_rate_1d - control_rate_1d) * 100

print(f"对照组次日留存率：{control_rate_1d:.2%}")
print(f"实验组次日留存率：{treatment_rate_1d:.2%}")
print(f"留存率差异（实验组 - 对照组）：{diff_pp_1d:.2f} 个百分点")
print(f"p 值：{p_value_1d:.4f}")
```

### 本次结果与解释

- 对照组次日留存率：44.82%。
- 实验组次日留存率：44.23%。
- 差异：`-0.59` 个百分点。
- p 值：`0.0744 > 0.05`。

次日留存同样呈现下降方向，但在 5% 显著性水平下没有足够证据表明两组存在差异。

---

## 9. 非参数检验：游戏参与度

### 为什么使用 Mann-Whitney U 检验

`sum_gamerounds` 是游戏轮次，通常具有明显右偏分布：少数重度玩家的轮次很高，会拉高均值。此类数据不宜只依赖普通 t 检验。

Mann-Whitney U 检验不要求数据服从正态分布，适合比较两组用户游戏轮次的整体分布是否存在差异。

### Python 代码

```python
# 提取两组用户的 14 天累计游戏轮次
control_rounds = control["sum_gamerounds"]
treatment_rounds = treatment["sum_gamerounds"]

# 检验两组游戏轮次分布是否存在显著差异
u_stat, p_value_rounds = mannwhitneyu(
    control_rounds,
    treatment_rounds,
    alternative="two-sided",
)

rounds_summary = pd.DataFrame({
    "group": ["gate_30", "gate_40"],
    "mean_rounds": [
        control_rounds.mean(),
        treatment_rounds.mean(),
    ],
    "median_rounds": [
        control_rounds.median(),
        treatment_rounds.median(),
    ],
})

display(rounds_summary.round(2))
print(f"Mann-Whitney U 统计量：{u_stat:.0f}")
print(f"p 值：{p_value_rounds:.4f}")
```

### 本次结果与解释

- 对照组平均游戏轮次：52.46；中位数：17。
- 实验组平均游戏轮次：51.30；中位数：16。
- p 值：`0.0502`，略高于 0.05。

在常用的 5% 显著性水平下，不能认为实验组和对照组游戏参与度存在显著差异；但实验组的均值和中位数均更低，应在后续实验中持续监控。

---

## 10. 实验结果汇总表

```python
# 汇总主指标、护栏指标和参与度指标，便于写报告或导出 Excel
experiment_summary = pd.DataFrame({
    "metric": [
        "7日留存率",
        "次日留存率",
        "平均游戏轮次",
    ],
    "control_group": [
        f"{control_rate:.2%}",
        f"{control_rate_1d:.2%}",
        f"{control_rounds.mean():.2f}",
    ],
    "treatment_group": [
        f"{treatment_rate:.2%}",
        f"{treatment_rate_1d:.2%}",
        f"{treatment_rounds.mean():.2f}",
    ],
    "treatment_minus_control": [
        f"{diff_pp:.2f} 个百分点",
        f"{diff_pp_1d:.2f} 个百分点",
        f"{treatment_rounds.mean() - control_rounds.mean():.2f}",
    ],
    "p_value": [
        round(p_value, 4),
        round(p_value_1d, 4),
        round(p_value_rounds, 4),
    ],
})

display(experiment_summary)
```

---

## 11. 最终业务决策

### 结论

不建议将 Cookie Cats 的首次关卡门槛从第 30 关直接后移到第 40 关。

### 决策依据

1. 实验组 7 日留存率显著低于对照组，下降 0.82 个百分点。
2. 95% 置信区间完全为负，说明负向影响并非偶然波动。
3. 次日留存和游戏轮次未显示出显著提升，无法抵消主指标损失。
4. 实验存在 SRM 风险，正式业务决策前应排查随机分流和数据采集链路，并在修复后重复实验。

### 可以写进报告的业务建议

- 保留 `gate_30` 方案，不直接上线 `gate_40`。
- 排查实验分流、埋点与数据抽取逻辑，修复 SRM 问题。
- 若继续探索关卡门槛优化，可测试更小幅度的门槛变化，或配合关卡引导、奖励机制进行新的实验。
- 将 7 日留存作为主指标，次日留存和游戏轮次作为护栏指标持续监控。