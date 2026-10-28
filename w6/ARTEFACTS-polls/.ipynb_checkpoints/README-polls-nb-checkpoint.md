---
jupytext:
  custom_cell_magics: kql
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.19.5
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# US polls with altair

+++

```{admonition} download the zip
:class: warning

to run this code on your own laptop,
{download}`start with downloading the zip file<./ARTEFACTS-polls.zip>`
```

```{code-cell} ipython3
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

import geopandas as gpd

#activate this if running under jlab
%matplotlib ipympl
```

## 1. loading presidential averages

538.com - actually <http://fivethirtyeight.com> - was a site that published data about the US presidential election  
the site is no longer online, so we use a copy of its **presidential averages** that is stored in `data/DATA.csv`

load this file into a dataframe `df`; make sure to
- be careful about the types of the columns, in particular `date`, that must be a proper timestamp
- notice that the first column is a leftover index, and so ensure it does not end up as a column in `df`

```{code-cell} ipython3
:tags: [level_basic]

df = pd.read_csv("data/DATA.csv")
df.dtypes
```

```{code-cell} ipython3
df["date"] = pd.to_datetime(df.date)
df.dtypes
```

```{code-cell} ipython3
df.head()
```

```{code-cell} ipython3
df.columns
```

```{code-cell} ipython3
del df['Unnamed: 0']
df.head()
```

## 2. plotting

And what we want to do is to plot the average of the polls for each candidate.  
In other words, you should obtain something like this - we will arbitrarily focus on the 2024 year only

```{image} media/polls-over-time-2024.png
:width: 800px
:align: center
```

```{code-cell} ipython3
:tags: [level_basic]

df2024 = df.set_index('date').loc["2024"].reset_index()
df2024.pivot_table(index="date", columns="candidate", values="pct_estimate").plot()
```

## 3. use interactivity

using the interactive view (if under Jupyter Lab) make sure to use `%matplotlib ipympl`, and zoom into the figure and retrieve
- the date for the last data about Joe Biden
- the date for the first data about Kamala Harris

+++ {"tags": ["level_basic"]}

- The date for the last data about Joe Biden is 2024-07-21
- The date for the first data about Kamala Harris is 2024-07-24

+++

also write a line of code to compute this second date

```{code-cell} ipython3
:tags: [level_basic]

# your code
by_candidate = df.groupby(by="candidate")
Harris = by_candidate.get_group('Harris')
first_harris_date = Harris["date"].min()
print("First Harris date is", first_harris_date.strftime("%d/%m/%Y"))

Biden = by_candidate.get_group('Biden')
last_biden_date = Biden["date"].max()
print("Last Biden date is", last_biden_date.strftime("%d/%m/%Y"))
```

## 4. race end

from this part we will focus on the period after `first_harris_date`
1. how many candidates are still in the data ?
1. make sure to keep only the 2 most famous ones
1. how many data points do we have now for each of them ?

```{code-cell} ipython3
:tags: [level_basic]

df_reduced = df[ df["date"] >= first_harris_date ]
print(len(df_reduced["candidate"].unique()), " candidates are stil in the data")
df_reduced["candidate"].unique()
```

```{code-cell} ipython3
means = df_reduced.groupby(by="candidate")["pct_estimate"].mean()
top2 = means.nlargest(2).index
df_reduced = df_reduced[df_reduced["candidate"].isin(top2)]
df_reduced["candidate"].unique()
```

```{code-cell} ipython3
by_candidate = df_reduced.groupby(by="candidate")
by_candidate.size()
```

We now have 900 data points for each of them

+++

## 5: loading shapefiles with geopandas

```{code-cell} ipython3
# here's the typical nickname for geopandas

import geopandas as gpd
```

we want to plot this data on a geographic map; first we need a definition of the various US states; this comes from the US Census Bureau (file `us-states.zip`), and we have a copy of it under `data/`

we load this shapefile into a geopandas object

```{code-cell} ipython3
gdf = gpd.read_file("data/us-states.zip")
```

1. how many entries ?

```{code-cell} ipython3
:tags: [level_basic]

print("There are ", len(gdf), " entries")
```

2. what are the various column types

+++

as you can see, this is almost like a regular dataframe, except for the `geometry` column; which is a geographic entity, hence the term `geo-dataframe`

```{code-cell} ipython3
gdf.dtypes
```

## digression: using altair to produce a geographic viz

```{admonition} this may require an installation
:class: dropdown tip
of course you might have to install `altair`...  how do you go about doing that again ?
```

````{admonition} see also on altair and its use on geographic data
:class: dropdown warning

more details on this topic can be found here: 

- <https://altair-viz.github.io/user_guide/marks/geoshape.html>
- <https://idl.uw.edu/visualization-curriculum/altair_cartographic.html>
- <https://altair-viz.github.io/altair-tutorial/notebooks/09-Geographic-plots.html>
````

```{code-cell} ipython3
%pip install altair
```

```{code-cell} ipython3
import altair as alt

# this is for rendering altair charts within the notebook
alt.renderers.enable("html")
```

Here are a few examples of what we can do with this library

+++

### the simplest chart

```{code-cell} ipython3
# to show a geographic map from that geo-df
# (suboptimal, we'll improve that...)

# we could just write this and it would render
# alt.Chart(gdf).mark_geoshape()

# but we can also this more usual form

chart = (
    alt.Chart(gdf)
    .mark_geoshape()
)

chart.display()
```

### improving that view

+++

now in terms of presentation, it is a little suboptimal, let's improve this a bit

