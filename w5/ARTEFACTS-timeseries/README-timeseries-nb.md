---
jupytext:
  encoding: '# -*- coding: utf-8 -*-'
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

# time series, the basics

```{admonition} to download
:class: warning

to work on this assignment locally, {download}`start by downloading the zip<./ARTEFACTS-timeseries.zip>`
```

In this assignment we study **cycling in Paris**, using the hourly counts of one of the
bike counters that the city has installed in its streets.

This is a *first contact* with time series in pandas; we will practice

* **parsing** strings into timestamps, and dealing with time zones
* the **`.dt` accessor**, to extract the hour, the day of the week, etc.
* **slicing** a dataframe on its time index
* **`resample()`**, to aggregate by day, week, month
* and finally **joining** with another time series, daily weather data

the companion assignment *time series: events* deals with series that are **not** on a regular grid

here is the kind of pictures we will end up with

<img src="media/hourly-profile.svg" width=500px>

```{code-cell} ipython3
import pandas as pd
import matplotlib.pyplot as plt

print(f"using pandas {pd.__version__}")

# time series look better in a wide figure
plt.rcParams["figure.figsize"] = (10, 4)
```

```{code-cell} ipython3
# optional
import itables
itables.init_notebook_mode()
```

## 0. the data

+++

there are two files in the `data/` folder

| file | one row per | columns |
|-|-|-|
| `bike-counter.csv` | hour | `timestamp`, `count` (number of bikes that went through the counter) |
| `weather.csv` | day | `jour`, `temperature` (°C, daily mean), `rain_mm` (daily total) |

the counter is on the *voie Georges Pompidou*, along the Seine; it counts the bikes going in one direction only  
we start with the first one

+++

## 1. parsing the timestamps

+++

### loading

+++

1. load `data/bike-counter.csv` into a dataframe named `df`
1. look at the first rows and at the types of the columns - what is the type of the `timestamp` column ?
1. the name `count` is unfortunate, because `df.count` is a method of the dataframe;  
   rename that column into `n_bikes`

```{code-cell} ipython3
:tags: [level_basic]

from IPython.display import display

df = pd.read_csv("data/bike-counter.csv")
display(df.head(10))

df.dtypes
```

The type of the timestamp column is string

```{code-cell} ipython3
df["n_bikes"] = df["count"]
del df["count"]
df.head(10)

# alternative: df.rename(columns= {"count":"n_bikes"})
```

### converting

+++

we want to turn the `timestamp` column into actual timestamps

1. use `pd.to_datetime()` for that; what do you get ? read the message carefully
1. look at the strings in the data: apart from the date and the hour, what else is there ?  
   how many different values are there for that part ? can you guess why ?

```{code-cell} ipython3
:tags: [level_basic]

df["timestamp"] = pd.to_datetime(df.timestamp) # format ISO 8601
display(df.dtypes)
df.head()
```

We get the message: 
ValueError: Mixed timezones detected. Pass utc=True in to_datetime or tz='UTC' in DatetimeIndex to convert to a common timezone.

There are the letter T and a time offset apart from the date and the hour.

+++

:::{admonition} hint
:class: tip dropdown

the last 6 characters of a string can be obtained with `df.timestamp.str[-6:]`  
and then you can use `value_counts()`
:::

```{code-cell} ipython3
offset = df.timestamp.str[-6:]
offset.value_counts()
```

Apart from the date and the hour, the timestamp strings contain a time-zone offset. There are two different vales: +02.00 and +01.00. 
This is because the dataset spans both summer time and winter time.

+++

### converting, again

+++

pandas has a hard time with a column that mixes several time zone offsets  
the way out is to ask for everything to be expressed in UTC, using the `utc=True` option

1. redo the conversion with that option, and store the result back into the `timestamp` column;  
   what is the dtype now ?
1. we are interested in the **local time** in Paris, since this is when people go to work  
   convert that column to the timezone `"Europe/Paris"`  
   you will need to find the right method in the `.dt` accessor (or do a search on `tz_convert`)

```{code-cell} ipython3
:tags: [level_basic]

df["timestamp"] = pd.to_datetime(df.timestamp, utc=True)
# utc = true convertie la date dans le fuseau horaire UTC (Coordinated Universal Time)
print("The type now is ", df.timestamp.dtype)
df.head()
```

```{code-cell} ipython3
df.timestamp = df["timestamp"].dt.tz_convert("Europe/Paris")
df.head()
```

## 2. the `.dt` accessor

+++

now that `timestamp` has a proper type, the `.dt` accessor gives us access to all the details  
add these columns to `df`

