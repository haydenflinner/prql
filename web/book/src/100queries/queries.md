# PRQL In 100 Queries

Translated from [this popular SQL article](https://gvwilson.github.io/sql-tutorial/), PRQL is introduced below
showing its strengths early and often.

# Setup
One can follow the sqlite setup given in the article. Use a PRQL to SQL translator in [the playground](https://prql-lang.org/playground/) or add a prql sqlite plugin.

### Select constant.
This one is a little weird. The only thing I could get to compile without a compiler error is:
```
from numbers
select { mynum = 1 }
```

### Select all values from table;
```
from little_penguins
```

This shows our first difference from SQL. Rather than requiring the SELECT first (often considered a [mistake](https://stackoverflow.com/a/5075797)), PRQL permits it to be basically anywhere you'd like to filter down the set of columns. Here, since we didn't write it, it's implicitly all of them.

### 3. Specify Columns
```diff
from little_penguins
+select {species, island, sex}
```

### 4. Sort

Vs SQL, differences:
Anywhere you would see a comma-separated list, we wrap with {} to disambiguate (or technically to group the arguments we're passing to the select function). Rather than ASC/DESC, we use +/-. And again, trailing select.
```diff
from little_penguins
+sort {+age, -sex}
select {species, island, sex}
```

### 5. Limit Output
Prql's version of `LIMIT` is `take`.
[Doc here.](https://prql-lang.org/book/reference/stdlib/transforms/take.html)

```diff
from penguins
sort {+age, -sex}
select {species, island, sex}
+take 10
```

### 6. Page Output (Pagination)

```diff
from penguins
sort {+age, -sex}
select {species, island, sex}
-take 10
+take 3..13
```

### 7. Remove Duplicates
The SQL query here looks simple but the original article points out there may be clouds on the horizon:
>SQL was supposed to read like English

[Prql has no distinct.](https://prql-lang.org/book/how-do-i/distinct.html?highlight=distinct#how-do-i-remove-duplicates)

TODO How to know when use `()` vs `{}`?

```
from penguins
select { species, sex, island }
group penguins.* (
  take 1
)
```

### 8/9. Filter Results (with more complex conditions) (WHERE)
PRQL's WHERE is [`filter`](https://prql-lang.org/book/reference/stdlib/transforms/filter.html?highlight=filter#filter).
It also uses programming style comparators (`==` rather than `=`, which is generally for assignment) and boolean operators (`&&` rather than `AND`).

```diff
from penguins
select { species, sex, island }
group penguins.* (
  take 1
)
+filter (island == 'Biscoe' && sex != 'MALE')
```


### 10/11. Do Calculations + Rename Columns
Here I choose to demonstrate labeling the results of your calculations with a unit in the name (In SQL, `AS`), because leaving them unlabeled is asking for trouble.

You don't have to though, as what would be `body_mass_kg` below shows.

```
from penguins
select {
  flipper_length_cm = flipper_length_mm / 10.0,
  body_mass_g / 1000.0
}
take 3
```

### 12-16. Nulls
See [null handling](https://prql-lang.org/book/reference/spec/null.html) section of PRQL book, but generally one must consider the behavior of the underlying SQL/database NULL.

We use `col == null` in PRQL rather than SQL's special `col IS NULL`. Same for `!=` and `IS NOT`.


### 17. Aggregate
```
from penguins
aggregate {
    total_mass = sum body_mass_g,
}
```
If you punch this in on the playground you will notice that it actually compiles to
`COALESCE(SUM(body_mass_g), 0) AS total_mass`. This is to avoid a random `NULL` appearing as the answer unexpectedly.


### 18. Common aggregation functions
```
from penguins
aggregate {
    longest_bill = max bill_length_mm,
    shortest_flipper = min flipper_length_mm,
    bill_length_depth_ratio = (average bill_length_mm) / (average bill_depth_mm)
}
```

### 19. Counting
It's interesting that `count sex` actually compiles to `count *`, but if you think about it,
[it makes sense](https://stackoverflow.com/questions/2710621/count-vs-count1-vs-countpk-which-is-better).

`count_distinct` seems to be deprecated in favor of something not yet implemeentd.

```
from penguins
aggregate {
    count_star = count *,
    count_specific = count sex,
    count_distinct = count distinct sex
}
```

### 20. Group
```
from penguins
group sex (
    aggregate {
        average_mass_g = average body_mass_g,
    }
)
```

This is an interesting one because it points out a mistake/oversight/boilerplate in SQL,
that the original query doesn't show the sex for which the average is being taken.

This was corrected in 21, so we'll skip it.

### 22. Arbitrary Choice in Aggregation
22 is impossible to write in PRQL, which is good because it's invalid in most DBs.
We'll write it as what you probably wanted, which is just to take a 'random' row, you
don't care which because you believe all will be the same for your purposes.

We're saying to take the "first" row, but not specifying what order we want things in,
so the database will return the one it feels is easiest to return first.
```
from penguins
group sex (
    take 1
)
select {sex, body_mass_g}
```

Note how much prettier this `take 1` is in PRQL vs the equivalent SQL; that friction
visible in the SQL is probably why sqlite chose to support technically invalid SQL statements such as this one.
Just as a matter of convenience.

### 23. Filter Aggregated Values
Here we see SQL's `HAVING`, which is symptomatic of SQL's overall design. PRQL's pipelined approach means that `filter` can just happen after the group, no different syntax needed.
```diff
from penguins
group sex (
    aggregate {
        average_mass_g = average body_mass_g,
    }
)
+filter average_mass_g > 4000.0
```

### 24. Readable Output (Rounding)
```diff
from penguins
group sex (
    aggregate {
        # Not sure why `round` seems to have less precedent than e.g. average,
        # but prefixing with `math.` gets it back. Also not sure why bad type error for first.
        # average_mass_g = math.round (1 (average body_mass_g)),
+        average_mass_g = average body_mass_g | math.round 1,
    }
)
filter average_mass_g > 4000.0
```

### 25. Filter Aggregate Inputs
Here we see yet another nonstandard SQL extension. Some of these extensions are supported
by many DBs but under different syntaxes. This particular extension I've never seen before,
despite spending quite some time with sqlite. It does look a lot like PRQL, though!

```diff
from penguins
+filter body_mass_g < 4000.0
group sex (
    aggregate {
        # Not sure why `round` seems to have less precedent than e.g. average,
        # but prefixing with `math.` gets it back. Also not sure why bad type error for first.
        # average_mass_g = math.round (1 (average body_mass_g)),
        average_mass_g = average body_mass_g | math.round 1,
    }
)
-filter average_mass_g > 4000.0
```

### 26 - 30 DDL
DDL is Data Definition Language, stuff like creating tables, inserting, updating values.
DDL is outside of PRQL's scope, and usually database-specific.
For example, check out postgres's INSERT RETURNING and the equivalents in every other database. It's a whole nother world of things for PQL to target.


### 31 Join
TODO I can't figure out how to do a cross-join in PRQL.


### 32 Inner Join.
```
from work
join job (work.job == job.name)
```

### 33. Aggregate Joined Data
This is another one where the pipelined nature of PRQL shines. You can see the SQL one totally changes for, with only the middle of it staying the same.
Also note that we don't have to explicitly re-list the things we're grouping on in select, preventing bugs/mistakes/boilerplate.
```diff
from work
join job (work.job == job.name)
+group work.person (
+   aggregate {
+     pay = sum job.billable
+   }
+)
```

TODO This gives a horrible misleading error when the correction is to use parens (and what decides parents vs {}'s ?
```
from work
join job (work.job == job.name)
group work.person {
    aggregate {
      pay = sum job.billable
    }
}
```

### 34. Left join
```
from work
join side:left job (work.job == job.name)
```

### 35. Aggregate Left Joins
Note PRQL already gives 0 rather than NULL, no hack needed as the SQL guide note asks for.
```
from work
join side:left job (work.job == job.name)
group work.person {
    aggregate {
        pay = sum job.billable
    }
}
```

### 36. Coalesce Values
This (making sure you get 0 rather than NULL) is already done automaticallyby PRQL, you don't have to worry about this.

## Constructing a higher-level query
In the below, we'll follow a few steps to arrive at our most complicated query yet.

### 37. Negate Incorrectly
This is an interesting demonstration. The SQL version is wrong, but someone reading it as English
may assume it means what the author implied it to mean; something it doesn't do.

The PRQL version is more obvious. What is the "work" table? A list of jobs a person has. We'll filter
to job entries that are not 'calibrate', and then we'll take only distinct names who had jobs
that weren't calibrate.
So, our query tells us "persons who have a job that is not calibration", which is what it does.
There is no implication from an incorrect/quick read that we're taking people whose job is not to calibrate.

```
from work
filter (job != 'calibrate')
group person (
  take 1
)
```
oI
### 38. Set Membership
TODO it is here that i discovered Book > Reference > Stdlib. This is good stuff.
It didn't tell me about !( though, and searching "IN" in the reference finds nothing since it's only two letters.
This is SQL NOT IN.
```
from work
filter !(person | in ['mik', 'tay'])
```


### 39. Subqeuries
Here's really the crux of what makes PRQL PRQL.
In SQL, we quickly reach a point where the provided English-like syntax is not sufficient.
Computing folks know that an additional level of indirection is all that's needed to solve virtually every problem. Thus, subquery syntax, added on to model the abstraction not present in the original model.
I actually went to poke at PRQL to see if it had an equivalent to subqueries.
What I found was not at all; just a reference to a [problem with SQL subqueries](https://dba.stackexchange.com/questions/260446/does-the-order-by-clause-of-a-subquery-pass-through-to-the-main-results-postg) that I had never heard of
but which probably has lead to undefined results in large, critical queries that I have written in the past.
You may have noticed that as a common theme in the early answers, the insistence that PRQL was protecting you from SQL sharp edges that you may not have even known existed.
Enough adieu, let's see PRQL's answer here.

```
from work
select person
remove (from work | select {person, job} | filter job == 'calibrate')
# filter !(person | in from work | filter job = 'calibrate')
```

TODO badd err:
```
# 3 │ filter !(work.person | in (from work | filter (job = 'calibrate')))
#   │                                                      ─────┬─────  
#   │                                                           ╰─────── function std.filter, param `condition` expected type `bool`, but found type `text`
```
TODO lots of errors in the making of that one about unsupported wildcard. I knew that meant to shuffle some selects before because I've wondered about how the compiler works, but a lot of folks aren't going to figure that out. Some quick suggestions in this case may go a long way, see Rust compiler.

### 40-43. More DDL
autoinc / primary key (SKIP), alter tables, create new tables from old, remove tables

### 44. Compare individual values to aggregates
Here, PRQL really shines.

"Two-pass" data scan is documented in the comments of the SQL but apparent from the PRQL here. Although in general a db may simplify to something faster than we wrote, see the compiled SQL for the previous question... Here at least, the probable need to scan twice is apparent and not hidden behind a WHERE. No performance surprises.

Futher, the order of operations is reversed in this query vs SQL. To me, this makes more sense, you start reading in the state that the data starts, and it gets transformed as you go into what you want. You wouldn't want to read directions that start at where you're going rather than where you are.

Finally, this is just objectively better. We've given the name body_mass_g only twice as opposed to 3 times, improving signal-to-noise-ratio (SNR). Penguins once instead of twice, doubling SNR. Finally, we named the value "avg_body_mass_g" for use in the WHERE clause, improving legibility for cases where the nested query may not be obvious at a glance (and didn't use comments to explain). PRQL guides you without you noticing into doing the right thing.

```
from penguins
 # This doesn't do what you might think it does based on SQL.
 # derive { avg_body_mass_g = average body_mass_g }
  aggregate {
    avg_body_mass_g = average body_mass_g,
  }
join penguins (true)
filter body_mass_g > avg_body_mass_g
```

TODO another weird bug
```
from penguins
 select {body_mass_g}
 derive { avg_body_mass_g = average body_mass_g }
  aggregate {
    avg_body_mass_g = average body_mass_g,
  }
join penguins (true)
filter body_mass_g > avg_body_mass_g
```
This gives the exact same thing as removing derive. I wonder if this is a comment pattern, queries for which a non-trivial line being removed yields the same result. Although this one semantically I don't think makes sense.
```diff
from penguins
 select {body_mass_g}
 # This doesn't do what you might think it does based on SQL.
- derive { avg_body_mass_g = average body_mass_g }
  aggregate {
    avg_body_mass_g = average body_mass_g,
  }
join penguins (true)
filter body_mass_g > avg_body_mass_g
```



### 45. Compare individual values to aggregates within groups
Notice how the comments on this one are given in the same order as the PRQL answer, while they're out of order with SQL's required declaration order.
> * Subquery runs first to create temporary table averaged with average mass per species
>
> * Join that with penguins
>
> * Filter to find penguins heavier than average within their species

Use self-equality
[operator](https://prql-lang.org/book/reference/stdlib/transforms/join.html?highlight=join#self-equality-operator)
to improve SNR.

Also note how similar it is to the above query, with the addition/change of only 2 or 3 lines, for 3/7 >50% increase.. Vs the SQL version growing by a static 6 lines, or 8/14 (>50%) increase. Interestingly, SNR may be a constant factor better in PRQL? We'll see as we get to more advanced queries.

```
from penguins
group species (
    aggregate {
      avg_body_mass_g = average body_mass_g,
    }
)
join side:inner penguins (==species)
filter penguins.body_mass_g > avg_body_mass_g
```

TODO I am not sure this query would make any sense if entered into a db.
```
from penguins
# This doesn't do what you might think it does based on SQL.
# derive { avg_body_mass_g = average body_mass_g }
  aggregate {
    avg_body_mass_g = average body_mass_g,
  }
join side:inner penguins (species)
#filter body_mass_g > avg_body_mass_g
```

### 46. Common Table Expressions

These are a necessary tool in SQL to name your values and rearrange your queries to a more natural order.
in PRQL we've already been naming our values and doing things in a natural order.
I will instead link for you functions and loops, which CTEs are commonly used to implement/approximate.
https://prql-lang.org/book/reference/declarations/functions.html?highlight=function#functions
https://prql-lang.org/book/reference/stdlib/transforms/loop.html?highlight=loop#loop

### 47. Rowid
This is Sqlite-specific advice about the implicit per-row column "rowid".

### 48. If-else
PRQL has case matching instead.
https://prql-lang.org/book/reference/syntax/case.html?highlight=case#case

11 lines instead of 17, and we handle nulls properly whereas in the SQL, null body_mass_g's gets called 'large'.

TODO man, it never stops annoying me that I use ()'s on group and then immediately {} on aggregate. And i never remember or see at my font size / tiredness level.

```
from sized_penguins
derive size = case [
  body_mass_g < 3500 => 'small',
  body_mass_g >= 3500 => 'large',
]
group {species, size} (
  aggregate {
    num = count species
  }
)
sort {species, size}
```

### 49. CASE
We just covered this, it's no different than 48 for us. PRQL has no special if-else.

### 50. Check Range

```
from sized_penguins
derive size = case [
  (body_mass_g | in 50..100) => 'small',
  body_mass_g >= 3500 => 'large',
]
group {species, size} (
  aggregate {
    num = count species
  }
)
sort {species, size}
```

TODO Without the parentheses I get this. Would be nice to autosuggest parens if they'd help.
```
 3 │   body_mass_g | in 50..100 => 'small',
   │               ┬  
   │               ╰── unexpected | while parsing function call
```

### 50. Check range
TODO This pattern (without the parens) fails to parse with what could be a better error message.
```
 3 │   body_mass_g | in 50..100 => 'small',
   │               ┬  
   │               ╰── unexpected | while parsing function call
```

```
from sized_penguins
derive size = case [
  body_mass_g | in 50..100 => 'small',
  body_mass_g >= 3500 => 'large',
]
group {species, size} (
  aggregate {
    num = count species
  }
)
sort {species, size}
```

```
from sized_penguins
derive size = case [
  (body_mass_g | in 50..100) => 'small',
  body_mass_g >= 3500 => 'large',
]
group {species, size} (
  aggregate {
    num = count species
  }
)
sort {species, size}
```

### 51. Pattern matching
```
from staff
# Extra credit: Use S-strings to use sqlite's glob feature instead
filter (name ~= "ya" || family ~= "De")
select {personal, family}
```

### 52. Select first and last rows

TODO Can't get this one to not ICE
```
from experiment
append (from experiment | sort started)
#append (from experiment | sort {-started} | take 5 | select started)
```

### 53. Intersection

```
let mbs = (from staff | select {personal, family, dept, age} | filter dept == 'mb')
let young = (from staff | select {personal, family, dept, age} | filter age < 50)

from mbs
intersect young
```

### 54. Exclusion

```
let mbs = (from staff | select {personal, family, dept, age} | filter dept == 'mb')
let young = (from staff | select {personal, family, dept, age} | filter age < 50)

from mbs
remove young
```
