# Chapter 5 Questions and Answers

#### 1. What do you think `SELECT std::to_datetime(3600);` will return, and why?

The one function signature that takes a single integer is this one:

```
std::to_datetime(epochseconds: int64) -> datetime
```

And since it gives the number of seconds after the Unix Epoch (1970), it will return this:

`{<datetime>'1970-01-01T00:01:00Z'}`

So one hour (3600 seconds) after the epoch began.

#### 2. How many seconds went by between 5:00 am on Christmas Day 2003 in Turkmenistan (TMT) and 7:00 pm on New Year's Eve for the same year in Uzbekistan (UZT)?

This one is easy with the `to_datetime()` function:

```
SELECT to_datetime(2003, 12, 31, 19, 0, 0, 'UZT') - to_datetime(2003, 12, 25, 5, 0, 0, 'TMT');
```

The answer is 568,000 seconds: `{568800s}`

3.

4.

5.
