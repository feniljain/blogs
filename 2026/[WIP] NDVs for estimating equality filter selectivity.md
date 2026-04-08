Now that I work on databases, I have a habit of keeping up with upstream datafusion PRs. Today I noticed an interesting PR talking about usage of NDVs in equality filter selectivity. I have always been fascinated by NDVs cause my colleagues in planner team always mention them as something super helpful. I started looking into the PR and it turned out to be a small one, but there was a review on it and honestly I did not understand it at all. So I sat down to do some reading on how this works.

Firstly, what are NDVs? NDVs are number of distinct values, these are usually stored at parquet file level. We can also compute NDVs for a column, i.e. how many distinct values a single column contains.

What is filter selectivity? For a filter supplied in a query, number of rows selected by it is called it's selectivity. For e.g. a filter which filters out 50 rows out of total 100 has a selectivity of 50%.

Now lets understand how do they come together, lets say we have a join query like this:
```
select * from A where A.x = B.y;
```

In this, we have our join condition as `A.x = B.y`, if we can predict what will be the selectivity of this filter expression we can make interesting decisions based on it. A good example is: do we want to use partition-wise join or non-partition-wise join? ( i.e. if we have loads of rows to join on, we can distribute them across cores instead of doing it on single core itself )

NDVs help us do exactly that, let's say we have a condition where `y = 42`. And let's say `y` column has 5 distinct values, that means our NDV count is 5. As we don't have exact histograms telling us about data distribution, we assume each value is "uniformly distributed" across whole column. For e.g. if `y` column is made up of `{38,39,40,41,42}` and has 100 values in total, we assume there are 20 values of 38, 20 values of 39 and so on. This assumption means probability of 42 getting matched is equal to all others distinct values i.e. 1 / 5. If we multiply this with total number of rows in the column, we get selectivity of `y = 42` as 20. Here key point is understanding us assuming uniform distribution, if we had histograms, we could exactly tell how many rows have value 42 in the column, but NDVs work as next best case.

This estimation of rows helps in join order estimation, join type estimation, etc.

After understanding this I noticed there was a review comment on the PR and tried to decode that. Review was as follows

```
I think this new `1 / distinct_count` branch is a little too broad as written. Right now it fires whenever the pruned interval collapses to a single value, but that is not quite the same thing as proving we have an equality filter.

For example, if the incoming stats already describe a singleton interval, or if a conjunction of inequalities narrows the range to one point without actually adding any selectivity beyond the existing stats, we would still scale by `1 / NDV` here and end up under-estimating the row count.
```

This was a total bouncer for me, it was so high that if this were a cricket match, umpire would call it a WIDE. But let's try to break it down, so author's current condition to use `1/NDV` is as follows:
```rust
if ...
	target.distinct_count
                    && distinct_count > 0
                    && !target_interval.lower().is_null()
                    && target_interval.lower() == target_interval.upper() {...}
```

In this interval means zone maps i.e. min/max values of that column. In our case above `y` would have min/max values as `{39,42}`.

Condition checks if NDV count is not zero and `target_interval`'s lower value is same as upper value, if everything passes we assume our filter selectivity as `1/NDV`. 

According to reviewer, `1/NDV` estimation is incorrect in following cases:
- "if the incoming stats already describe a singleton interval"
- "if a conjunction of inequalities narrows the range to one point without actually adding any selectivity beyond the existing stats"

Both of these reviews at the core address the problems of:

```
shape of data changes as it gets processed by different operators
```

```
singleton does not guarantee an equality filter source
```

Lets try to understand above line with an example. Lets say we have two filters on our `y` column due to some CTE/subquery etc:

- first being: `y >= 41 || y <= 42`
- and second being: `y > 33 AND y < 42` (non equality condition)

After first filter we would have:

- bounds: `[41, 42]`
- NDV count: 5 (notice it didn't change)

When we come to second filter and apply bounds to predicate we only get rows containing 41. Here we will predict selectivity as `1/NDV`. This is the exact problem, lets say out of first filter we get 70 rows out i.e. first filter has selectivity of 70%. Now lets say we have 35 rows of `41` and 35 rows of `42`, after applying second filter 35 rows are remaining i.e. 50% selectivity. But, if we go by NDV route, we get `70/5` i.e. 14 rows, that is a super low estimation!

Our NDV count did not change as data flowed through both filters, same phenomenon can happen with different operators in the middle. We also saw that even though we got singleton interval as an 

This was an interesting dive, which confused me a lot at different places, even while writing this down!

References:
- https://learn.microsoft.com/en-us/sql/relational-databases/performance/cardinality-estimation-sql-server?view=sql-server-ver17
- https://github.com/apache/datafusion/pull/20789/
- https://blobs.duckdb.org/papers/tom-ebergen-msc-thesis-join-order-optimization-with-almost-no-statistics.pdf