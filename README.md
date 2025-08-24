# US_BabyNames_1880_2010 Analysis

### Introduction
the data between our hands in this project is the evolution of female and male baby names in the united states from 1880 to 2010
this data Analysis projects aims to analyse some aspects of the evolution of the us baby names in the period going from 1880 to 2010
a lot can be done using this data,we can for example:

-Visualize the proportion of babies given a particular name over time

-Determine the relative rank of a name

-Determine the most popular names in each year or the names whose popularity has advanced or declined the most

-Analyze trends in names: vowels, consonants, length, overall diversity, changes in
spelling, first and last letters

-Analyze external sources of trends: biblical names, celebrities, demographics

### Table of contents
-[Data Sources](#data-sources)

-[Data Wrangling and Preparation](#data-wrangling-and-preparation)

-[Analysing Naming Trends](#analysing-naming-trends)

### Data Sources 

The united states social security Administration(ssa) [download here]([https://github.com/wesm/pydata-book](https://github.com/wesm/pydata-book/tree/3rd-edition/datasets/babynames))

###Tools

Python,pandas , matplotlib, numpy

### Data Wrangling and Preparation

.1 The data is already in coma separated value so we can just load the data using pandasead_csv to load the files as a DataFrame, but since the data is split in     multiple files, we need to assemble the whole
  data in one DataFrame this will be done using the pandas.concat, also we create a columns for the years
  to do so i created a list i called pieces, then i create a loop for year in range(1880-2011), this loop reads as a csv each file
  each file of the data and puts it as a dataframe with the columns indicated in the code, then we create a column for the years  and we append each frame in the
  pieces list after this we concate all the pieces while ignoring the index because we'are not interested in the original index of each file
  ```python
  pieces=[]
for year in range(1880,2010):
    path= f"babynames/yob{year}.txt"
    frame=pd.read_csv(path,names=["names","sex","births"])
    frame["year"] = year
    pieces.append(frame)
names = pd.concat(pieces, ignore_index=True)
```

.2  i then add a column to indicate the percentage of the each name relative to the total population names but i don't want a global percentage that dosent take       account of the year and the sex so i parse in a function to calculate the percentage in regards of the year and the sex,fter this a sanity check is done to        ensure that the sum of props is equal to one,    
  ```python
def add_prop(group):
  group["prop"] = group["births"] / group["births"].sum()
  return group
names=names.groupby(["year","sex"]).apply(add_prop)
names = names.reset_index(drop=True)
#the verification of the sum of the rates is done using the next line :
names.groupby(["year","sex"])["prop"].sum()
```

.3 the third step would be the extraction of a subset equivalent to the top 1000 names in the sex/year combination 
```python
  top1000 = (
    names.sort_values(by=["year", "sex", "births"], ascending=[True, True, False])
    .groupby(["year", "sex"])
    .head(1000)
)
```
### Analysing Naming Trends

.1 i start of by creating two subdatasets one for the boys and one for the girls, then using the pivot methode :
```python
total_births = top1000.pivot_table("births", index="year",
.....: columns="name",
.....: aggfunc=sum)
```
the resulting table is the number of births for each name in each year, this table gives us the possibility to plot the evolution of any name we want as long as it exists in the table, in this case i did a plotting for the names :(john, Harry, Mary, Marilyn)


2. Analysing the increase in the naming diversity across the years.
  one of the most intriguing things about naming is the change that occured in the number of the unique names used as time goes on
  One measure is the proportion of births represented by the top 1,000 most popular names
  ```python
  table = top1000.pivot_table("prop", index="year",columns="sex", aggfunc=sum)
  table.plot(title="Sum of table1000.prop by year and sex", yticks=np.linspace(0, 1.2, 13))
  ```
  Another interesting metric is the number
  of distinct names, taken in order of popularity from highest to lowest, in the top 50%
  of births. This number is trickier to compute. Let’s consider just the boys names from 2010
  i take a substrata of the boys names of the year 2009, and then using the searchsorted(0.5)
  then i apply the same process on the year 1900 in order to compare both year
```python
df = boys[boys.year == 1900]
in1900 = df.sort_values("prop", ascending=False).prop.cumsum()
in1900.searchsorted(0.5) + 1
```
  the results show that in order to reach 50% of the names used in 1900 we only needed 25 names, on the other hand to reach the same percentage in the 2009 we need 113 name
  You can now apply this operation to each year/sex combination, groupby those fields,
  and apply a function returning the count for each group:
 ```python
  def get_quantile_count(group, q=0.5):
    group = group.sort_values("prop", ascending=False)
    return group.prop.cumsum().searchsorted(q) + 1
  diversity = top1000.groupby(["year", "sex"]).apply(get_quantile_count)
  diversity = diversity.unstack()
  diversity.plot(title="Number of popular names in top 50%")
```
the resulting plot would be :
figure 1 : diversity plot
<img width="792" height="522" alt="Capture" src="https://github.com/user-attachments/assets/c9e55e76-67eb-4130-b6ea-7c1394c03179" />
as we can see the the diversity increased overtime, but we can also see that the girls names have always been more diverse than those of the boys
and even the rate of evolution is far bigger then it's counterpart

3. Analysing the evolution of the last letter

we start of by a function that takes the last letter of each name :
```python
def get_last_letter(x):
    return x[-1]
last_letters = names["names"].map(get_last_letter)
last_letters.name="last_letter"
```

then we pivot the table to get the number of last letters by sex and by years, and we reduce the table's year to only get three representative years of our revolution
```python
table = names.pivot_table("births", index=last_letters,columns=["sex", "year"], aggfunc=sum)
subtable = table.reindex(columns=[1910, 1960, 2009], level="year")
```
after getting the number of each last letter in each of 1910,1960 and 2009, i then normalize by the total births per year and sex to compute a new table containing the
proportion of total births for each sex ending in each letter
```python
subtable.sum()
letter_prop = subtable / subtable.sum()
```
  figure 2 : proportion of boy and girl names ending in each letter
  <img width="792" height="658" alt="Capture2" src="https://github.com/user-attachments/assets/a2ec426c-900a-404d-9a89-465865bca995" />

  then again i normalize by year and sex by dividing the each value in the table by the sum of course the values are grouped by year and sex which i stressed before,after that i select a subset of letters for     the boy names,("d", "n", "y"), finally transposing to make each column a time series:
  ```python
  letter_prop = table / table.sum()
  dny_ts = letter_prop.loc[["d", "n", "y"], "M"].T
  dny_ts.plot()
  ```
  figure 3 : evolution of the proportion of names ending with "d","n","y :
  
  <img width="696" height="540" alt="c3" src="https://github.com/user-attachments/assets/1deb03d3-3bc6-4fc6-8b5a-74c3331afa04" />


  4. Boy names that became girl names
  one of the names that were used for boys and became after a century mostly used for girls is the name lesly and it's variations(leslie,lesley...)
  so i start of by creating a time serie that contains all the names, without a repetition using the unique(), then  i substract the ones containing "Lesl"
  
  ```python
  all_names = pd.Series(top1000["names"].unique())
  lesley_like = all_names[all_names.str.contains("Lesl")]
  ```
  after that i used the generated lesley likes names to  filter down to just those names and sum births grouped by name to see the relative frequencies, also i pivot by the 
  years and the sex because thos are two are what matters to us, as we are looking into the change that occured in the use of those names for boys and girls,then i did divide by the sum of the two columns 
  in each year to get the percentage of use for each sex, so i can finally get a plot that shows the evolution of the use of the lesley-like name for boys and girls.
  ```python
  filtered = top1000[top1000["names"].isin(lesley_like)]
  filtered.groupby("names")["births"].sum()
  table = filtered.pivot_table("births", index="year",columns="sex", aggfunc="sum")
  table = table.div(table.sum(axis="columns"), axis="index")
  table.plot(style={"M": "k-", "F": "k--"})
  ```
  figure 4 : Proportion of male/female Lesley-like names over time
  
  <img width="784" height="489" alt="C4" src="https://github.com/user-attachments/assets/9617b9af-7173-492d-8b91-601934e5d3fc" />

  



















