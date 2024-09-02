---
layout: post
title: "Expected Strike Difference -- A simple catcher framing metric"
categories:
- Baseball
tags: []
comments: []
---

In this post I'll build a simple catcher framing metric I'm calling Expected Strike Difference.
The approach is similar to some other recent posts of mine like [this one](/baseball/2024/03/03/team-fielding.html).
Using [pybaseball]((https://github.com/jldbc/pybaseball)) Statcast data I'll train a classifier to predict whether a pitch will be called a ball or a strike based on the location where it crosses the plate and the measured transverse velocities.
Then I'll go through every pitch caught by each catcher and count up the differences between the predicted strikes and called strikes.
This difference theoretically represents the number of extra strikes stolen (or lost) compared to an average catcher.

# The Data
First the basics.
In the Statcast coordinate system, x is inside-outside in the strikezone with the first-base side positive, y is toward-away from the catcher with the field side positive, z is up-down with up positive.
![Statcast coordinate system](/assets/img/2024/08/coordinates.png){: .center width="40%"}

Here's what the observed strike zone looks like:
![Strikezone heatmap](/assets/img/2024/08/x_z.png){: .center width="60%"}

I normalized the pitch location such that the plate spans from -1 to 1 in each dimension.
That's what the "norm" means in the axis labels. 
The unit here is half plate-widths since -1 to 1 is a full width.

The color represents the probability of a pitch being called a strike based on location.
The strikezone is a bit wider than the nominal 17 inches of the plate.
Most of this is due to the width of the ball itself (about 3 inches or or 0.17 plate widths).

Next, here a few fun details I found and tried to account for.

## Batter handedness
I noticed that the strikezone looks a bit different for left handed and right handed hitters.
Specifically, it seems to be offset a little bit away from the hitter. 
I decided to correct for this by reversing the x direction of the strike zone for lefties.

{: .center"}
![Strikezone x coordinate uncorrected for handedness](/assets/img/2024/08/handedness.png){: width="45%"}
![Strikezone x coordinate corrected for handedness](/assets/img/2024/08/handedness_norm.png){: width="45%"}

These plots show the just the x coordinate of the strike zone for lefties and righties.
In the left plot, the yellow band is offset to the negative-x side for lefties compared to righties.
After correcting for handedness by reversing x for lefties, this offset is mostly gone in the right plot.

## Two strikes, three balls
I also noticed that umpires are less likely to call a marginal pitch a strike when there are two strikes.
I've heard this called "omission bias" and it is a well documented effect in strike calling.
Basically, umps are a bit adverse to calling strike three or ball four and directly ending the at-bat.

![](/assets/img/2024/08/two_strikes.png){: .center width="60%"}

This plot shows a skinnier yellow band when there are two strikes and less than three balls.
This indicates that the effective strike zone is smaller in that situation.

Similarly, they are less likely to call a ball with three balls and less than two strikes.

## Pitch movement

My intiution was that the transverse velocity (the left-right and up-down velocity) of the pitch will have an influence on the umpire's call.
For example, I expected a pitch at the bottom of the zone to have a better chance of being called a ball if it is breaking sharply downward.
I was not able to observe a clear effect like this in the data so I left it out of this model.

## The final result
Ultimately, the features I used were:
- Normalized and handedness-corrected x position when crossing the plate
    - Runs from -1.0 on the outside edge of the zone to 1.0 on the inside edge
- Normalized z position when crossing the plate
    - Runs from -1.0 on the low edge of the zone to 1.0 on the high edge
- Two strikes and less than three balls
    - Set to 1.0 if there are two strikes and less than three balls, 0.0 otherwise
- Three balls and less than two strikes
    - Set to 1.0 if there are three balls and less than two strikes, 0.0 otherwise

Here's what the data processing code looks like if you're curious.
It uses PyBaseball and Polars.
```python
PLATE_WIDTH_IN = 17
PLATE_HALF_WIDTH_FT = PLATE_WIDTH_IN / 2 / 24
balls_strikes = (
    pl.from_pandas(statcast(start_dt="2023-04-01", end_dt= "2024-04-01"))
    .filter(
        pl.col('description').is_in(['called_strike', 'ball'])
    )
    .select(
        pl.col("fielder_2").alias("catcher_id"),
        (pl.col("description") == "called_strike").cast(int).alias("strike"),
        "plate_x", # x location at plate in feet from center
        "plate_z", # z location at plate in feet from ground
        "sz_bot", # bottom of strike zone for batter in feet from ground
        "sz_top", # top of strike zone for batter in feet from ground
        (pl.col("sz_top") - pl.col("sz_bot")).alias("sz_height"),
        (pl.col("strikes") == 2).alias("two_strikes"),
        (pl.col("balls") == 3).alias("three_balls"),
        (1 - (2 * (pl.col("stand") == "L"))).alias("handedness_factor"), # -1 for lefties, 1 for righties
    )
    .with_columns(
        ((pl.col("plate_z") - pl.col("sz_bot")) / pl.col("sz_height") * 2 - 1).alias("plate_z_norm"), # plate_z from -1 to 1       
        (pl.col("plate_x") / PLATE_HALF_WIDTH_FT).alias("plate_x_norm"), # plate_x from -1 to 1
        (pl.col("two_strikes") & ~pl.col("three_balls")).alias("two_strikes_and_not_three_balls"),
        (~pl.col("two_strikes") & pl.col("three_balls")).alias("three_balls_and_not_two_strikes"),
    )
    .drop_nulls()
)
```

# K Nearest Neighbors
I used a method called [K Nearest Neighbors](https://en.wikipedia.org/wiki/K-nearest_neighbors_algorithm)(KNN) to establish the expected strike probability for each pitch.
For each pitch, I look up the 25 most similar pitches.
The set of similar pitches is called the "neighborhood" and (in this case) it's determined based on distance when plotted on a graph.
For example, the neighborhood might look like this for a pitch near the inside corner for a right handed hitter.

![Inside corner neighborhood](/assets/img/2024/08/neighborhood.png){: .center width="60%"}

The expected strike probability is defined as the the fraction of pitches in the neighborhood that were called strikes.
In the data I used, the neighborhoods are much smaller than the one depicted.
There are so many pitches that the 25 nearest ones are close together.

Two of the features I used (two strikes less than three balls, three balls less than two strikes) have values that are either 0 or 1.
These have an interesting effect on the neighborhood calculation.
Because of the relative scales involved, neighborhoods will almost always include only pitches with the same value for these discrete features.

![Inside corner neighborhood](/assets/img/2024/08/neighborhood_cat.png){: .center width="60%"}

# Results
With this KNN method it's easy to evaluate catcher.
For each catcher, look at the pitches they recieved.
For each pitch find the neighborhood pitches and calculate the fraction that were called strikes. 
Add up these fractions to determine the number of expected strikes.
Compare that sum to the total number of strike calls they got.

Here's how the numbers come out for 2023
<table border="0" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>Name</th>
      <th>Avg expected strike difference<br>(per 100 pitches)</th>
      <th>Total expected strike difference</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Francisco Álvarez</td>
      <td>1.52</td>
      <td>125.72</td>
    </tr>
    <tr>
      <td>Patrick Bailey</td>
      <td>1.94</td>
      <td>124.24</td>
    </tr>
    <tr>
      <td>Austin Hedges</td>
      <td>2.33</td>
      <td>118.20</td>
    </tr>
    <tr>
      <td>Jonah Heim</td>
      <td>0.83</td>
      <td>77.64</td>
    </tr>
    <tr>
      <td>Jose Trevino</td>
      <td>1.87</td>
      <td>72.00</td>
    </tr>
    <tr>
      <td>Víctor Caratini</td>
      <td>1.61</td>
      <td>68.20</td>
    </tr>
    <tr>
      <td>Cal Raleigh</td>
      <td>0.77</td>
      <td>64.00</td>
    </tr>
    <tr>
      <td>Jake Rogers</td>
      <td>0.87</td>
      <td>60.76</td>
    </tr>
    <tr>
      <td>Alejandro Kirk</td>
      <td>0.83</td>
      <td>58.56</td>
    </tr>
    <tr>
      <td>William Contreras</td>
      <td>0.70</td>
      <td>58.24</td>
    </tr>
    <tr>
      <td>Adley Rutschman</td>
      <td>0.70</td>
      <td>56.96</td>
    </tr>
    <tr>
      <td>Seby Zavala</td>
      <td>1.22</td>
      <td>54.20</td>
    </tr>
    <tr>
      <td>Kyle Higashioka</td>
      <td>0.99</td>
      <td>53.88</td>
    </tr>
    <tr>
      <td>Jason Delay</td>
      <td>1.11</td>
      <td>47.48</td>
    </tr>
    <tr>
      <td>Cam Gallagher</td>
      <td>1.36</td>
      <td>46.28</td>
    </tr>
    <tr>
      <td>Christian Vázquez</td>
      <td>0.69</td>
      <td>43.68</td>
    </tr>
    <tr>
      <td>Tucker Barnhart</td>
      <td>1.54</td>
      <td>40.64</td>
    </tr>
    <tr>
      <td>Yasmani Grandal</td>
      <td>0.66</td>
      <td>39.64</td>
    </tr>
    <tr>
      <td>Austin Wynns</td>
      <td>0.95</td>
      <td>31.72</td>
    </tr>
    <tr>
      <td>Joey Bart</td>
      <td>1.25</td>
      <td>25.00</td>
    </tr>
    <tr>
      <td>Freddy Fermin</td>
      <td>0.58</td>
      <td>24.64</td>
    </tr>
    <tr>
      <td>Miguel Amaya</td>
      <td>0.97</td>
      <td>24.28</td>
    </tr>
    <tr>
      <td>Ben Rortvedt</td>
      <td>1.15</td>
      <td>23.64</td>
    </tr>
    <tr>
      <td>Nick Fortes</td>
      <td>0.34</td>
      <td>23.24</td>
    </tr>
    <tr>
      <td>Gary Sánchez</td>
      <td>0.50</td>
      <td>23.00</td>
    </tr>
    <tr>
      <td>Travis D'arnaud</td>
      <td>0.47</td>
      <td>22.24</td>
    </tr>
    <tr>
      <td>Sean Murphy</td>
      <td>0.30</td>
      <td>21.56</td>
    </tr>
    <tr>
      <td>Brian Serven</td>
      <td>2.54</td>
      <td>19.44</td>
    </tr>
    <tr>
      <td>Blake Sabol</td>
      <td>0.59</td>
      <td>19.16</td>
    </tr>
    <tr>
      <td>René Pinto</td>
      <td>0.73</td>
      <td>17.80</td>
    </tr>
    <tr>
      <td>Austin Barnes</td>
      <td>0.43</td>
      <td>15.68</td>
    </tr>
    <tr>
      <td>Tomás Nido</td>
      <td>0.93</td>
      <td>13.16</td>
    </tr>
    <tr>
      <td>Tyler Heineman</td>
      <td>1.18</td>
      <td>11.92</td>
    </tr>
    <tr>
      <td>Mike Zunino</td>
      <td>0.38</td>
      <td>9.92</td>
    </tr>
    <tr>
      <td>Ali Sánchez</td>
      <td>6.04</td>
      <td>9.48</td>
    </tr>
    <tr>
      <td>Austin Wells</td>
      <td>0.51</td>
      <td>9.24</td>
    </tr>
    <tr>
      <td>Reese Mcguire</td>
      <td>0.17</td>
      <td>6.56</td>
    </tr>
    <tr>
      <td>Danny Jansen</td>
      <td>0.12</td>
      <td>5.68</td>
    </tr>
    <tr>
      <td>Roberto Pérez</td>
      <td>2.07</td>
      <td>5.28</td>
    </tr>
    <tr>
      <td>Omar Narváez</td>
      <td>0.16</td>
      <td>5.24</td>
    </tr>
    <tr>
      <td>Alex Jackson</td>
      <td>2.65</td>
      <td>5.04</td>
    </tr>
    <tr>
      <td>Dillon Dingler</td>
      <td>3.70</td>
      <td>4.88</td>
    </tr>
    <tr>
      <td>Logan Porter</td>
      <td>0.71</td>
      <td>4.84</td>
    </tr>
    <tr>
      <td>Will Smith</td>
      <td>0.05</td>
      <td>4.16</td>
    </tr>
    <tr>
      <td>Bo Naylor</td>
      <td>0.08</td>
      <td>3.80</td>
    </tr>
    <tr>
      <td>Sandy León</td>
      <td>0.42</td>
      <td>3.32</td>
    </tr>
    <tr>
      <td>Anthony Bemboom</td>
      <td>0.76</td>
      <td>3.28</td>
    </tr>
    <tr>
      <td>Mj Melendez</td>
      <td>0.55</td>
      <td>3.20</td>
    </tr>
    <tr>
      <td>Caleb Hamilton</td>
      <td>2.08</td>
      <td>3.00</td>
    </tr>
    <tr>
      <td>Andrew Knapp</td>
      <td>8.67</td>
      <td>2.60</td>
    </tr>
    <tr>
      <td>Grant Koch</td>
      <td>3.45</td>
      <td>2.52</td>
    </tr>
    <tr>
      <td>Payton Henry</td>
      <td>1.72</td>
      <td>2.20</td>
    </tr>
    <tr>
      <td>Israel Pineda</td>
      <td>8.17</td>
      <td>1.88</td>
    </tr>
    <tr>
      <td>Carlos Narvaez</td>
      <td>7.27</td>
      <td>1.60</td>
    </tr>
    <tr>
      <td>Tres Barrera</td>
      <td>2.47</td>
      <td>1.48</td>
    </tr>
    <tr>
      <td>Henry Davis</td>
      <td>0.09</td>
      <td>0.48</td>
    </tr>
    <tr>
      <td>Austin Allen</td>
      <td>4.36</td>
      <td>0.48</td>
    </tr>
    <tr>
      <td>Chris Okey</td>
      <td>0.69</td>
      <td>0.40</td>
    </tr>
    <tr>
      <td>Zack Collins</td>
      <td>0.28</td>
      <td>0.36</td>
    </tr>
    <tr>
      <td>Rob Brantly</td>
      <td>1.27</td>
      <td>0.28</td>
    </tr>
    <tr>
      <td>Mickey Gasper</td>
      <td>1.41</td>
      <td>0.24</td>
    </tr>
    <tr>
      <td>Dom Nuñez</td>
      <td>2.67</td>
      <td>0.16</td>
    </tr>
    <tr>
      <td>David Bañuelos</td>
      <td>-0.00</td>
      <td>-0.00</td>
    </tr>
    <tr>
      <td>Hunter Goodman</td>
      <td>-0.52</td>
      <td>-0.12</td>
    </tr>
    <tr>
      <td>Joe Hudson</td>
      <td>-0.61</td>
      <td>-0.20</td>
    </tr>
    <tr>
      <td>Chuckie Robinson</td>
      <td>-0.93</td>
      <td>-0.28</td>
    </tr>
    <tr>
      <td>Mark Kolozsvary</td>
      <td>-0.34</td>
      <td>-0.48</td>
    </tr>
    <tr>
      <td>Aramis Garcia</td>
      <td>-0.60</td>
      <td>-0.52</td>
    </tr>
    <tr>
      <td>Jorge Alfaro</td>
      <td>-0.16</td>
      <td>-0.64</td>
    </tr>
    <tr>
      <td>Tyler Cropley</td>
      <td>-0.47</td>
      <td>-0.88</td>
    </tr>
    <tr>
      <td>Pedro Pagés</td>
      <td>-1.40</td>
      <td>-1.12</td>
    </tr>
    <tr>
      <td>Manny Piña</td>
      <td>-1.06</td>
      <td>-1.32</td>
    </tr>
    <tr>
      <td>Jhonny Pereda</td>
      <td>-9.20</td>
      <td>-1.84</td>
    </tr>
    <tr>
      <td>César Salazar</td>
      <td>-0.46</td>
      <td>-2.16</td>
    </tr>
    <tr>
      <td>Mitch Garver</td>
      <td>-0.12</td>
      <td>-2.20</td>
    </tr>
    <tr>
      <td>Kyle Mccann</td>
      <td>-1.28</td>
      <td>-2.36</td>
    </tr>
    <tr>
      <td>Drew Romo</td>
      <td>-10.21</td>
      <td>-2.96</td>
    </tr>
    <tr>
      <td>Korey Lee</td>
      <td>-0.17</td>
      <td>-3.28</td>
    </tr>
    <tr>
      <td>James Mccann</td>
      <td>-0.10</td>
      <td>-4.08</td>
    </tr>
    <tr>
      <td>Brian O'keefe</td>
      <td>-1.11</td>
      <td>-4.84</td>
    </tr>
    <tr>
      <td>Michael Pérez</td>
      <td>-3.89</td>
      <td>-5.64</td>
    </tr>
    <tr>
      <td>Chad Wallach</td>
      <td>-0.19</td>
      <td>-6.48</td>
    </tr>
    <tr>
      <td>Drew Millas</td>
      <td>-0.94</td>
      <td>-6.68</td>
    </tr>
    <tr>
      <td>Chadwick Tromp</td>
      <td>-1.52</td>
      <td>-6.80</td>
    </tr>
    <tr>
      <td>Eric Haase</td>
      <td>-0.20</td>
      <td>-7.96</td>
    </tr>
    <tr>
      <td>Luis Torrens</td>
      <td>-5.09</td>
      <td>-8.24</td>
    </tr>
    <tr>
      <td>Meibrys Viloria</td>
      <td>-4.40</td>
      <td>-8.44</td>
    </tr>
    <tr>
      <td>Iván Herrera</td>
      <td>-0.77</td>
      <td>-8.52</td>
    </tr>
    <tr>
      <td>Sam Huff</td>
      <td>-3.14</td>
      <td>-9.68</td>
    </tr>
    <tr>
      <td>Endy Rodríguez</td>
      <td>-0.28</td>
      <td>-9.76</td>
    </tr>
    <tr>
      <td>Carlos Pérez</td>
      <td>-1.34</td>
      <td>-13.24</td>
    </tr>
    <tr>
      <td>Curt Casali</td>
      <td>-0.66</td>
      <td>-13.88</td>
    </tr>
    <tr>
      <td>Gabriel Moreno</td>
      <td>-0.18</td>
      <td>-15.28</td>
    </tr>
    <tr>
      <td>Luis Campusano</td>
      <td>-0.50</td>
      <td>-16.40</td>
    </tr>
    <tr>
      <td>Matt Thaiss</td>
      <td>-0.33</td>
      <td>-16.84</td>
    </tr>
    <tr>
      <td>Tyler Soderstrom</td>
      <td>-1.57</td>
      <td>-17.32</td>
    </tr>
    <tr>
      <td>Carson Kelly</td>
      <td>-0.51</td>
      <td>-18.32</td>
    </tr>
    <tr>
      <td>Christian Bethancourt</td>
      <td>-0.34</td>
      <td>-22.68</td>
    </tr>
    <tr>
      <td>Brett Sullivan</td>
      <td>-1.48</td>
      <td>-27.72</td>
    </tr>
    <tr>
      <td>Yainer Diaz</td>
      <td>-0.70</td>
      <td>-28.00</td>
    </tr>
    <tr>
      <td>Jacob Stallings</td>
      <td>-0.56</td>
      <td>-30.04</td>
    </tr>
    <tr>
      <td>David Fry</td>
      <td>-3.01</td>
      <td>-30.60</td>
    </tr>
    <tr>
      <td>Austin Nola</td>
      <td>-1.00</td>
      <td>-31.08</td>
    </tr>
    <tr>
      <td>Tom Murphy</td>
      <td>-1.17</td>
      <td>-32.20</td>
    </tr>
    <tr>
      <td>Garrett Stubbs</td>
      <td>-1.31</td>
      <td>-33.56</td>
    </tr>
    <tr>
      <td>Ryan Jeffers</td>
      <td>-0.63</td>
      <td>-35.12</td>
    </tr>
    <tr>
      <td>Salvador Pérez</td>
      <td>-0.55</td>
      <td>-35.52</td>
    </tr>
    <tr>
      <td>Yan Gomes</td>
      <td>-0.52</td>
      <td>-36.60</td>
    </tr>
    <tr>
      <td>José Herrera</td>
      <td>-1.35</td>
      <td>-38.56</td>
    </tr>
    <tr>
      <td>Carlos Pérez</td>
      <td>-1.82</td>
      <td>-45.64</td>
    </tr>
    <tr>
      <td>Francisco Mejía</td>
      <td>-1.47</td>
      <td>-45.76</td>
    </tr>
    <tr>
      <td>Luke Maile</td>
      <td>-1.05</td>
      <td>-47.04</td>
    </tr>
    <tr>
      <td>Andrew Knizner</td>
      <td>-1.01</td>
      <td>-50.92</td>
    </tr>
    <tr>
      <td>Willson Contreras</td>
      <td>-0.75</td>
      <td>-52.12</td>
    </tr>
    <tr>
      <td>Riley Adams</td>
      <td>-1.84</td>
      <td>-57.80</td>
    </tr>
    <tr>
      <td>Connor Wong</td>
      <td>-0.75</td>
      <td>-60.68</td>
    </tr>
    <tr>
      <td>Logan O'hoppe</td>
      <td>-1.57</td>
      <td>-65.24</td>
    </tr>
    <tr>
      <td>Tyler Stephenson</td>
      <td>-1.13</td>
      <td>-69.72</td>
    </tr>
    <tr>
      <td>Elías Díaz</td>
      <td>-1.07</td>
      <td>-96.52</td>
    </tr>
    <tr>
      <td>Keibert Ruiz</td>
      <td>-1.06</td>
      <td>-97.56</td>
    </tr>
    <tr>
      <td>Shea Langeliers</td>
      <td>-1.03</td>
      <td>-99.08</td>
    </tr>
    <tr>
      <td>Martín Maldonado</td>
      <td>-1.41</td>
      <td>-126.52</td>
    </tr>
    <tr>
      <td>J. T. Realmuto</td>
      <td>-1.18</td>
      <td>-126.96</td>
    </tr>
  </tbody>
</table>

This correlates well with Fangraphs and Baseball Savant catcher framing statistics.
![Correlation with other statistics](/assets/img/2024/08/corr.png){: .center width="60%"}