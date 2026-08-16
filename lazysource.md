# Delay-loaded series

The `DelayedSeries` type provides an efficient way to create series whose data is loaded
on-demand. For example, you may have a large time series stored in a CSV file or in a
database and you do not want to load all the data in memory if the user only needs a
small part of it.

When you create a delayed series, you specify the overall range of the series (i.e. the
minimum and maximum key value) and you provide a function that loads a specified sub-range
of the series. When the user accesses a continuous range of the series, the loading function
is called to retrieve the data.

<a name="create"></a>
## Creating a delayed series

To create a delayed series, we need a function that generates data for a given range.
The following function generates a series with random data for a given date range with
a day frequency:

```fsharp
let generate (low:DateTime) (high:DateTime) : seq<KeyValuePair<DateTime,float>> = 
    let rnd = Random()
    let days = int (high - low).TotalDays
    seq [ for d in 0 .. days -> KeyValuePair(low.AddDays(float d), rnd.NextDouble()) ]
```

Now we use `DelayedSeries.FromValueLoader` to create a delayed series. It takes the overall
minimum and maximum key of the series and a function that loads data for a sub-range. The
loading function gets the lower and upper bound as a tuple of `(key, BoundaryBehavior)`
values where `BoundaryBehavior` is either `Inclusive` or `Exclusive`:

```fsharp
let min = DateTime(2010, 1, 1)
let max = DateTime(2013, 1, 1)

let ls = DelayedSeries.FromValueLoader(min, max, fun (lo, lob) (hi, hib) -> async {
    printfn "Query: %A - %A" lo hi
    let lo = if lob = BoundaryBehavior.Inclusive then lo else lo.AddDays(1.0)
    let hi = if hib = BoundaryBehavior.Inclusive then hi else hi.AddDays(-1.0)
    return generate lo hi })
```

The key thing about the above is that, so far, no data has been loaded. The loading function
is called only when we access part of the series.

<a name="slicing"></a>
## Slicing and using delayed series

We can now use the series as usual - for example, to get data for the entire year 2012:

```fsharp
let slice = ls.[DateTime(2012, 1, 1) .. DateTime(2012, 12, 31)]
slice
```

```
val slice: Series<DateTime,float> =
  
(Delayed series [01/01/2012 .. 12/31/2012]) 

val it: Series<DateTime,float> =
  
(Delayed series [01/01/2012 .. 12/31/2012])
```

Similarly, we can add the delayed series to a data frame. When doing this, Deedle will
only load the data that is needed. In the following example, we add the series to a frame
and then access only a slice:

```fsharp
let df = frame ["Values" => ls]
let slicedDf = df.Rows.[DateTime(2012,6,1) .. DateTime(2012,6,30)]
slicedDf
```

```
Query: 01/01/2010 00:00:00 - 01/01/2013 00:00:00
Query: 06/01/2012 00:00:00 - 06/30/2012 00:00:00
val df: Frame<DateTime,string> =
  
              Values               
01/01/2010 -> 0.49966101794707674  
01/02/2010 -> 0.8321020021267803   
01/03/2010 -> 0.6535841555245155   
01/04/2010 -> 0.015357202274300374 
01/05/2010 -> 0.6034899904029314   
01/06/2010 -> 0.3912842793733601   
01/07/2010 -> 0.9504585488795269   
01/08/2010 -> 0.7656073562375441   
01/09/2010 -> 0.34673597266794687  
01/10/2010 -> 0.17019885358391518  
01/11/2010 -> 0.6248724567519237   
01/12/2010 -> 0.8054078605696605   
01/13/2010 -> 0.5728527800518847   
01/14/2010 -> 0.04549166898521462  
01/15/2010 -> 0.26181309995719704  
:             ...                  
12/18/2012 -> 0.013237251036229636 
12/19/2012 -> 0.06863704317049102  
12/20/2012 -> 0.7088080155754566   
12/21/2012 -> 0.1101945855355202   
12/22/2012 -> 0.20384461777099316  
12/23/2012 -> 0.49368206711990925  
12/24/2012 -> 0.21402243529861087  
12/25/2012 -> 0.6237491911811096   
12/26/2012 -> 0.45802983234255323  
12/27/2012 -> 0.36821279031707377  
12/28/2012 -> 0.22979297242839158  
12/29/2012 -> 0.35876446941155915  
12/30/2012 -> 0.8638231953914562   
12/31/2012 -> 0.3061904646330834   
01/01/2013 -> 0.8847319866603327   

val slicedDf: Frame<DateTime,string> =
  
              Values              
06/01/2012 -> 0.6964380025309842  
06/02/2012 -> 0.1305095018657937  
06/03/2012 -> 0.1509865113299118  
06/04/2012 -> 0.8090873423559201  
06/05/2012 -> 0.08924384015652542 
06/06/2012 -> 0.625884799332181   
06/07/2012 -> 0.06229965182585295 
06/08/2012 -> 0.7017745133375601  
06/09/2012 -> 0.3321850396012367  
06/10/2012 -> 0.23434288443232232 
06/11/2012 -> 0.5376842841758026  
06/12/2012 -> 0.7339187354661388  
06/13/2012 -> 0.19353676272863096 
06/14/2012 -> 0.9489953568530483  
06/15/2012 -> 0.5609718187934891  
06/16/2012 -> 0.7108399105701002  
06/17/2012 -> 0.7252833661450184  
06/18/2012 -> 0.7530610660653271  
06/19/2012 -> 0.8976642146553938  
06/20/2012 -> 0.691178491323665   
06/21/2012 -> 0.640463034101986   
06/22/2012 -> 0.7445872815494404  
06/23/2012 -> 0.3738664841529751  
06/24/2012 -> 0.09901765455242106 
06/25/2012 -> 0.64046094823065    
06/26/2012 -> 0.5256575421222498  
06/27/2012 -> 0.49463512969037016 
06/28/2012 -> 0.6325260779849133  
06/29/2012 -> 0.14345838775252073 
06/30/2012 -> 0.7718488966710241  

val it: Frame<DateTime,string> =
  
              Values              
06/01/2012 -> 0.6964380025309842  
06/02/2012 -> 0.1305095018657937  
06/03/2012 -> 0.1509865113299118  
06/04/2012 -> 0.8090873423559201  
06/05/2012 -> 0.08924384015652542 
06/06/2012 -> 0.625884799332181   
06/07/2012 -> 0.06229965182585295 
06/08/2012 -> 0.7017745133375601  
06/09/2012 -> 0.3321850396012367  
06/10/2012 -> 0.23434288443232232 
06/11/2012 -> 0.5376842841758026  
06/12/2012 -> 0.7339187354661388  
06/13/2012 -> 0.19353676272863096 
06/14/2012 -> 0.9489953568530483  
06/15/2012 -> 0.5609718187934891  
06/16/2012 -> 0.7108399105701002  
06/17/2012 -> 0.7252833661450184  
06/18/2012 -> 0.7530610660653271  
06/19/2012 -> 0.8976642146553938  
06/20/2012 -> 0.691178491323665   
06/21/2012 -> 0.640463034101986   
06/22/2012 -> 0.7445872815494404  
06/23/2012 -> 0.3738664841529751  
06/24/2012 -> 0.09901765455242106 
06/25/2012 -> 0.64046094823065    
06/26/2012 -> 0.5256575421222498  
06/27/2012 -> 0.49463512969037016 
06/28/2012 -> 0.6325260779849133  
06/29/2012 -> 0.14345838775252073 
06/30/2012 -> 0.7718488966710241
```
