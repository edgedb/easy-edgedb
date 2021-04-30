---
tags: Indexing, String Functions
---

# Chapter 16 - Is Renfield telling the truth?

> Arthur Holmwood's father has died and now Arthur is the head of the house. His new title is Lord Godalming, and he has a lot of money. With this money he helps the team to find the houses where Dracula has hidden his boxes.

> Meanwhile, Van Helsing is curious and asks John Seward if he can meet Renfield. He is surprised to see that Renfield is very educated and well-spoken. Renfield talks about Van Helsing's research, politics, history, and so on - he doesn't seem crazy at all! But later, Renfield doesn't want to talk and just calls him an idiot. Very confusing. And one night, Renfield was very serious and asks them to let him leave. He says: “Don’t you know that I am sane and earnest...a sane man fighting for his soul? Oh, hear me! hear me! Let me go! let me go! let me go!” They want to believe him, but can't trust him. Finally Renfield stops and calmly says: “Remember, later on, that I did what I could to convince you tonight.”

## `index on` for quicker lookups

We're getting closer to the end of the book and there is a lot of data that we haven't entered yet. There is also a lot of data from the book that might be useful but we're not ready to organize yet. Fortunately, the original book Dracula is all organized into letters, diaries, etc. that begin with the date and sometimes the time. They all start out in this sort of way:

```
Dr. Seward’s Diary.
1 October, 4 a. m.—Just as we were about to leave the house...

Letter, Van Helsing to Mrs. Harker.
“24 September.
“Dear Madam...

Mina Murray’s Journal.
8 August. — Lucy was very restless all night, and I, too, could not sleep...
```

This is very convenient for us. With this we can make a type that holds a date and a string from the book for us to search through later. Let's call it `BookExcerpt` (excerpt = part of a book).

```sdl
type BookExcerpt {
  required property date -> cal::local_datetime;
  required property excerpt -> str;
  index on (.date);
  required link author -> Person
}
```

