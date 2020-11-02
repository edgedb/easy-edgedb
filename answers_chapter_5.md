# Chapter 5 Questions and Answers

#### 1. What do you think `SELECT std::to_datetime(3600);` will return, and why?

The one function signature that takes a single integer is this one:

```
std::to_datetime(epochseconds: int64) -> datetime
```

And since it gives the number of seconds after the Unix Epoch (1970), it will return this:

`{<datetime>'1970-01-01T00:01:00Z'}`

So one hour (3600 seconds) after the epoch began.

2.

3.

4.

5.
