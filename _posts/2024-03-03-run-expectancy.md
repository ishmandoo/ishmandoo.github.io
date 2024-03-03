---
layout: post
title: "Calculating Run Expectancy Tables"
categories:
- Baseball
tags: []
comments: []
---

Here's some simple code for building a run expectancy table based on Statcast data:
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
    pitch_data["runs_after_ab"] = (
        pitch_data["inning_final_bat_score"] - pitch_data["bat_score"]
    )

    # filter down to one row per at-bat
    ab_data = pitch_data[pitch_data["pitch_number"] == 1]

    # group by base/out state and calculate mean runs scored after that state
    return ab_data.groupby(["outs_when_up", "1b", "2b", "3b"])["runs_after_ab"].mean()
```

Here's what it looks like for 2021:
```python
print(run_expectancy("2021-04-01", "2022-04-01"))
---
outs_when_up  1b     2b     3b   
0             False  False  False    0.488405
                            True     1.237288
                     True   False    1.060241
                            True     2.141935
              True   False  False    0.907478
                            True     1.726829
                     True   False     1.45768
                            True     2.412121
1             False  False  False    0.256036
                            True     0.901515
                     True   False     0.62087
                            True     1.247956
              True   False  False     0.54761
                            True     1.166667
                     True   False    0.863067
                            True      1.56599
2             False  False  False    0.098779
                            True     0.359133
                     True   False    0.327983
                            True     0.553738
              True   False  False    0.227557
                            True     0.490132
                     True   False    0.433183
                            True     0.775794
```