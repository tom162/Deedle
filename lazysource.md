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
01/01/2010 -> 0.2151114167944127   
01/02/2010 -> 0.18123723728518548  
01/03/2010 -> 0.10793134852611697  
01/04/2010 -> 0.5420104966460336   
01/05/2010 -> 0.564772136874553    
01/06/2010 -> 0.6464726035827457   
01/07/2010 -> 0.5700769929552012   
01/08/2010 -> 0.3135997061130926   
01/09/2010 -> 0.46322791529732166  
01/10/2010 -> 0.2532419390139836   
01/11/2010 -> 0.43741012824865155  
01/12/2010 -> 0.5010332149907948   
01/13/2010 -> 0.11483700657382312  
01/14/2010 -> 0.6217039002824685   
01/15/2010 -> 0.5951239652233622   
:             ...                  
12/18/2012 -> 0.46220383218472694  
12/19/2012 -> 0.9598088092839276   
12/20/2012 -> 0.12992325311079878  
12/21/2012 -> 0.11851936952554865  
12/22/2012 -> 0.778067391710587    
12/23/2012 -> 0.24872653527687982  
12/24/2012 -> 0.18730508381485023  
12/25/2012 -> 0.22314500897668965  
12/26/2012 -> 0.5313507310774014   
12/27/2012 -> 0.8677403804543451   
12/28/2012 -> 0.3093608945626608   
12/29/2012 -> 0.9340265185099206   
12/30/2012 -> 0.12096337702111837  
12/31/2012 -> 0.8510034256284627   
01/01/2013 -> 0.042801728058227795 

val slicedDf: Frame<DateTime,string> =
  
              Values              
06/01/2012 -> 0.24518815168879682 
06/02/2012 -> 0.8330280253931518  
06/03/2012 -> 0.537731198757038   
06/04/2012 -> 0.5764249182177448  
06/05/2012 -> 0.18381552104572607 
06/06/2012 -> 0.7782037287872507  
06/07/2012 -> 0.9552243695632909  
06/08/2012 -> 0.682901493878961   
06/09/2012 -> 0.45358668492196863 
06/10/2012 -> 0.46980560516615455 
06/11/2012 -> 0.9497744742014721  
06/12/2012 -> 0.6086606668206125  
06/13/2012 -> 0.45310121203469267 
06/14/2012 -> 0.2455513168078044  
06/15/2012 -> 0.27431835492768974 
06/16/2012 -> 0.818737187318256   
06/17/2012 -> 0.20687090306105238 
06/18/2012 -> 0.24646177923886303 
06/19/2012 -> 0.770699779957472   
06/20/2012 -> 0.08600056379968202 
06/21/2012 -> 0.8667419160272641  
06/22/2012 -> 0.20808284470937188 
06/23/2012 -> 0.7237070702944666  
06/24/2012 -> 0.2744990346717038  
06/25/2012 -> 0.5606290805027039  
06/26/2012 -> 0.8951371173094179  
06/27/2012 -> 0.24689962183050118 
06/28/2012 -> 0.787790683108691   
06/29/2012 -> 0.7677188409253726  
06/30/2012 -> 0.3702191848433285  

val it: Frame<DateTime,string> =
  
              Values              
06/01/2012 -> 0.24518815168879682 
06/02/2012 -> 0.8330280253931518  
06/03/2012 -> 0.537731198757038   
06/04/2012 -> 0.5764249182177448  
06/05/2012 -> 0.18381552104572607 
06/06/2012 -> 0.7782037287872507  
06/07/2012 -> 0.9552243695632909  
06/08/2012 -> 0.682901493878961   
06/09/2012 -> 0.45358668492196863 
06/10/2012 -> 0.46980560516615455 
06/11/2012 -> 0.9497744742014721  
06/12/2012 -> 0.6086606668206125  
06/13/2012 -> 0.45310121203469267 
06/14/2012 -> 0.2455513168078044  
06/15/2012 -> 0.27431835492768974 
06/16/2012 -> 0.818737187318256   
06/17/2012 -> 0.20687090306105238 
06/18/2012 -> 0.24646177923886303 
06/19/2012 -> 0.770699779957472   
06/20/2012 -> 0.08600056379968202 
06/21/2012 -> 0.8667419160272641  
06/22/2012 -> 0.20808284470937188 
06/23/2012 -> 0.7237070702944666  
06/24/2012 -> 0.2744990346717038  
06/25/2012 -> 0.5606290805027039  
06/26/2012 -> 0.8951371173094179  
06/27/2012 -> 0.24689962183050118 
06/28/2012 -> 0.787790683108691   
06/29/2012 -> 0.7677188409253726  
06/30/2012 -> 0.3702191848433285
```
