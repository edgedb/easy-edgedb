# Chapter 5 Questions and Answers

#### 1. What do you think `SELECT std::to_datetime(3600);` will return, and why?

The one function signature that takes a single integer is this one:

```
std::to_datetime(epochseconds: int64) -> datetime
```

And since it gives the number of seconds after the Unix Epoch (1970), it will return this:

`{<datetime>'1970-01-01T01:00:00Z'}`

So one hour (3600 seconds) after the epoch began.

#### 2. Will `SELECT <int16>9 + 1.06n IS decimal;` work? And if it does, will it return `{true}`?

Yes, and yes. EdgeDB will choose `decimal` as the more precise of the two types. You can also see the type just with `SELECT <int16>9 + 1.06n;` because the return shows the n: `{10.06n}`

#### 3. How many seconds went by between 5:00 am on Christmas Day 2003 in Turkmenistan (TMT) and 7:00 pm on New Year's Eve for the same year in Uzbekistan (UZT)?

This one is easy with the `to_datetime()` function:

```edgeql
SELECT to_datetime(2003, 12, 31, 19, 0, 0, 'UZT') - to_datetime(2003, 12, 25, 5, 0, 0, 'TMT');
```

The answer is 158 hours: `{<duration>'158:00:00'}`

So we can either multiply this by 3600 second per hour as a separate query:

```edgeql
SELECT 158 * 3600;
```

Which gets us `{568800}`;

Or we can use another built-in function called `duration_to_seconds`:

```edgeql
SELECT duration_to_seconds(to_datetime(2003, 12, 31, 19, 0, 0, 'UZT') - to_datetime(2003, 12, 25, 5, 0, 0, 'TMT'));
```

Which gets us `{568800.000000n}`. It's still 568,000 seconds but using more precision.

#### 4. How would you write the same query using `WITH` for each of the two times?

It would look something like this (depending on the name you give the variable and the order you prefer):

```edgeql
WITH
  uzbek_time := (SELECT to_datetime(2003, 12, 31, 19, 0, 0, 'UZT')),
  turkmen_time := (SELECT to_datetime(2003, 12, 25, 5, 0, 0, 'TMT')),
SELECT duration_to_seconds(uzbek_time - turkmen_time);
```

The output is exactly the same: `{568800.000000n}`

#### 5. What's the best way to describe a type if you only want to see how you wrote it?

The best way is `DESCRIBE TYPE AS SDL`, which doesn't have all the extra info that `AS TEXT` gives you. Here's `MinorVampire` for example:

```
{
  'type default::MinorVampire extending default::Person {
    required link master -> default::Vampire;
};',
}
```

It doesn't show any of the information from the `Person` type that it extends.