The [`index on (.date)`](https://www.edgedb.com/docs/datamodel/indexes#indexes) part is new, and means to create an index to make future queries faster. Lookups are faster with `index on` because now the database doesn't need to scan the whole set of objects in sequence to find objects that match. Indexing makes a lookup by an exact match faster compared to always scanning everything.

We could do this for certain other types too - it might be good for types like `Place` and `Person`.

Note: `index` is good in limited quantities, but you don't want to index everything. Here is why:

- It makes the queries faster, but increases the database size.
- This may make `insert`s and `update`s slower if you have too many.

This is probably not surprising, because you can see that `index` is a choice that the user needs to make. If using `index` was the best idea in every case, then EdgeDB would just do it automatically.

Finally, here are two times when you don't need to create an `index`:

- on links,
- on exclusive constraints for a property.

Indexes are automatically created in these two cases so you don't need to use indexes for them.

So let's insert two book excerpts. The strings in these entries are very long (pages long, sometimes) so we will only show the beginning and the end here:

```edgeql
INSERT BookExcerpt {
  date := cal::to_local_datetime(1887, 10, 1, 4, 0, 0),
  author := (SELECT Person FILTER .name = 'John Seward' LIMIT 1),
  excerpt := 'Dr. Seward\'s Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once..."You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night."',
};
```

```edgeql
INSERT BookExcerpt {
  date := cal::to_local_datetime(1887, 10, 1, 5, 0, 0),
  author := (SELECT Person FILTER .name = 'Jonathan Harker' LIMIT 1),
  excerpt := '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
};
```

Then later on we could do this sort of query to get all the entries in order and displayed as JSON.

```edgeql
SELECT <json>(
  SELECT BookExcerpt {
    date,
    author: {
      name
    },
    excerpt
  } ORDER BY .date
);
```

Here's the JSON output with just a small part of the excerpts:

```
{
  "{\"date\": \"1887-10-01T04:00:00\", \"author\": {\"name\": \"John Seward\"}, \"excerpt\": \"Dr. Seward\'s Diary.\\n 1 October, 4 a.m... -- Just as we were about to leave the house...\\\"\"}",
  "{\"date\": \"1887-10-01T05:00:00\", \"author\": {\"name\": \"Jonathan Harker\"}, \"excerpt\": \"1 October, 5 a.m. -- I went with the party to the search...\"}",
}
```

After this, we can add a link to our `Event` type to join it to our new `BookExcerpt` type. `Event` now looks like this:

```sdl
type Event {
  required property description -> str;
  required property start_time -> cal::local_datetime;
  required property end_time -> cal::local_datetime;
  required multi link place -> Place;
  required multi link people -> Person;
  multi link excerpt -> BookExcerpt; # Only this is new
  property exact_location -> tuple<float64, float64>;
  property east_west -> bool;
  property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ '_N_' ++ <str>.exact_location.1 ++ '_' ++ 'E' if .east = true else 'W';
}
```

You can see that `description` is a short string that we write, while `excerpt` links to the longer pieces of text that come directly from the book.

## More functions for strings

The [functions for strings](https://www.edgedb.com/docs/edgeql/funcops/string) can be particularly useful when doing queries on our `BookExcerpt` type (or `BookExcerpt` via `Event`). One is called [`str_lower()`](https://www.edgedb.com/docs/edgeql/funcops/string#function::std::str_lower) and makes strings lowercase:

```edgeql-repl
edgedb> SELECT str_lower('RENFIELD WAS HERE');
{'renfield was here'}
```

Here it is in a longer query:

```edgeql
select BookExcerpt {
  excerpt,
  length := (<str>(SELECT len(.excerpt)) ++ ' characters'),
  the_date := (SELECT (<str>.date)[0:10]),
} FILTER contains(str_lower(.excerpt), 'mina');
```

It uses `len()` which is then cast to a string, and `str_lower()` to compare against `.excerpt()` by making it lowercase first. It also slices the `cal::local_datetime` into a string so it can just print indexes 0 to 10. Here is the output:

```
{
  Object {
    excerpt: '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
    length: '182 characters',
    the_date: '1887-10-01',
  },
}
```

Another way to make `the_date` is with the [to_str](https://www.edgedb.com/docs/edgeql/funcops/string#function::std::to_str) method, which (as you can probably guess) will turn it into a string:

```edgeql
select BookExcerpt {
  excerpt,
  length := (<str>(SELECT len(.excerpt)) ++ ' characters'),
  the_date := (SELECT to_str(.date)), #only this part is different
} FILTER contains(str_lower(.excerpt), 'mina');
```

Some other functions for strings are:

- `find()` This gives the index of the first match it finds, and returns `-1` if it can't find anything:

`SELECT find(BookExcerpt.excerpt, 'sofa');` produces `{-1, 151}`. That's because first `BookExcerpt.excerpt` doesn't have the word `sofa`, while the second has it at index 151.

- `str_split()` lets you make an array from a string, split however you like. Most common is to split by `' '` to separate words:

```edgeql-repl
edgedb> SELECT str_split('Oh, hear me! hear me! Let me go! let me go! let me go!', ' ');

{
  [
    'Oh,',
    'hear',
    'me!',
    'hear',
    'me!',
    'Let',
    'me',
    'go!',
    'let',
    'me',
    'go!',
    'let',
    'me',
    'go!',
  ],
}
```

But this works too:

```edgeql
SELECT MinorVampire {
  names := (SELECT str_split(.name, 'n'))
};
```

Now the `n`s are all gone:

```
{
  default::MinorVampire {names: ['Woma', ' 1']},
  default::MinorVampire {names: ['Woma', ' 2']},
  default::MinorVampire {names: ['Woma', ' 3']},
  default::MinorVampire {names: ['Lucy Weste', 'ra']},
}
```

You can also split by `\n` to split by new line. You can't see it but from the point of view of the computer every new line has a `\n` in it. So this:

```edgeql
SELECT str_split('Oh, hear me!
hear me!
Let me go!
let me go!
let me go!', '\n');
```

will split it by line and give the following array:

```
{['Oh, hear me! ', 'hear me! ', 'Let me go! ', 'let me go! ', 'let me go!']}
```

- Two functions called `re_match()` (for the first match) and `re_match_all()` (for all matches) if you know how to use [regular expressions](https://en.wikipedia.org/wiki/Regular_expression) (regexes) and want to use those. This could be useful because the book Dracula was written over 100 years ago and has different spelling sometimes. The word `tonight` for example is always written with the older `to-night` spelling in Dracula. We can use these functions to take care of that:

```edgeql-repl
edgedb> SELECT re_match_all('[Tt]o-?night', 'Dracula is an old book, so the word tonight is written to-night. Tonight we know how to write both tonight and to-night.');
{['tonight'], ['to-night'], ['Tonight'], ['tonight'], ['to-night']}
```

The function signature is `std::re_match(pattern: str, string: str) -> array<str>`, and as you can see the pattern comes first, then the string. The pattern `[Tt]o-?night` means words that:

- start with a `T` or a `t`,
- then have an `o`,
- maybe have an `-` in between,
- and end in `night`,

so it gives: `{['tonight'], ['to-night']}`.

And to match anything, you can use the wildcard character: `.`

## Two more notes on `index on`

By the way, `index on` can also be used on expressions that you make yourself. This is especially useful now that we know all of these string functions. For example, if we always need to query a `City`'s name along with its population, we could index in this way:

```sdl
type City extending Place {
  annotation description := 'Anything with 50 or more buildings is a city - anything else is an OtherPlace';
  property population -> int64;
  index on (.name ++ ': ' ++ <str>.population);
}
```

Also don't forget that you can add add an annotation to this as well. `(.name ++ ': ' + <str>.population)` might be a good case for an annotation if you think readers of the code might not know what it's for:

```
type City extending Place {
    annotation description := 'Anything with 50 or more buildings is a city - anything else is an OtherPlace';
    property population -> int64;
    index on (.name ++ ': ' ++ <str>.population) {
      annotation title := 'Lists city name and population for use in game function get_city_names';
    }
}
```

`get_city_names` isn't a real function; we're just pretending that it's used somewhere in the game and is important to remember.

[Here is all our code so far up to Chapter 16.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you split all the `Person` names into two strings if they have two words, and ignore any that don't have exactly two words?

2. How would you display all the `Person` names and where the string 'ma' is in their name?

   Hint: this uses the function `find()`.

3. How would you index on the `pen_name` property for type Person?

   Hint: try using `describe type Person as SDL` to take a look at it the `pen_name` property again.

4. How would you display the name of every `Person` in uppercase followed by a space and then the same name in lowercase?

   Hint: the [str_repeat()](https://www.edgedb.com/docs/edgeql/funcops/string#function::std::str_repeat) function could help (though there is more than one way to do it)

5. How would you use `re_match_all()` to display all the `Person.name`s with `Crewman` in the name? e.g. Crewman 1, Crewman 2, etc.

   Hint: [Here are some basic concepts](https://en.wikipedia.org/w/index.php?title=Regular_expression&oldid=988356211#Basic_concepts) if you want a quick read on regular expressions.

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _The truth about Renfield._
