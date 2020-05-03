## Design

This is a walkthrough of the domain model of the unison-datetime library.  It covers the following pieces: `TimeInterval`, `TimeStamp`, `Calendar`, and the `std` Gregorian+UTC calendar.

> So far there's just this design, no actual code!

### `TimeInterval`

We can measure time intervals. 

```ucm
.datetime> view TimeInterval

  unique type TimeInterval t = TimeInterval t

```
The `t` type parameter is the numeric type we'll use for measuring time.  

> `t` denotes a totally ordered field.  But we'll usually substitute an integral type that can give only truncated division.  We'll be making use of a `Group` handler for `t`.

> `TimeInterval t` denotes a one-dimensional vector space over `t`.  So it offers all the operations that `t` does, except you can't multiply two `TimeInterval t` values.

Typically we work with `TimeInterval Int`.  

The units (i.e. the time-value of the unit of the field `t`) are ticks of 100 SI nanoseconds, i.e. there are ten million ticks per second.

### `TimeStamp`

We can store timestamps, identifying a moment in time.  

This is a linear clock measuring the time interval since an 'epoch' instant (TODO - which instant?  And specify the timezone.)  (It's not 'time since boot' or 'cputime since app start'.  It also keeps incrementing regardless of leap seconds, unlike UNIX/POSIX time.)

```ucm
.datetime> view TimeStamp

  unique type TimeStamp t = TimeStamp t

```
> If `t` is Int then this can store timestamps covering 28,000 years either side of the epoch.

Typically `TimeStamp t` values are treated as opaque, since we don't really want our program to be sensitive to the arbitrary choice of the epoch.  Rather than inspect them directly, we take differences between them (yielding `TimeInterval t` values), or convert them to `DateTime` (TODO) values.  

> `TimeStamp t` denotes a set with a total order inherited from `t`.  `TimeInterval t` has a group action on `TimeStamp t` - i.e. you can add an interval to a timestamp.  

```ucm
.datetime> find TimeStamp.plus

  1. TimeStamp.plus : TimeInterval t
                      -> TimeStamp t
                      ->{algebra.group.Group t} TimeStamp t
  

```
```ucm
.datetime> find TimeStamp.diff

  1. TimeStamp.diff : TimeStamp t
                      -> TimeStamp t
                      ->{algebra.group.Group t} TimeInterval t
  

```
`TimeStamp.diff` is the inverse of `TimeStamp.plus`, in that `plus (diff s t) t == s`

### Calendar

We can convert timestamps to and from their representation as a date and time in some calendar.  Strictly speaking, the system for this representation we call the 'calendar system', and a specific instantiation of it (including some choice of time zone offset and data about leap seconds) we call a calendar.  However we typically blur this distinction and use 'calendar' for both.  

The basic domain model in this library is intended to cover a wide variety of calendar systems in use across the world and through history.  But we'll explain it with reference to the most familiar example.  

#### Gregorian Calendar (the `std` calendar)

Here is the calendar system we typically use.  In this library, a calendar handles dates _and times_ together, so this calendar is actually the sum of the following.

- The Gregorian calendar is the one used in most of the world, and specifies 12 months of various lengths, a system of leap years with a 400 year cycle, and the era (CE/AD) for numbering years.  
- UTC (Coordinated Universal Time) specifies how days/hours/minutes are built up from SI seconds, with positive (and theoretically negative) leap seconds inserted based on observations of the length of solar days.  
- A fixed choice of time-zone offset from UTC.  In this model, changing time-zone (including daylight-saving transitions) means changing calendar.  

Here's how we represent a date and time in this calendar system. 

##### DateTime

```haskell
-- found in namespace datetime.std: 'std' indicates the Gregorian+UTC system.
type DateTime t = {
  year : Int,
  month : Month,
  day : Nat,
  hour : Nat,
  minute : Nat,
  second : Nat,
  tick : t
}

type Month = January | February | March | April | May | June | July
           | August | September | October | November | December

-- example: 2007-04-05T14:30:12.5
> DateTime 2007 April 5 14 30 12 0 5000000

```

Not all `DateTime t` values are valid dates in the calendar.  The validity constraints are:
- `day` must be one of 1..31 for January, 1..30 for April etc; 1-28 for February, or 1-29 if `year` is a leap year
- `hour` must be 0..23 (i.e. 24 hour representation)
- `minute` must be 0..59
- `second` must be 0..59, or 0-60 in minutes with a positive leap second, or 0-58 in minutes with a negative leap second
- `tick` must be 0..9,999,999
- `year` can be any integer except zero (since the convention is that 1 CE/AD was preceded by 1 BCE/BC), except:
- overall, the date/time referred to must refer to an instant that fits in a `TimeStamp t` (which depends on the type t.)

##### DateTimeInterval

Just as a `TimeInverval` represents the difference between two `TimeStamp` values, so we have a `DateTimeInterval` to represent the difference between two `DateTime` values.

```haskell
type DateTimeInterval t = {
  years : Int,
  months : Int,
  days : Int,
  hours : Int,
  minutes : Int,
  seconds : Int,
  ticks : t
}

-- example: "a year and a day a later"
> DateTimeInterval 1 0 1 0 0 0 0

```

#### Calendar ability

With this calendar system in mind as an example, let's go back and see the abstract `Calendar` ability.  This describes the basic facilities that any (supported) calendar system provides.  

```haskell
-- t is the same time measurement type as in TimeStamp and TimeInterval
-- d t is the datetime type, e.g. std.DateTime
-- i t is the interval type, e.g. std.DateTimeInterval
-- (i t must be a commutative group, d t and i t must be totally ordered, plus is a group action ... TODO)
ability Calendar d i t where
  toDateTime : TimeStamp t -> d t
  fromDateTime : d t ->{Range t} Optional (Timestamp t)
  plus : i t -> d t -> d t
  diff : d t -> d t -> i t
  normalize : d t -> d t
  valid : d t -> Boolean
  toText : (t -> Text) -> d t -> Text
  fromText : (Text -> t) -> Text ->{Error Text} d t

```

The pieces here are
- `toDateTime` and `fromDateTime` which give a one-to-one mapping between valid `TimeStamp` values and `valid` datetimes
- `plus` and `diff` which allow for adding/subtracting datetimes/intervals, for example '20 January, plus one month'
- `normalize`, which for example (in the case of the `std` calendar) takes the _invalid_ result of calculating '31 May plus one month', namely 31 June, and truncates the day component to yield 30 June, a valid date
- `valid`, which must return `true` exactly on the image of `toDateTime`
- `toText` and `fromText` which translate between datetimes and some human-readable representation - in the `std` calendar, this is the ISO 8601 representation, for example "2007-04-05T14:30:12.0Z".

> We typically work with `Calendar std.DateTime std.DateTimeInterval Int`.

Some observations:
- It is required that the calendar system does provide a one-to-one mapping between `TimeStamp` values and valid datetimes.  
  - An example of a calendar that doesn't guarantee this is the Julian calendar (predecessor to the Gregorian), which as the intercalary day added in leap years added a _second_ 24 February, which would make `toDateTime` non-injective.  We can still work with this calendar, but would need to augment the datetime representation type with an indicator field to say which 24 February we are talking about!  
  - Also note that the calendar must not have any 'holes': moments in time for which there is no valid representation - in particular its extent must reach forward to the latest representable `TimeStamp` and backward to the earliest representable `TimeStamp`.  That last condition is saying that all calendars in this library must be 'proleptic' (extended into the past before the calendar was defined), although it's not saying that anyone has to _use_ datetime values that region.
- There is no single natural way of normalizing dates after adding intervals: should '31 May plus one month' be 30 June or 1 July?  APIs specific to a calendar system can provide different options here - `normalize` just provides access to some, possible arbitrary, choice between those options.  In the case of the `std` calendar it is truncation (30 June).
- There are many ways of representing dates/times as text, even within the scope of the Gregorian calendar, and these vary by language, personal preference and local culture.  For example, month names, 12 hour vs 24 hour clock, and DD/MM/YYYY vs MM/DD/YYYY etc.  This is currently out of scope of this library, but it's easy to imagine an API which reads from system culture/locale/preference settings to provide alternatives to `toText`.

> There is a `'{Calendar d i t, Group t, Group i t, Range t} [Test]` provided for testing the `Calendar` adheres to all the appropriate laws (e.g. `plus` is a group action, `normalize` returns a `valid` result, etc.)

### Other sketch notes

##### Building up the std calendar handler

```haskell
-- start of 1-Jan-1970 ?
timeStampEpoch : t -> StdDateTime t
-- leap seconds known as at x/y/2019
leapSeconds2019 : [StdDateTime ()]
-- ms between timeStampEpoch and the Gregorian cycle anchor
gregorianCycleAnchor : Nat
-- the TimeInterval t is added to input timestamps as an offset for timezone
gregorianCycle : TimeInterval t -> Request (Calendar StdDateTime StdDateTimeInterval t) a -> a
-- this is with 0 CE removed and replaced with 1 BCE -- implemented by proxying to gregorianCycle
gregorianCalendar : TimeInterval t -> Request (Calendar StdDateTime StdDateTimeInterval s) a -> a
-- the arg is a list of dates of UTC leap seconds, in ascending order 
  - implemented by proxying to gregorianCalendar
std : [StdDateTime ()] -> TimeInterval s -> 
  Request (Calendar StdDateTime StdDateTimeInterval s) a -> a

```

##### Other bits and pieces

```haskell
ability SystemTime where -- consult system clock
  now : TimeStamp
-- might be fun correcting for leap second shenanigans possibly already added to the clock
-- POSIX time is not just linear - it includes leap second 'corrections', plus 
-- extra bodging so that every day still includes exactly 86400 seconds.
-- Or some implementations scale time to follow NTP.

ability TimeZoneOffset where
  -- current timezone offset (to add to UTC), based on system configuration
  -- and possibly time of year.
  timeZoneOffsetMinutes : Int

ability GetCalendar where
  -- get a Calendar handler based on currently configured TimeZoneOffset and leap-second data
  -- TODO think about DST-aware calendars, so this doesn't change over DST jumps
  handler : Request (Calendar std.DateTime std.DateTimeInterval Int) a -> a

-- UTC as it was known in 2019.  Expected to be accurate to within x seconds until 20xx.
UTC2019 : Request (Calendar std.DateTime std.DateTimeInterval Int) a -> a

```

