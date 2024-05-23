# What's this "Margin Attribution"?

In my university course on portfolio analysis, I learned a technique the CFA Institute calls[^cfapaper] **performance attribution**, which like most things from school I never used for its intended purpose. I did however find it extremely useful at my FP&A job for explaining gross margin movements to the CFO in our quarterly business reviews:

> Yes, margin in the Eyewear category is down 30 bps YoY. That's mainly due to shifts in the sales mix: revenue dropped 15% in the super-profitable Direct channel, hitting us for 142 bps, plus sales of the Holbrook SKU are strong this quarter. That's the one we pay royalties on, so thanks to its' sales growth we lost another 255 bps. Without those two factors, our GM would be up - take a look at that 367 bps gain in the Prescription sub-category; looks like our new factory tooling is paying off there.

With this margin attribution analysis we were able to trace margin movements to two distinct causes: **performance contributions** and **mix contributions**, plus we had actual math for comparing our product lines, sales channels, and regions, replacing the guesswork.

This was a big hit, for obvious reasons; I'll spare you any further cooking-blog-style backstory and get straight to the recipe.

# The Formulae

The portfolio's total change in margin, period-over-period, is the sum of the **performance** and **mix** contribution effects for each of its components:

$\Delta TotalMargin =\displaystyle\sum_{0}^n (p_{c0}+m_{c0} ... p_{cn}+m_{cn})$

A component's performance effect is the product of its change in margin (measured in bps) times its weight in the portfolio in the prior period:

$p_c = \Delta Margin*Weight_{t-1}$

A component's mix effect is the product of its change in weight times the difference between the component's margin and the portfolio's total margin in the current period:

$m_c = \Delta Weight*(ComponentMargin_t-TotalMargin_t)$

Now that I've butchered the math notation, let's build a simple example using Python and Pandas:

# The Code

Let's say we're running a taco truck, we've just finished up Q1, and it's time to compare this quarter's performance to last year's Q1. Also, we bucket our sales into three product categories: Tacos, Sides, and Drinks. Let's import pandas and start with a dataframe:

```python
import pandas as pd

# sales and cost figures for our taco truck
d = {
    'category': ['Tacos', 'Sides', 'Drinks'],
    'revenue_t1': [20000, 10000, 5000],
    'cost_t1': [16600, 7800, 4250],
    'revenue_t0': [15000, 15000, 5000],
    'cost_t0': [12450, 12000, 3400],
    }

df = pd.DataFrame(data=d)
```

Note: In the real world, instead of hard-coding, we'd start by reading something like a csv or database table into the dataframe. Some cleanup, remapping, aggregation, etc. will likely be involved. The five key ingredients (columns) you must produce are:

1. Some categorization or bucketing dimension to slice your "portfolio" (virtually anything, e.g. product SKUs, product rollups, sales channels, geographical regions, etc - this will work at any level of detail!)
2. Revenue for current time period
3. Revenue for prior time period
4. Profit for current time period
5. Profit for prior time period

From there, we'll run these calculations:

```python
# calculating columns for gross profit, profit ratio, 
# sales weight, and period-over-period diffs
df['profit_t1'] = df['revenue_t1'] - df['cost_t1']
df['profit_t0'] = df['revenue_t0'] - df['cost_t0']
df['gm_t1'] = df['profit_t1'] / df['revenue_t1']
df['gm_t0'] = df['profit_t0'] / df['revenue_t0']
df['wt_t1'] = df['revenue_t1'] / df['revenue_t1'].sum()
df['wt_t0'] = df['revenue_t0'] / df['revenue_t0'].sum()
df['delta_gm_bps'] = ( df['gm_t1'] - df['gm_t0'] ) * 10000
df['delta_wt_bps'] = ( df['wt_t1'] - df['wt_t0'] ) * 10000

# calc current and prior overall profit ratios
tot_gm_t1 = df['profit_t1'].sum() / df['revenue_t1'].sum()
tot_gm_t0 = df['profit_t0'].sum() / df['revenue_t0'].sum()

# calc margin, mix, and total effects
df['perf_effect_bps'] = df['delta_gm_bps'] * df['wt_t0']
df['mix_effect_bps'] = df['mix_wt_bps'] * ( df['gm_t1'] - tot_gm_t1 )
df['total_effect_bps'] = df['perf_effect_bps'] + df['mix_effect_bps']

print(df)
```