| column | content |
|-|-|
| `hour` | the hour of the day, 0 to 23 |
| `weekday` | the day of the week, as a string like `"Monday"` |
| `is_weekend` | a boolean, `True` on Saturday and Sunday |
| `month` | the month, as a number 1 to 12 |

```{code-cell} ipython3
:tags: [level_basic]

df["hour"] = df.timestamp.dt.hour
df["weekday"] = df.timestamp.dt.day_name()
df["month"] = df.timestamp.dt.month
df["is_weekend"] = df["weekday"] == (("Saturday") or ("Sunday"))
print(df["weekday"].unique())
display(df["weekday"].head())
df["is_weekend"].head()
```

:::{admonition} hint
:class: tip dropdown

* `.dt.hour`, `.dt.month` are attributes
* `.dt.day_name()` is a method, and there is also an attribute `.dt.dayofweek` that returns a number with 0 for Monday
:::

+++

### the hourly profile

+++

1. what is the average number of bikes **per hour of the day** ? use a `groupby` and plot the result with `.plot()`  
   what are the busiest hours ? does that match your expectations ?
1. now do the same but separately for week days and for week ends, so that both curves show on the same plot  
   we expect something like the picture in the introduction

:::{admonition} hint
:class: tip dropdown

for the second question, group by 2 criteria, and think about `unstack()` to get 2 columns  
this is the same trick as what we have seen with `pivot_table`
:::

```{code-cell} ipython3
:tags: [level_basic]

by_hour = df.groupby(by="hour")
df_hour_mean = by_hour["n_bikes"].mean().reset_index(name="mean_n_bikes")
df_hour_mean.head()

# le reset_index sert à transformer le résultat d'un groupby en un DataFrame
# avec une colonne pour l'index (ici: hour)
# une colonne pour les valeurs (ici: la moyenne de n_bikes)
# un nom explicite pour cette colonne

plt.plot(df_hour_mean["hour"], df_hour_mean["mean_n_bikes"])
plt.xlabel("Heure")
plt.ylabel("Nombre moyen de cyclistes")
plt.title("Moyenne de n_bikes par heure")
```

The busiest hours are around 9:00 am and 6:00 pm.

```{code-cell} ipython3
by_hour_iswk = df.groupby(by=["hour", "is_weekend"])
df_mean = by_hour_iswk["n_bikes"].mean().reset_index(name="mean_n_bikes")
display(df_mean.head())

df_weekend = df_mean[df_mean["is_weekend"] == True]
df_weekdays = df_mean[df_mean["is_weekend"] == False]

plt.plot(df_weekend["hour"], df_weekend["mean_n_bikes"], label="weekend")
plt.plot(df_weekdays["hour"], df_weekdays["mean_n_bikes"], label="weekdays")

plt.xlabel("Heure")
plt.ylabel("Nombre moyen de cyclistes")
plt.title("Moyenne de n_bikes par heure")
plt.legend()
plt.show()
```

## 3. using the time as the index

+++

it is very common to use the timestamps as the index of a time series; this makes slicing very convenient

1. make `timestamp` the index of `df`
1. make sure the index is sorted in chronological order (check that with `index.is_monotonic_increasing`)
1. what are the first and the last timestamps ? so, how long is the period covered ?

```{code-cell} ipython3
:tags: [level_basic]

df.set_index("timestamp", inplace=True)
print(df.index.is_monotonic_increasing)
```

```{code-cell} ipython3
print("First timestamp :", df.index.min())
print("Last timestamp :", df.index.max())
print("Lenght of the covered period :", df.index.max() - df.index.min())
```

### missing data

+++

this is a counter that sits in the street, so there can be some missing data  
we expect one row per hour

1. how many rows would you expect between the first and the last timestamp ? how many do we have ?
1. how many rows contain `NaN` in the `n_bikes` column ?
1. which days are entirely missing ?

:::{admonition} hint
:class: tip dropdown

* `pd.date_range()` can create the index with all the hours; use the option `freq="h"`
* or use `df.asfreq("h")`, that creates the missing rows with `NaN` in it
:::

```{code-cell} ipython3
:tags: [level_basic]

print("We expect ", 394*24 + 9, "rows")
print("We have ", len(df.index), "rows")
```

```{code-cell} ipython3
nbr_nan = df["n_bikes"].isna().sum()
print(nbr_nan, "rows contain NaN in the n_bikes column")
```

```{code-cell} ipython3
dates_present = df.index.date

all_days = pd.date_range(df.index.min(), df.index.max(), freq="D").date

missing_days = sorted(set(all_days) - set(dates_present))

print("Days entirely missing:", missing_days)
```

## 4. slicing

+++

now that we have a time index, we can slice with `.loc[]` using strings that look like dates  
let us practice, using `df.loc[...]` to answer these questions

