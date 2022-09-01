![](media/image14.png){width="3.744792213473316in"
height="0.5532994313210848in"}\
Evaluating Modin as a Pandas replacement

![short line](media/image17.png){width="0.4895833333333333in"
height="6.25e-2in"}

August 2022

# 

[Introduction](#introduction) **2**

[Comparison tools (methods/data)](#comparison-tools-methodsdata) **5**

[Comparison data used:](#comparison-data-used) **6**

[Goal:](#goal) **7**

[Test (comparison criteria):](#test-comparison-criteria) **7**

[Possible(anticipated) blockers/challenges (unknowns) and proposed
solutions:](#possibleanticipated-blockerschallenges-unknowns-and-proposed-solutions)
**7**

[Script (approach)](#script-approach) **7**

[Results](#section-5) **10**

[Results enumerated:](#results-enumerated) **11**

[Fun exercise (future comparisons):](#fun-exercise-future-comparisons)
**16**

[Questions?](#questions) **16**

#  

## Introduction

One of the current undertakings here at xVG group is evaluating Modin
for use in large datasets - to learn what advantages and disadvantages
it provides when compared to pandas.

Modin is a potentially full replacement for Pandas because it can use
more than one CPU at any given time (Pandas is single-threaded). Modin
allows the use of all of the cores on your machine or cluster while
Pandas leaves idle cores.

The question is: how much improvement in efficiency (execution time) can
Modin provide?

This guide will walk you through our approaches to answer this question,
what our script does and what the main results are.

First, let's define what Modin really is. As Ray's website explains:

> "[[Modin]{.underline}](https://github.com/modin-project/modin),
> previously Pandas on Ray, is a dataframe manipulation library that
> allows users to speed up their pandas workloads by acting as a drop-in
> replacement. Modin also provides support for other APIs (e.g.
> spreadsheet) and libraries, like xgboost."
>
> "The modin.pandas DataFrame is an extremely light-weight parallel
> DataFrame. Modin transparently distributes the data and computation so
> that all you need to do is continue using the pandas API as you were
> before installing Modin. Unlike other parallel DataFrame systems,
> Modin is an extremely light-weight, robust DataFrame. Because it is so
> light-weight, Modin provides speed-ups of up to 4x on a laptop with 4
> physical cores.
>
> In pandas, you are only able to use one core at a time when you are
> doing computation of any kind. With Modin, you are able to use all of
> the CPU cores on your machine. Even in read_csv, we see large gains by
> efficiently distributing the work across your entire machine."
>
> Advantages of Modin over Pandas, per Modin documentation - direct
> quotes:

1.  Performance

> Modin's implementation enables you to use all of the cores on your
> machine, or all of the cores in an entire cluster. The additional
> utilization leads to improved performance.

2.  Memory usage and immutability

> The pandas API contains many cases of "inplace" updates, which are
> known to be controversial. This is due in part to the way pandas
> manages memory: the user may think they are saving memory, but pandas
> is usually copying the data whether an operation was inplace or not.
>
> Modin allows for inplace semantics, but the underlying data structures
> within Modin's implementation are immutable, unlike pandas. This
> immutability gives Modin the ability to internally chain operators and
> better manage memory layouts, because they will not be changed. This
> leads to improvements over pandas in memory usage in many common
> cases, due to the ability to share common memory blocks among all
> dataframes.
>
> Modin provides the inplace semantics by having a mutable pointer to
> the immutable internal Modin dataframe. This pointer can change, but
> the underlying data cannot, so when an inplace update is triggered,
> Modin will treat it as if it were not inplace and just update the
> pointer to the resulting Modin dataframe.

3.  API vs implementation

> It is well known that the pandas API contains many duplicate ways of
> performing the same operation. Modin instead enforces that any one
> behavior have one and only one implementation internally. This
> guarantee enables Modin to focus on and optimize a smaller code
> footprint while still guaranteeing that it covers the entire pandas
> API. Modin has an internal algebra, which is roughly 15 operators,
> narrowed down from the original \>200 that exist in pandas. The
> algebra is grounded in both practical and theoretical work. Learn more
> in our [VLDB 2020 paper](https://arxiv.org/abs/2001.00888). More
> information about this algebra can be found in the
> [architecture](https://modin.readthedocs.io/en/stable/development/architecture.html)
> documentation.

##  

## Comparison tools (methods/data)

We have selected 32 common methods to compare Modin and Pandas. These
choices are based on what people generally use the pandas API for:

1.  as measured by the Rise lab study on top 20 used Pandas methods in
    > kaggle (Reference: PyData 2019 NY presentation video here:
    > [[https://www.youtube.com/watch?v=-HjLd_3ahCw]{.underline}](https://www.youtube.com/watch?v=-HjLd_3ahCw))

    -   We have selected the top 10 from this list

2.  As can be inferred from our own experience

3.  All Pandas commands that are currently in Fletcher

Comparison functions (first 10 are in order of Rise Lab ranking, yellow
highlighted are all Pandas commands that are currently in Fletcher ):

+-------------+-------------+-------------+-------------+-------------+
| 1.          | 2.  .rea    | 3.  .pd.    | 4.          | 5.  .mean() |
| .read_csv() | d_parquet() | DataFrame() |   .append() |             |
+=============+=============+=============+=============+=============+
| 6.  .head() | 7.  .drop() | 8.  .sum()  | 9.          | 10. .t      |
|             |             |             |   .to_csv() | o_parquet() |
+-------------+-------------+-------------+-------------+-------------+
| 11. .get()  | 12. .mode() | 13.         | 14. .std()  | 1           |
|             |             | .describe() |             | 5. .shape() |
+-------------+-------------+-------------+-------------+-------------+
| 16          | 17          | 18          | 19. .gro    | 20          |
| . .unique() | . .cumsum() | . .concat() | upby.mean() | . transpose |
+-------------+-------------+-------------+-------------+-------------+
| 21. .val    | 22. .so     | 23. .to     | 24          | 2           |
| ue_counts() | rt_values() | _datetime() | . .astype() | 5. .merge() |
+-------------+-------------+-------------+-------------+-------------+
| 26. .join() | 2           | 28. hash    | 29. d       | 30. .       |
|             | 7. .apply() |             | ate_range() | timestamp() |
+-------------+-------------+-------------+-------------+-------------+
| 31. isna()  | 32          |             |             |             |
|             | . .series() |             |             |             |
+-------------+-------------+-------------+-------------+-------------+

## Comparison data used:

1.  Different file sizes

    a.  Small: CoronaHack Respiratory Sound Dataset

> (https://www.kaggle.com/datasets/praveengovi/coronahack-respiratory-sound-dataset)
>
> This data has 37 columns with features such as USER_ID, Gender,
> demographic details, dates, ailments, etc. For integer manipulations
> we used the Age column.
>
> ![](media/image11.png){width="4.814057305336833in"
> height="0.6735258092738408in"}

b.  Large: NYC taxi data

> This data has 20 columns with features such as VendorID, trip
> distance, tip and tall amount, etc. For integer manipulations we used
> the trip distance column.
>
> ![](media/image8.png){width="3.763888888888889in"
> height="0.7249704724409449in"}

2.  CSV vs parquet (loading/saving)

###  

## Goal:

-   To create a python script with functions that provide table and
    > plots of comparison results

## Test (comparison criteria):

-   Function execution time

## Possible(anticipated) blockers/challenges (unknowns) and proposed solutions:

1.  Some commands like df.head() do not 'work' on Modin, or their
    > implementations need detailed investigations

-   Solutions/approaches:

    -   Use regular pandas as ray defaults to it where methods are not
        > available

    -   If the functions are important and no good evaluation can be
        > made, leave black

2.  Finding data that can be used for all 32 functions might be a
    > challenge

    -   Solutions/approaches:

        -   If no publicly available data can be found, create one
            > artificially

3.  Some functions that might be hugely different in performance in the
    > two modules might be completely missed by our list of 32

    -   Solutions/approaches:

        -   We will consider them as we become aware of them in the
            > future

## Script (approach)

This script compares Modin and Pandas execution times using 32 Pandas
methods. Not all are implemented in Modi.

(read here:
[[https://modin.readthedocs.io/en/stable/supported_apis/defaulting_to_pandas.html]{.underline}](https://modin.readthedocs.io/en/stable/supported_apis/defaulting_to_pandas.html))

First two points on how to use this script:

1\. The script is written as a class with a driver which sets up the
analysis and plotting. The user is required to change the path,
file-saving, etc. parameters in the driver section (i.e. if \_\_name\_\_
== \"\_\_main\_\_\":).

2\. At this point, Modin doesn\'t work in cluster mode properly.
Therefore run everything in local mode.

The script is written as a class with a driver which sets up the
analysis and plotting. The user is required to change the path,
file-saving, etc. parameters in the driver section (i.e. if \_\_name\_\_
== \"\_\_main\_\_\":).

These lines are enclosed in between two double lines. The following
screenshot shows part of the driver code.

#### ![](media/image6.png){width="5.682292213473316in" height="1.5287379702537183in"}

The script's main function (the Modin_vs_Pandas class) is to create a
dataframe that has 4 columns
(\[\'Pandas\',\'Modin\',\'improvement_factor\',\'Time_diffs\'\]) and
fill it up method by method. Below are snapshots of the main
definitions:

> ![](media/image10.png){width="5.630208880139983in"
> height="3.441213910761155in"}

![](media/image19.png){width="5.619792213473316in"
height="1.3365988626421696in"}

The other main functions (screenshots below) perform dataframe filling
and plotting results. The first two show examples of filling Pandas and
Modin columns with execution times it took to read the csv file. The
last lines on each panel make sure data is imported and looks intact
(not timed).

![](media/image1.png){width="6.40625in"
height="1.2068285214348207in"}![](media/image4.png){width="6.46875in"
height="1.200906605424322in"}![](media/image12.png){width="3.8645833333333335in"
height="0.6760739282589676in"}

![](media/image9.png){width="7.0in"
height="0.2916666666666667in"}![](media/image20.png){width="7.0in"
height="0.3888888888888889in"}![](media/image7.png){width="7.0in"
height="0.3611111111111111in"}![](media/image27.png){width="6.666666666666667in"
height="0.37159776902887137in"}

Here is the plotting function (cropped) showing what is sent to
performance plotters. It sends the data and title of the generated
figure.

![](media/image21.png){width="5.635416666666667in"
height="1.7191371391076116in"}

##  

## Results

First the blockers. We found that the following are not implemented and
give different errors such as :

UserWarning: sort_values defaulting to pandas implementation.:

1.  sort_values

2.  Value_counts

3.  Merge

4.  Hash ---\-\-\-\-\-\-\-- AttributeError: module \'modin.pandas\' has
    > no attribute \'util\'

5.  to_csv

6.  to_parquet

Therefore, some comparisons will not be accurate, so please consider
these two points (caveats):

1.  Modin defaults to Pandas if there is an equivalent function. This
    > causes long execution time since Modin uses Ray and gathering
    > results takes time.

2.  We have taken precautions for plots not to be misleading by showing
    > unaccompanied bar plots when Pandas has the method, and very long
    > execution times for Modin (because of the above reason)

    -   some methods like pandas.util.hash cause error in Modin, so they
        > are encapsulated in \'try\' statements so the script can still
        > be useful when they are implemented in the future.
        > Corresponding lines in pandas are commented out. Examples
        > include: gGet', 'shape', 'merge', 'hash'

    -   some methods like read_csv in Modin revert to Pandas but still
        > show improvement over Pandas, thus are left in. Please follow
        > warnings during execution time

For more info on method implementations read:

[[https://modin.readthedocs.io/en/stable/supported_apis/dataframe_supported.html]{.underline}](https://modin.readthedocs.io/en/stable/supported_apis/dataframe_supported.html)

The results are summarized in three bar plots (i.e. 4 using popularity,
4 using Fletcher methods as sorting parameter):

1.  Sorted by function popularity as measured by:

    -   the Rise lab study on top 20 used Pandas methods in kaggle
        > (Reference: PyData 2019 NY presentation video here:
        > [[https://www.youtube.com/watch?v=-HjLd_3ahCw]{.underline}](https://www.youtube.com/watch?v=-HjLd_3ahCw)).
        > We have selected the top 10 from this list

    -   As can be inferred from our own experience

2.  Sorted by improvement factor in execution time (i.e. by how much
    > Modin improved performance, vs Pandas). Here is how it was
    > calculated:

> improvement_factor =
> round(self.performance_df.Pandas.astype(np.double)/ \\
>
> self.performance_df.Modin.astype(np.double),2)

3.  By plotting the execution time differences as semilog bars for
    > visualization

4.  By putting (plotting bars on left corner) first the 7 Pandas
    > commands that are currently in Fletcher

## Results enumerated:

1.  Modin greatly improves method execution speed (**Figs. 1 & 2**)

2.  The magnitude of improvement (such as when calculated with a factor
    > change) depends on data size. I.e. larger datasets see more
    > relative improvements in execution time.

3.  Improvements depend on methods. For example, transposing is much
    > faster in Modin (vs Pandas). (**Figs. 3 & 4**)

4.  Some methods (such as merge which are not yet implemented) are much
    > slower in Modin as they default to Pandas. This is because merging
    > in a distributed time causes longer wait times.(**Figs. 5 - 8**)

![](media/image25.png){width="7.0in" height="2.8055555555555554in"}

## ![](media/image13.png){width="7.0in" height="2.8055555555555554in"}

## Figure 1. Pandas vs Modin Fletcher function execution time, sorted by popularity 

## ![](media/image23.png){width="7.0in" height="2.8055555555555554in"}

**Figure 2.** Pandas vs Modin Fletcher function execution time. The
first 7 are in Fletcher, the rest are sorted by popularity

![](media/image24.png){width="7.0in" height="2.8055555555555554in"}

**Figure 3.** Pandas vs Modin function execution time improvement
factor, sorted by improvement gained from Modin

![](media/image15.png){width="7.0in" height="2.8055555555555554in"}

**Figure 4.** Pandas vs Modin function execution time improvement
factor. The first 7 are in Fletcher, the rest are sorted by popularity

![](media/image18.png){width="7.0in" height="2.8055555555555554in"}

**Figure 5.** Pandas vs Modin Function execution time differences (in
seconds) i.e. pandas - modin. Positive numbers indicate Modin is faster
(sorted by method popularity)

![](media/image22.png){width="7.0in" height="2.8055555555555554in"}

**Figure 6.** Pandas vs Modin Fletcher function execution time
differences (in seconds). Positive numbers indicate Modin is faster
(sorted by popularity but the First 7 are in Fletcher)

![](media/image26.png){width="7.0in"
height="2.8055555555555554in"}**Figure 7.** Pandas vs Modin semilog
scale function execution time differences (in seconds) i.e. pandas -
modin. Positive numbers indicate Modin is faster, sorted by method
popularity

![](media/image25.png){width="7.0in"
height="2.8055555555555554in"}**Figure 8.** Pandas vs Modin semilog
scale Fletcher function execution time differences (in seconds).
Positive numbers indicate Modin is faster (sorted by popularity but the
first 7 are in Fletcher)

##  

## Fun exercise (future comparisons):

-   The data tested was not sufficient to show most methods perform
    > better in Modin, TBD

-   Modin can use Ray or dask - compare them explicitly, including
    > vanilla pandas

-   Modin uses Ray Actors for the machine learning support it currently
    > provides - making it an ideal place to start the next approaches.

## Questions? 

If you have any questions, please feel free to contact us at
support@dnanexus.com
