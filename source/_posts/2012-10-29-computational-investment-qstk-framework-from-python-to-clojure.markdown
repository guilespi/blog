---
layout: post
title: "Computational Investment QSTK Framework from Python to Clojure"
date: 2012-10-29 01:07
comments: true
categories: 
- clojure
- coursera
- python
---
Last week started the course on [Computational Investing][1] from Coursera and I've been taking a look.

What caught my attention is the libraries used for portfolio construction and management, [QSTK][2], an opensource python framework, based on numpy, scipy, matplotlib, pandas, etc.

Looking at the [first tutorial][4]'s [source code][3], saw it as an opportunity to migrate the tutorials and libraries to Clojure and get to play a little with [Incanter][5].

I'm going to highlight what I've found interesting when migrating the tutorials. I'm assuming you have QSTK installed and the QS environment variable is set, since the code depends on that for data reading.

``` clojure
(def ^{:dynamic true} *QS* (get (System/getenv) "QS"))
```

**NYSE operation dates**

As part of the initialization process the tutorial calls a function [getNYSEDays][6], which retrieves all the days there was trading at the NYSE. Migration is straightforward using incanter's read-dataset to read file into memory and then filter the required range.

{% gist 3970987 %}

Pay attention to the `time-of-day` set at 16 hours, [the time NYSE closes][9], we'll see it again in unexpected places.

**Data Access**

QSTK provides a helper class called [DataAccess][7] used for reading and caching stock prices.

As you see here there's some data reading happening, we're gonna take a look at these functions since we'll need to write them from scratch.

``` python Data initialization in python tutorial 
dataobj = da.DataAccess('Yahoo')
voldata = dataobj.get_data(timestamps, symbols, "volume",verbose=True)
close = dataobj.get_data(timestamps, symbols, "close",verbose=True)
actualclose = dataobj.get_data(timestamps, symbols, "actual_close",verbose=True)
```
We're going to separate this in two functions, first reading symbol data from disk using again read-dataset and creating a hash-map indexed by symbol name.

``` Clojure Creating a symbols hash-map of incanter datasets
(defn read-symbols-data
  "Returns a hashmap of symbols/incanter datasets read from QS data directory"
  [source-in symbols]
  (let [data-dir (str *QS* "/QSData/" source-in "/")]
    (reduce #(assoc %1 %2 (incanter.io/read-dataset (str data-dir %2 ".csv") :header true)) {} symbols)))
```

Then if you take a look at `voldata` in a python repl, you can see pretty much what it's doing

                           AAPL       GLD     GOOG        $SPX       XOM
     2012-05-01 16:00:00  21821400   7414800  2002300  2706893315  13816900
     2012-05-02 16:00:00  15263900   5632300  1611500  2634854740  11108700
     2012-05-03 16:00:00  13948200  13172000  1868000  2673299265   9998600

It's grabbing the specified column `volume` or `close` from each symbol dataset, and it's creating a new table with the resulting column renamed as the symbol.

All the get_data magic happens inside [get_data_hardread][8], it's an ugly piece of code making a lot of assumptions about column names, and even about market closing time. I guess you can only use this library for markets closing at 16 hours local time.

``` python
timemonth = int((timebase-timeyear*10000)/100) 
timeday = int((timebase-timeyear*10000-timemonth*100)) 
timehour = 16 
```

I've translated that into these two functions:

{% gist 3971076 %}

In this case Clojure shines, the [original function][10] is almost 300 lines of code. I'm missing a couple of checks but it's not bad for a rookie, I think.

The helper function `select-value` is there in order to avoid an exception when trying to find stock data for a non existent date. Also the function returns `:Date` as a double since it's easier to handle later for charting.

**Charting**

Charting with Incanter is straightforward, there a subtle difference with python since you need to add each series one by one. So what python is doing here charting multiple series at once

```python
newtimestamps = close.index                                                                                                                                                     
pricedat = close.values # pull the 2D ndarray out of the pandas object
plt.plot(newtimestamps,pricedat)
```

We need a little function to solve it with Incanter. Each iteration gets reduced into the next with all the series accumulated in one chart.

``` clojure creates multiple time-series at once
(defn multi-series-chart
  "Creates a xy-chart with multiple series extracted from column data
  as specified by series parameter"
  [{:keys [series title x-label y-label data]}]
  (let [chart (incanter.charts/time-series-plot :Date (first series)
                                                 :x-label x-label
                                                 :y-label y-label
                                                 :title title
                                                 :series-label (first series)
                                                 :legend true
                                                 :data data)]
  (reduce #(incanter.charts/add-lines %1 :Date %2 :series-label %2 :data data) chart (rest series))))
```

**Data Mangling**

Incanter has _a lot_ of built-in functions and helpers to operate on your data, unfortunately I couldn't use one of the many options for operating
on a matrix, or even `$=`, since the data we're processing has many `nil` values inside the dataset for dates the stock didn't trade which raises an exception when
treated as a number, which is what to-matrix does, tries to create an array of Doubles.

There's one more downside and it's we need to keep the `:Date` column as-is when operating on the dataset, so we need to remove it, operate, and add it later again, what happens to be a beautiful one-liner in python

``` python This attempts a naive normalization dividing each row by the first one. 
 normdat = pricedat/pricedat[0,:]
```

``` python Or the daily return function. 
dailyrets = (pricedat[1:,:]/pricedat[0:-1,:]) - 1
```

I ended up writing from scratch the iteration and function applying code.

{% gist 3971236 %}

Maybe there's an easier way but I couldn't think of it, if you know a better way please drop me a line!

Now normalization and daily-returns are at least manageable.

``` clojure Normalization and Daily Returns

(defn normalize
  "Divide each row in a dataset by the first row"
  [ds]
  (let [first-row (vec (incanter.core/$ 0 [:not :Date] ds))]
    (apply-rows ds (/ first-row) 0 (fn [n m] (and (not-any? nil? [n m]) (> m 0))))))


(defn daily-rets
  "Daily returns"
  [data]
  (apply-rows data
            ((fn [n m] (- (/ n m) 1)) (vec (incanter.core/$ (- i 1) [:not :Date] data)))
            1
            (fn [n m] (and (not-any? nil? [n m]) (> m 0)))))
```

Having the helper functions done, running of the tutorial is almost declarative.

{% gist 3971246 %}

If you wanna take a look at the whole thing together here's the [gist][11], I may create a repo later.

{% img center /images/blog/chartincanter.png 380 480 'Incanter charting finance data' %} 

Please remember NumPy is way much faster than Clojure since it links [BLAS/Lapack][12] libraries.

_Follow me on [twitter][13]_

[1]: https://class.coursera.org/compinvesting1-2012-001/class/index
[2]: http://wiki.quantsoftware.org/index.php?title=QuantSoftware_ToolKit
[3]: https://gist.github.com/3971007
[4]: http://wiki.quantsoftware.org/index.php?title=QSTK_Tutorial_1
[5]: http://incanter.org/
[6]: http://www.quantsoftware.org/Docs/html/QSTK.qstkutil.dateutil-pysrc.html#getNYSEdays
[7]: http://www.quantsoftware.org/Docs/html/QSTK.qstkutil.DataAccess.DataAccess-class.html
[8]: http://www.quantsoftware.org/Docs/html/QSTK.qstkutil.DataAccess-pysrc.html#DataAccess.get_data_hardread
[9]: http://en.wikipedia.org/wiki/List_of_market_opening_times
[10]: https://gist.github.com/3971102
[11]: https://gist.github.com/3971253
[12]: http://www.netlib.org/lapack/
[13]: http://www.twitter.com/guilespi