```{code-cell} ipython3
# we can improve by:
# (1) setting a width
# (2) using the albersUsa projection, which puts Alaska and Hawai under mainland
#     this way we use space more effectively
# NOTE that in the process Alaska is artificially shrunk down
#      it's actually the largest US state
(
    alt.Chart(gdf)
    .mark_geoshape()
    .properties(width=600)
    .project('albersUsa')
)
```

### using colors

+++

now, the initial geo-dataframe has some numeric values, that we can use to color the map !  
for example, there are `AWATER` and `ALAND`  
which we can interpret as meaning *area of water* (as in lakes) and *area of land* respectively

```{code-cell} ipython3
gdf.head(2)
```

and we can use one of these to color the different states  
for that we would use `encode()` like so

```{code-cell} ipython3
(
    alt.Chart(gdf)
    .mark_geoshape()
    .encode(
        color="ALAND:Q",          # Q stands for quantitative
    )
    .properties(width=600)
    .project('albersUsa')
)
```

### better with a legend

```{code-cell} ipython3
# or if we prefer, same result essentially
# but this way we can be more descriptive

(
    alt.Chart(gdf)
    .mark_geoshape()
    # legend
    .encode(
        color=alt.Color(field="ALAND", type="quantitative", title="land area")
    )
    # a bit narrower so the legend shows up
    .properties(width=600)
    .project('albersUsa')
)
```

### even better with tooltips

```{code-cell} ipython3
# also useful with altair, we can give a `tooltip` parameter to encode
# and this shows when your mouse hovers on a state

# NOTE: altair can only see the columns, not the index
# so as we want to show the name of the state, we need it in a regular column
# so you had made the choice to use the state as the index
# you may need to reset_index()

(
    alt.Chart(gdf)
    .mark_geoshape()
    .encode(
        color=alt.Color(field="AWATER", type="quantitative", title="water area"),
        # and we can show there anything from the table
        tooltip=["NAME", "ALAND"],
    )
    .properties(width=600)
    .project('albersUsa')
)
```

## our goal: geographic rendering

in this following we will produce a summary map, which looks like this:
- for all the states where we have data
- the color depicts the ratio between, Harris's and Trump's average scores over time
- also the tooltips allow to expose more details on the individual results

```{image} media/states-with-data.png
:width: 800px
:align: center
```

+++

## 6. first map

+++

1. for how many states do we have polls data ? which are they ?

```{code-cell} ipython3
:tags: [level_basic]

states = df_reduced["state"].dropna().unique()
print(f"Number of states: {len(states)}")
print(sorted(states))
```

2. for this data, compute a table with the state as index, and the two contestants polls (averaged)

```{code-cell} ipython3
:tags: [level_basic]

table = df_reduced.pivot_table(index="state", columns="candidate", values="pct_estimate", aggfunc="mean")
df_reduced.pivot_table(index="state", columns="candidate", values="pct_estimate", aggfunc="mean")
```

3. using what you've seen in the digression, and what you will gather if needed on the Internet,  produce the above map

```{code-cell} ipython3
:tags: [level_basic]

table["ratio"] = table["Harris"] / table["Trump"]
table = table.reset_index()   #"state" redevient une colonne normale
```

```{code-cell} ipython3
gdf_data = gdf.merge(table, left_on="NAME", right_on="state", how="inner")
```

```{code-cell} ipython3
(
    alt.Chart(gdf_data)
    .mark_geoshape()
    .encode(
        color=alt.Color(field="ratio", type="quantitative", title="Harris / Trump"),
        tooltip=["NAME", "Harris", "Trump", "ratio"],
    )
    .properties(width=600)
    .project("albersUsa")
)
```

## 7. outlining swing states

+++

from the graph above, we want to outline the swing states  
for example, we could come up with something like

```{image} media/swing-states-outlined.png
:width: 800px
:align: center
```

````{admonition} *hints*
:class: dropdown

- the method `pd.Series.isin()` might come in handy for this step
- also you will probably need to search a bit by yourself, outside of what we've covered in the digression
````

```{code-cell} ipython3
SWING_STATES = [
    'Nevada',
    'Arizona',
    'Wisconsin',
    'Michigan',
    'Pennsylvania',
    'Georgia',
    'North Carolina',
]
```

```{code-cell} ipython3
:tags: [level_basic]

swing = gdf_data[gdf_data["NAME"].isin(SWING_STATES)]
```

```{code-cell} ipython3
# calque 1 : la carte colorée de la 6.3
base = (
    alt.Chart(gdf_data)
    .mark_geoshape()
    .encode(
        color=alt.Color("ratio:Q", title="Harris / Trump"),
        tooltip=["NAME", "Harris", "Trump", "ratio"],
    )
)

# calque 2 : uniquement le contour des swing states
outline = (
    alt.Chart(swing)
    .mark_geoshape(fill=None, stroke="black", strokeWidth=2)
)
```

```{code-cell} ipython3
(
    alt.layer(base, outline)
    .properties(width=600)
    .project("albersUsa")
)
```

## 8. filling gaps (optional)

this is not very readable; instead we want to still get the missing states show up in gray,  
and to keep the outlining of the swing states, i.e.

```{image} media/all-states.png
:width: 800px
:align: center
```

```{admonition} tip
:class: dropdown tip
this one is less obvious, and you are allowed to use an IA to find the solution !
```

```{code-cell} ipython3
:tags: [level_basic]

background = (
    alt.Chart(gdf)
    .mark_geoshape(fill="lightgray", stroke="white")
    .encode(tooltip="NAME:N")
)
```

```{code-cell} ipython3
(
    alt.layer(background, base, outline)
    .properties(width=600)
    .project("albersUsa")
)
```
