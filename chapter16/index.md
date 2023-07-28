---
tags: Indexing, String Functions
---

# Chapter 16 - Is Renfield telling the truth?

> Arthur Holmwood's father has died and now Arthur is the head of the house. His new title is Lord Godalming, and he has a lot of money. With this money he helps the team to find the houses where Dracula has hidden his boxes.
>
> Meanwhile, Van Helsing is curious and asks John Seward if he can meet Renfield. Van Helsing is surprised to see that Renfield is very educated and well-spoken. Renfield talks about Van Helsing's research, politics, history, and so on - he doesn't seem crazy at all! But the next time Van Helsing sees him Renfield doesn't want to talk and just calls him an idiot. Very confusing.
>
> One night, Renfield becomes very serious and asks the men to let him leave. Renfield says: “Don’t you know that I am sane and earnest...a sane man fighting for his soul? Oh, hear me! hear me! Let me go! let me go! let me go!” The men want to believe Renfield, but can't trust him. Finally Renfield stops and calmly says: “Remember, later on, that I did what I could to convince you tonight.”

## `index on` for quicker lookups

We're getting closer to the end of the book and there is a lot of data that we haven't entered yet. There is also a lot of data from the book that might be useful but we're not ready to organize yet. Fortunately, the original text of Dracula is organized into letters, diary entries, newspaper reports, etc. that begin with the date and sometimes the time. They tend to start out like this:

```
Dr. Seward’s Diary.
1 October, 4 a. m.—Just as we were about to leave the house...

Letter, Van Helsing to Mrs. Harker.
“24 September.
“Dear Madam...

Mina Murray’s Journal.
8 August. — Lucy was very restless all night, and I, too, could not sleep...
```

This is convenient for us. With this we can make a type that holds a date and a string from the book for us to search through later. Let's call it `BookExcerpt` (an "excerpt" meaning one small part of a larger text). This type has a keyword that we haven't seen before:

```sdl
type BookExcerpt {
  required date: cal::local_datetime;
  required excerpt: str;
  index on (.date);
  required author: Person;
}
```

The {ref}` ``index on (.date)`` <docs:ref_datamodel_indexes>` part is new, and is a way to make queries that use `filter`, `order by`, or `group` faster. These three operations are faster with `index on` because now the database doesn't need to scan the whole set of objects in sequence to find objects that match.

```{eval-rst}
.. note::
`index` is good in limited quantities, but you don't want to index everything. Here is why:

- It makes the queries faster, but increases the database size.
- This may make `insert`s and `update`s slower if you have too many.

If there were no downside to indexing, EdgeDB would just index everything for you by default. Since there is a downside, indexing only happens when you say so. A good rule of thumb for indexes might be to think of them in the context of a real book:

- Faster search but database size increases: You can find content inside a book yourself, but you could also add an index. An index increases the book size somewhat, but helps you find content faster.
- Inserts take longer: each book you print has that many extra pages to print.
- Updates take longer: If you just update the content in a book, the update itself is the end of the operation. But if you have an index, then you'll have to update that as well to match the changes.
```

Indexing on `date` on the `BookExcerpt` type seems like a good idea because the `BookExcerpt` data all comes from a single book, is inserted once, and doesn't need to be updated. An index on a property inside `PC` probably makes less sense, because `PC` objects are going to be inserted and updated all the time.

EdgeDB will automatically index in a few cases. You don't need to think about adding `index` on:

- links,
- exclusive constraints for a property.

The automatically generated `id` property on every item is also always indexed.

So let's do a migration and insert two book excerpts. The strings in these entries are very long (pages long, sometimes) so we will only show the beginning and the end here:

```edgeql
insert BookExcerpt {
  date := cal::to_local_datetime(1893, 10, 1, 4, 0, 0),
  author := assert_single((select Person filter .name = 'John Seward')),
  excerpt := 'Dr. Seward\'s Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once..."You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night."',
};
```

```edgeql
insert BookExcerpt {
  date := cal::to_local_datetime(1893, 10, 1, 5, 0, 0),
  author := assert_single((select Person filter .name = 'Jonathan Harker')),
  excerpt := '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
};
```

Then later on we could do this sort of query to get all the entries in order and displayed as JSON. Perhaps the `PC` objects can visit a library where they can search for game details, and this requires sending a message in JSON format to the software that displays it on the screen:

```edgeql
select <json>(
  select BookExcerpt {
    date,
    author: {
      name
    },
    excerpt
  } order by .date
);
```

Here's the JSON output (remember, set with `\set output-format json-pretty`) which looks pretty nice:

```
{
  "date": "1893-10-01T04:00:00",
  "author": {"name": "John Seward"},
  "excerpt": "Dr. Seward's Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once...\"You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night.\""
}
{
  "date": "1893-10-01T05:00:00",
  "author": {"name": "Jonathan Harker"},
  "excerpt": "1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her."
}
```