## Result:

| category | ... | perf_effect_bps | mix_effect_bps | total_effect_bps |
| ---- | ---- | ---- | ---- | ---- |
| Tacos | ... | 0.000000 | -16.326531 | -16.326531 |
| Sides | ... | 85.714286 | -55.102041 | 30.612245 |
| Drinks | ... | -242.857143 | -0.000000 | -242.857143 |

We can sum each column to produce the total effects: 

| category | ... | perf_effect_bps | mix_effect_bps | total_effect_bps |
| ---- | ---- | ---- | ---- | ---- |
| Total | ... | -157.142857 | -71.428571 | -228.571428 |

Bonus step: if you like to tie out (and since we're doing FP&A, I know you do), you can spot check the total effect output back to the total GM change between periods:

```python
tieout_a = df['total_effect_bps'].sum()
tieout_b = ( tot_gm_t1 - tot_gm_t0 ) * 10000
tieout_check = tieout_a - tieout_b

print("tieout_a: ", tieout_a)
print("tieout_b: ", tieout_b)
print("CHECK DIFF: ", tieout_check)
```

```
tieout_a:  -228.57142857142864
tieout_b:  -228.57142857142853
CHECK DIFF:  -1.1368683772161603e-13
```

The small residual is an unavoidable tradeoff of geometric vs arithmetic averaging methodology, and it shouldn't pose any issue except in cases of extreme movements, if I understood the professor and CFA paper correctly. I lack the mathematical constitution to dig much further than that at the moment, especially since it's never prevented me from tying out to within a few cents.

# The Rationale

All of this fancy math is useless unless you understand why it works, and more importantly can discuss it fluently to look very smart to your CFO, so let's examine the two types of contributions:

## Performance Contributions

This statistic measures the amount of change in the overall margin that we can **attribute to the change in this particular component's margin**. It's calculated as the product of two factors:

1) **Component's change in margin** - The further a component's margin moves up or down, the larger impact it will have on our overall margin. Pretty self-explanatory.
2) **Component's sales weighting from the prior period** - The margin growth of a component representing a larger proportion of the sales mix will have a larger impact on our overall margin; this factor ensures that. And by using the state of the sales mix in the prior period, we're able to control other variables to isolate and measure **only the profitability change for this component**, ceteris paribus.

This "performance contribution" stat is answering the question:

> Holding the sales mix constant, how impactful was this component's margin change?

## Mix Contributions

This statistic measures the amount of change in the overall margin that we can **attribute to the change in this particular component's weight in the sales mix**. It's calculated as the product of two factors:

1) **Component's change in sales weighting** - as a component climbs the ranks and acquires a larger portion of the sales mix, its margin will have a stronger impact on the total, and likewise a shrinking component will have a weaker impact. Still, the impact could be positive or negative, which brings us to...
2) **The difference between the component and aggregate margins** - factors in the relative profitability of this component versus the whole portfolio. A super-profitable (or unprofitable) component will make larger impacts on our bottom line when it moves up or down in the sales mix, versus another one closer to average profitability.

This "mix contribution" stat is answering the question:

> How impactful was this component's movement in the sales mix, considering its relative profitability?

# Final Thoughts

As noted earlier, you can (and should) do this analysis on multiple levels of detail, and on as many dimensions as you like. You'll be measuring not just the Taco product category's contribution to the bottom line, but also the Carne Asada Taco SKU's contribution to the Taco category. You'll measure the performance effect of Truck A vs the mix effect of Truck C. Any dimension or column which allows you to slice your revenue can be used.

With a little setup work, and a few practice runs, you'll be quickly and accurately picking out product lines, regions, and channels requiring attention.

[^cfapaper]: [CFA Institute Paper](https://www.cfainstitute.org/-/media/documents/book/rf-lit-review/2019/rflr-performance-attribution.ashx) ("Selection effect" = "performance contribution effect",  "Allocation effect" = "mix contribution effect")