1. what are the total number of bikes counted in December 2025 ?
1. how many rows do we have between January 1st and January 7th 2026 (both included) ?  
   is the end of the slice included or not ?
1. how many bikes in total in the morning rush, i.e. between 7:00 and 10:00 (all days included) ?
1. how many rows do we have on 2026-03-29 ? and on 2025-10-26 ? why is that ?

:::{admonition} hint
:class: tip dropdown

for question 3, you can build a mask from the hour of the day, which is available
as an attribute of the index (`df.index.hour`) or from the column we have created earlier  
and if you like, there is also a method `between_time()` that does it directly
:::

```{code-cell} ipython3
:tags: [level_basic]

print("The total number of bikes counted in December 2025 is :", 
      df.loc["2025-12"].n_bikes.sum())
print("Between January 1st and January 7th 2026 we have", 
      len(df.loc["2026-01-01":"2026-01-07"]),
      "rows")

print("We have", len(df.loc["2026-03-29"]), "rows on 2026-03-29")
print("We have", len(df.loc["2025-10-26"]), "rows on 2025-10-26")
```

```{code-cell} ipython3
# print("Between 7:00 and 10:00 we have",
#       df.loc["07:00": "10:00", "n_bikes"].sum(), 
#       "bikes")

morning_total = df.loc[(df["hour"] >= 7) & (df["hour"] <= 10), "n_bikes"].sum()

print("Between 7:00 and 10:00 we have", morning_total, "bikes")
```

### is the practice growing ? (optional)

+++

our data covers the period from September 12th to October 11th **twice**: in 2025 and in 2026  
compare the total number of bikes in each of these 2 periods

```{code-cell} ipython3
:tags: [level_basic]

print("We have", df.loc["2026-10-11"].n_bikes.sum(), "bikes in 2026-10-11")
print("We have", df.loc["2025-10-11"].n_bikes.sum(), "bikes in 2025-10-11")
print("Gap :", abs(df.loc["2026-10-11"].n_bikes.sum() - df.loc["2025-10-11"].n_bikes.sum()), "bikes")
```

## 5. resampling

+++

`resample()` is like a `groupby`, where the groups are time intervals; it is the tool of choice to change the granularity of a time series  
so, to move from one point per hour to one point per day

1. compute `daily`, the total number of bikes per day (a `Series`), and plot it
1. what is the busiest day ? and the least busy one ?  
   hint: `idxmax()` and `idxmin()`

```{code-cell} ipython3
:tags: [level_basic]

daily = df["n_bikes"].resample("24h").sum() # ou "D" à la place de "24h"

daily.plot()
plt.xlabel("Date")
plt.ylabel("Number of bikes")
plt.title("Number of bikes per day")
plt.show()
```

```{code-cell} ipython3
print("Busiest day :", daily.idxmax().date())
print("Least busy day :", daily.idxmin().date())
```

### beware of the missing data

+++

some days have **no bike at all**, which is suspicious - recall that some days were missing entirely  
on the plot above they show as a few sharp drops down to 0, in the late summer and in the fall

1. which are these days ? what happened ? what does `sum()` return on a group with no data at all ?
1. fix `daily` so that these days show up as `NaN` rather than 0  
   hint: look at the options of `sum()`
1. plot again, but only the last 2 months, and with `marker="."` to see the individual points;  
   what does `plot()` do with a `NaN` ?

```{code-cell} ipython3
:tags: [level_basic]

zero_days = daily[daily == 0]
print(zero_days)
```

```{code-cell} ipython3
missing_days = sorted(set(all_days) - set(daily.index.date))
print("Missing entirely:", missing_days)
```

```{code-cell} ipython3
daily.loc["2026-03-29"]
```

```{code-cell} ipython3
new_daily = df["n_bikes"].resample("24h").sum(min_count=1)
# min_count=1 : la somme est calculée ssi il y a au moins 1 valeur
# sinon, il retourne NaN
print(new_daily[new_daily.isna()])
```

```{code-cell} ipython3
last_two_months = new_daily.loc["2026-09": "2026-10"]
last_two_months.plot(marker=".")
```

```{code-cell} ipython3
print("With a NaN, plot() skips the value of the date and doesn't plot it")
```

### weekly and monthly

+++

1. compute and plot the number of bikes per week, and per month  
   in the resulting series, what is the first date for a month ? and for a week ?
1. the first and last months are not complete; in addition, the months have different lengths  
   so a monthly total is not the best way to compare them; compute the **average per day** instead  
   hint: you can use `daily`
1. make a dataframe with 3 columns `total`, `mean` and `max` giving, for each month, the corresponding aggregates of the daily numbers  
   hint: `agg()`