After this, we can add a link to our `Event` type to join it to our new `BookExcerpt` type. `Event` now looks like this:

```sdl
type Event {
  required description: str;
  required start_time: cal::local_datetime;
  required end_time: cal::local_datetime;
  required multi place: Place;
  required multi people: Person;
  multi excerpt: BookExcerpt; # Only this is new
  location: tuple<float64, float64>;
  east: bool;
  property url := get_url() ++ <str>.location.0 
    ++ '_N_' ++ <str>.location.1 ++ '_' ++ ('E' if .east else 'W');
}
```

You can see that `description` is a short string that we write, while `excerpt` links to the longer pieces of text that come directly from the book.

With this done, let's get back to the `BookExcerpt` type and the indexing it uses. We now know that indexing speeds up `filter`, `order` and `group` queries, but by how much? Fortunately EdgeDB has a keyword that can provide some insight into this.

## The 'analyze' keyword

One particularly nice addition to EdgeDB 3.0 which released in 2023 is the `analyze` keyword, which just might be EdgeDB's easiest keyword to use.

## More functions for strings

The {ref}`functions for strings <docs:ref_std_string>` can be particularly useful when doing queries on our `BookExcerpt` type (or `BookExcerpt` via `Event`). One is called {eql:func}`docs:std::str_lower` and makes strings lowercase:

```
db> select str_lower('RENFIELD WAS HERE');
{'renfield was here'}
```

Here it is in a longer query:

```edgeql
select BookExcerpt {
  excerpt,
  length := (<str>(select len(.excerpt)) ++ ' characters'),
  the_date := (select (<str>.date)[0:10]),
} filter contains(str_lower(.excerpt), 'mina');
```

It uses `len()` which is then cast to a string, and `str_lower()` to compare against `.excerpt()` by making it lowercase first. It also slices the `cal::local_datetime` into a string so it can just print indexes 0 to 10. Here is the output:

```
{
  default::BookExcerpt {
    excerpt: '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
    length: '182 characters',
    the_date: '1893-10-01',
  },
}
```

Another way to make this `the_date` parameter is with the {eql:func}`docs:std::to_str` method, which (as you can probably guess) will turn it into a string. This function also allows us to change the format of the date depending on how readable we want to make it:

```edgeql
select BookExcerpt {
  excerpt,
  length := (<str>(select len(.excerpt)) ++ ' characters'),
  the_date := (select to_str(.date)),
  the_date_pretty := (select to_str(.date, 'YYYY-MM-DD')),
  the_date_time_pretty := (select to_str(.date, 'YYYY-MM-DD HH:MM:SS')),
  the_date_verbose := (select to_str(.date, 'The DD of MM, YYYY'))
} filter contains(str_lower(.excerpt), 'mina');
```

Here's the output for that long query:

```
{
  default::BookExcerpt {
    excerpt: '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
    length: '182 characters',
    the_date: '1893-10-01T05:00:00',
    the_date_pretty: '1893-10-01',
    the_date_time_pretty: '1893-10-01 05:10:00',
    the_date_verbose: 'The 01 of 10, 1893',
  },
}
```

Some other functions for strings are:

- `find()` This gives the index of the first match it finds, and returns `-1` if it can't find anything:

`select find(BookExcerpt.excerpt, 'sofa');` produces `{-1, 151}`. That's because first `BookExcerpt.excerpt` doesn't have the word `sofa`, while the second has it at index 151.

- `str_split()` lets you make an array from a string, split however you like. Most common is to split by `' '` to separate words:

