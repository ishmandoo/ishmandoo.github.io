---
layout: post
title: "Calculating Run Expectancy Tables"
categories:
- Baseball
tags: []
comments: []
---

Below is some simple code for building a run expectancy table based on Statcast data.
A run expectancy table gives the average number of runs scored after each base/out state.
For example, with runners on 1st and 2nd and one out, the table gives the average number of runs that scored.
```python
import pandas as pd
from pybaseball import statcast


def run_expectancy(start_date: str, end_date: str) -> pd.Series:
    """
    Returns a run expectancy table based on Statcast data from `start_date` to `end_date`
    """
    pitch_data: pd.DataFrame = statcast(start_dt=start_date, end_dt=end_date)

    # create columns for whether a runner is on each base
    for base in ("1b", "2b", "3b"):
        pitch_data[base] = pitch_data[f"on_{base}"].notnull()

    pitch_data["inning_final_bat_score"] = pitch_data.groupby(
        ["game_pk", "inning", "inning_topbot"]
    )["post_bat_score"].transform("max")

    # filter down to one row per at-bat
    ab_data = pitch_data[pitch_data["pitch_number"] == 1]

    ab_data["runs_after_ab"] = (
        ab_data["inning_final_bat_score"] - ab_data["bat_score"]
    )

    # group by base/out state and calculate mean runs scored after that state
    return ab_data.groupby(["outs_when_up", "1b", "2b", "3b"])["runs_after_ab"].mean()
```

Here's what it looks like for 2021:
```python
print(run_expectancy("2021-04-01", "2021-12-01"))
---
outs_when_up  1b     2b     3b   
0             False  False  False    0.507303
                            True     1.393333
                     True   False    1.135049
                            True     2.107407
              True   False  False    0.916202
                            True     1.745745
                     True   False    1.523861
                            True     2.446313
1             False  False  False    0.264921
                            True     0.958691
                     True   False    0.684807
                            True     1.409165
              True   False  False    0.534543
                            True     1.126154
                     True   False    0.923244
                            True      1.68007
2             False  False  False    0.101856
                            True     0.385488
                     True   False    0.324888
                            True     0.600758
              True   False  False    0.228621
                            True     0.493186
                     True   False    0.451022
                            True     0.825928
```