```{code-cell} ipython3
:tags: [level_basic]

weekly = df["n_bikes"].resample("W").sum() 
weekly.name = "weekly bikes"
monthly = df["n_bikes"].resample("ME").sum()
monthly.name = "monthly bikes"

weekly.plot()
monthly.plot()
plt.xlabel("Date")
plt.ylabel("Number of bikes")
plt.legend()
plt.show()
```

```{code-cell} ipython3
print(weekly.index[:5])
monthly.index[:5]
```

```{code-cell} ipython3
monthly_avg = new_daily.resample("ME").mean()

monthly_avg.plot(marker=".")
plt.xlabel("Date")
plt.ylabel("Average bikes per day")
plt.title("Monthly average daily bike count")
plt.show()
```

```{code-cell} ipython3
monthly_stats = new_daily.resample("ME").agg(["sum", "mean", "max"])
monthly_stats.columns = ["total", "mean", "max"]
monthly_stats.head()
```

:::{admonition} hint
:class: tip dropdown

the aliases for the frequency are `"W"` for weeks, and `"ME"` for the end of the months  
(`"MS"` for the start of the months); in older versions of pandas it was `"M"`
:::

+++

## 6. adding the weather

+++

does the weather make a difference ? let us find out with the second file

1. load `data/weather.csv` into a dataframe `weather`, and make sure the column `jour` becomes a proper timestamp  
   careful with the format: `12/09/2025` is meant to be September 12th; what would happen if pandas chose to read it as December 9th ?  
   use the `format=` option of `pd.to_datetime` to say what you mean
1. make that column the index of `weather`
1. combine `daily` and `weather` into one dataframe `both`, with the 3 columns `n_bikes`, `temperature` and `rain_mm`  
   at that point you may run into an issue with the indexes - that is because one has a time zone, and the other one has not  
   we recommend to remove the time zone from `daily`, you can use `tz_localize(None)` on its index

```{code-cell} ipython3
:tags: [level_basic]

weather = pd.read_csv("data/weather.csv")
print(weather.dtypes)
weather.head()
```

If pandas chose to read it as December 9th it would switch the day and the month for all data, and display an error when the day exceeds 12.

```{code-cell} ipython3
weather["jour"] = pd.to_datetime(weather.jour, format="%d/%m/%Y")
weather.dtypes
```

```{code-cell} ipython3
weather.set_index("jour", inplace=True)
```

```{code-cell} ipython3
weather.head()
```

```{code-cell} ipython3
daily = new_daily.copy()
daily.index = daily.index.tz_localize(None) #enlever le fuseau horaire 
daily.index = new_daily.index.date #convertir l'index de daily en date pure 
daily_df = daily.to_frame("n_bikes")

both = daily_df.merge(
    weather,
    left_index=True,
    right_index=True,
    how="left" #garde toutes les dates de weather même si pas de vélos 
)
both.head()
```

### a look at the result

+++

1. are there less bikes on rainy days ?  
   let us say a day is rainy if there is more than 1mm of rain  
   compare the average number of bikes on rainy days and on other days
1. draw a scatter plot of the number of bikes against the temperature, using `df.plot.scatter()` or similar
1. what is the correlation between the 3 columns ?  
   hint: `corr()`

```{code-cell} ipython3
:tags: [level_basic]

rainy = both[both["rain_mm"]>1]
not_rainy = both[both["rain_mm"]<=1]

print("Average number of bikes on rainy days :", rainy["n_bikes"].mean())
print("Average number of bikes on other days :", not_rainy["n_bikes"].mean())

if rainy["n_bikes"].mean() < not_rainy["n_bikes"].mean():
    print("There are less bikes on rainy days")
else: 
    print("There are more bikes on rainy days")S
```

```{code-cell} ipython3
both.plot.scatter(x="temperature", y="n_bikes")
# alternative: plt.scatter(x=both.temperature, y=both.n_bikes)
plt.xlabel("Temperature (°C)")
plt.ylabel("Number of bikes")
plt.title("Number of bikes against the temperature")
plt.show()
```

```{code-cell} ipython3
both.corr()
```

## appendix: where the data comes from

+++

* the bike counts come from the open data portal of the city of Paris, dataset *comptage vélo - données compteurs*  
  <https://opendata.paris.fr>
* the weather comes from the *Open-Meteo* archive API, for Paris  
  <https://open-meteo.com>
* the script `fetch-data.py`, that comes with this assignment, was used to acquire this data
  - it picks one counter, and exports the hours in local time **with the UTC offset**, as the original does
  - and it writes the days of the weather in the French format on purpose
* the data is frozen, no need to refresh it