```
db> select str_split('Oh, hear me! hear me! Let me go! let me go! let me go!', ' ');
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

But we can choose a letter to split at too:

```edgeql
select MinorVampire {
  names := (select str_split(.name, 'n'))
};
```

Now, the names have been split into arrays at each instance of `n`:

```
{
  default::MinorVampire {names: ['Vampire Woma', ' 1']},
  default::MinorVampire {names: ['Vampire Woma', ' 2']},
  default::MinorVampire {names: ['Vampire Woma', ' 3']},
  default::MinorVampire {names: ['Lucy']},
  default::MinorVampire {names: ['Billy']},
  default::MinorVampire {names: ['Bob']},
}
```

You can also split by `\n` to split by new line. You can’t see it as a human but from the point of view of the computer every new line has a `\n` in it. Take this for example:

```edgeql
select str_split('Oh, hear me!
hear me!
Let me go!
let me go!
let me go!', '\n');
```

The output is an array of the text split by line:

```
{['Oh, hear me!', 'hear me!', 'Let me go!', 'let me go!', 'let me go!']}
```

- Two functions called `re_match()` (for the first match) and `re_match_all()` (for all matches) if you know how to use [regular expressions](https://en.wikipedia.org/wiki/Regular_expression) (regexes) and want to use those. This could be useful because the book Dracula was written over 100 years ago and has different spelling sometimes. The word `tonight` for example is always written with the older `to-night` spelling in Dracula. We can use these functions to take care of that:

```
db> select re_match_all('[Tt]o-?night', 'Dracula is an old book,
so the word tonight is written to-night.
Tonight we know how to write both tonight and to-night.');
{['tonight'], ['to-night'], ['Tonight'], ['tonight'], ['to-night']}
```

The function signature is `std::re_match_all(pattern: str, string: str) -> set of array<str>`, and as you can see the pattern comes first, then the string. The pattern `[Tt]o-?night` means words that:

- start with a `T` or a `t`,
- then have an `o`,
- maybe have an `-` in between,
- and end in `night`,

Here is the output: `{['tonight'], ['to-night'], ['Tonight'], ['tonight'], ['to-night']}`

And to match anything, you can use the wildcard character: `.`

```
db> select re_match_all('.oo.', 'Noo, Lord Dracula, why did you lock the door?');
{['Noo,'], ['door']}
```

The `.` wildcard operator still determines the length of the slice of the string to match on, so you can use more of them to lengthen the part of the string in which we are looking for a match.

```
db> select re_match_all('.h...oo..', 'Noo, Lord Dracula, why did you lock the door?');
{['the door?']}
```

## Two more notes on `index on`

By the way, `index on` can also be used on expressions that you we make ourselves. This is especially useful now that we know all of these string functions. For example, if we always need to query a `City`'s name along with its population, we could index in this way:

```sdl
type City extending Place {
  annotation description := 'A place with 50 or more buildings. Anything else is an OtherPlace';
  population: int64;
  index on (.name ++ ': ' ++ <str>.population);
}
```

Also don't forget that you can add add an annotation to this as well. `(.name ++ ': ' + <str>.population)` might be a good case for an annotation if you think readers of the code might not know what it's for:

```
type City extending Place {
    annotation description := 'A place with 50 or more buildings. Anything else is an OtherPlace';
    population: int64;
    index on (.name ++ ': ' ++ <str>.population) {
      annotation title := 'Lists city name and population for use in game function get_city_names';
    }
}
```

`get_city_names` isn't a real function; we're just pretending that it's used somewhere in the game and is important to remember.

## Changing a property to a link

Our `Place` type has had a property called `important_places` for quite some time now: almost since the beginning of the book! This property is just an `<array<str>>`, which is better than nothing but not as good as a link. Let's remind ourselves what `important_places` are in the database at the moment:

```edgeql
select Place {
  name, important_places
  } filter exists .important_places;
```

It turns out that we have a total of four `important_places`, all of which are located inside `City` objects:

```
{
  default::City {name: 'Whitby', important_places: ['Whitby Abbey']},
  default::City {name: 'Bistritz', important_places: ['Golden Krone Hotel']},
  default::City {
    name: 'Buda-Pesth',
    important_places: ['Hospital of St. Joseph and Ste. Mary', 'Buda-Pesth University'],
  },
}
```

So let's think of a type we could make for these places. Our types that extend `Place` are all related to whether they have coffins or not, which is important in our game because places with coffins have a higher chance of being terrorized by vampires. These types so far are `City`, `Country`, `OtherPlace`, and `Castle` (which includes castle towns): all fairly large places that can be explored. But `important_places` seems more like a list of locations that are so tiny that keeping track of the number of coffins doesn't make any sense. In other words, it doesn't matter if the Golden Krone Hotel has coffins or not: it only matters if Bistritz has coffins in it or not.

So let's create a new type that is independent of the abstract type `Place`, and just call it `Landmark`:

```sdl
type Landmark {
  required name: str;
  multi context: str;
}
```

Inside the `context` property we can just add parts of the book that reference the `Landmark`. We could have chosen `<array<str>>` but we haven't used `multi` properties much yet, so let's give that a try.

Eventually we will change our `Place` type so that `important places` changes from an `<array<str>>` to a `multi` link to `Landmark` that looks like the type below. The code below shows the final form for `Place`, but don't change it to this yet!

```sdl
abstract type Place extending HasCoffins {
  required name: str {
    delegated constraint exclusive;
  }
  modern_name: str;
  multi important_places: Landmark;
}
```

We don't want to make this change yet because we will lose our data if we just change the type of `important_ places` to something else. Instead, we can change our schema to add the `Landmark` type in addition to a new link on `Place` we'll call `linked_important_places`. Let's make those changes and migrate the schema now:

```sdl
type Landmark {
  required name: str;
  multi context: str;
}

abstract type Place extending HasCoffins {
  required name: str {
    delegated constraint exclusive;
  }
  modern_name: str;
  important_places: array<str>;
  multi linked_important_places: Landmark;
}
```

After doing a migration we now have the type `Landmark`, and our `important_places` data is untouched. Now we can create some `Landmark` objects. Let's just insert them with the required `name` property to start. We can use the `array_unpack()` method to grab each `str` from all the `important_places` in all the `Place` objects we have so far, and use that to do the inserts:

```edgeql
for place_name in select (array_unpack(Place.important_places))
union (insert Landmark {
  name := place_name
});
```

After that we can update our `Place` objects to have the link called `linked_important_places` to the `Landmark` objects that we have just inserted.

```edgeql
update Place filter exists .important_places set {
  linked_important_places := (
    select Landmark filter .name in array_unpack(Place.important_places))
  };
```

This has returned three objects, so looks like it worked! Let's do a query to make sure:

```edgeql
select Place {
  name,
  important_places,
  linked_important_places: { name }
  } filter exists .linked_important_places;
```

The output shows us that it worked! We can compare the existing data inside `important_places` and see that we now have `Landmark` objects with the same name as before.

```
{
  default::City {
    name: 'Whitby',
    important_places: ['Whitby Abbey'],
    linked_important_places: {default::Landmark {name: 'Whitby Abbey'}},
  },
  default::City {
    name: 'Bistritz',
    important_places: ['Golden Krone Hotel'],
    linked_important_places: {default::Landmark {name: 'Golden Krone Hotel'}},
  },
  default::City {
    name: 'Buda-Pesth',
    important_places: ['Hospital of St. Joseph and Ste. Mary', 'Buda-Pesth University'],
    linked_important_places: {
      default::Landmark {name: 'Hospital of St. Joseph and Ste. Mary'},
      default::Landmark {name: 'Buda-Pesth University'},
    },
  },
}
```

And now that our data is safe, we can now do two more migrations. First we remove `important_places` and migrate.

```sdl
abstract type Place extending HasCoffins {
  required name: str {
    delegated constraint exclusive;
  }
  modern_name: str;
  multi linked_important_places: Landmark;
}
```

The CLI will ask us if we wanted to `drop property 'important_places' of object type 'default::Place'?`, to which the answer is yes.

And then we change the link name from `linked_important_places` to `important_places`, and do a migration again.

```sdl
abstract type Place extending HasCoffins {
  required name: str {
    delegated constraint exclusive;
  }
  modern_name: str;
  multi important_places: Landmark;
}
```

This time the CLI asks us if we renamed `link 'linked_important_places' of object type 'default::Place' to 'important_places'?`, to which the answer once again is yes.

Let's do a final check to make sure that the data is there:

```edgeql
select Place {
  name,
  important_places: { name }
} filter exists .important_places;
```

And the data is all there! Beautiful.

```
{
  default::City {name: 'Whitby', important_places: {default::Landmark {name: 'Whitby Abbey'}}},
  default::City {
    name: 'Bistritz',
    important_places: {default::Landmark {name: 'Golden Krone Hotel'}},
  },
  default::City {
    name: 'Buda-Pesth',
    important_places: {
      default::Landmark {name: 'Hospital of St. Joseph and Ste. Mary'},
      default::Landmark {name: 'Buda-Pesth University'},
    },
  },
}
```

As always, the code at the end of the chapter will feature the inserts we would have made if we had had the current schema to begin with. So the insert for Buda-Pesth for example will now look like this, with the `Landmark` inserts at the same time.

```edgeql
insert City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest',
  important_places := {
    (insert Landmark {name := 'Hospital of St. Joseph and Ste. Mary'}),
    (insert Landmark {name := 'Buda-Pesth University'})
  }
};
```

[Here is all our code so far up to Chapter 16.](code.md)

<!-- quiz-start -->

## Time to practice

1. How would you write a query to show all the `Person` names split into an array of two strings if they have two names and ignored if they don't have exactly two names?

2. How would you display all the `Person` names and where the string 'ma' is in their name?

   Hint: this uses the function `find()`.

3. How would you index on the `pen_name` property for type Person?

   Hint: try using `describe type Person as SDL` to take a look at it the `pen_name` property again.

4. How would you display the name of every `Person` in uppercase followed by a space and then the same name in lowercase?

   Hint: the {eql:func}`docs:std::str_upper` function could help (though you will also need another function)

5. How would you use `re_match_all()` to display all the `Person.name`s with `Crewman` in the name? e.g. Crewman 1, Crewman 2, etc.

   Hint: [Here are some basic concepts](https://en.wikipedia.org/w/index.php?title=Regular_expression&oldid=988356211#Basic_concepts) if you want a quick read on regular expressions.

[See the answers here.](answers.md)

<!-- quiz-end -->

__Up next:__ _The truth about Renfield._
