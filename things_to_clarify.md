Hi all,

Here's a quick rundown of all the notes I jotted down - in no particular order - while writing the book that I've been unable to find the answer to myself or am curious to hear more insight on.

## EdgeDB in Windows

Right now this is how I get EdgeDB to work on Ubuntu inside Windows:

```
rm -rf /home/mithridates/.local/share/edgedb/data/default
sudo mkdir -p /run/user/1000
sudo chown $(id -u):$(id -g) /run/user/1000
edgedb server init --nightly default --start-conf=manual --overwrite
edgedb server start --foreground default
edgedb --admin -H /run/user/1000/edgedb-default/.s.EDGEDB.admin.10700
```

(Last line is in a new terminal)

Should something similar to this be added to the documentation for casual Windows users?

## Migrations

Right now I do migrations by copying and pasting the whole schema into the command line terminal in Ubuntu:

```
START MIGRATION TO {
  module default {
    type Person {
      required property name -> str;
      multi link places_visited -> City;
    }
    type City {
      required property name -> str;
      property modern_name -> str;
    }
  }
};

POPULATE MIGRATION;
COMMIT MIGRATION;
```

By about Chapter 8 it starts getting weighty so I shrink the size of the window to speed it up. Is there any other way to do this?


## Ranges

Are there any plans for ranges? I have an insert in the book that kind of gets around it with this: 

```
FOR n IN {1, 2, 3, 4, 5}
  UNION (
  INSERT Crewman {
  number := n
});
```

(These Crewman types at this point in the book just have numbers instead of names)

but it feels like maybe there's a better way to do it that I've missed.

## Self-referencing inserts

In Chapter 4 there's an insert of Jonathan Harker and Mina Murray who are set to get married once Jonathan gets back from Romania. Their NPC type extends Person which has a link called `lover`:

```
abstract type Person {
  required property name -> str;
  multi link places_visited -> City;
  link lover -> Person;
}
```

I've tried a few ways to insert them at the same time but get this message:

```
error: invalid reference to default::NPC: self-referencing INSERTs are not allowed
```

And on that note,

## Objects that link to each other


I wrote the following in Chapter 6:

>Then we can INSERT the MinorVampire type at the same time as we insert the information for Count Dracula. But first let's remove the link from MinorVampire, because we don't want two objects linking to each other.

What I originally had was a Vampire type with a `slaves` link that links to MinorVampire, and a MinorVampire type with a `master` link that links to Vampire, which quickly felt like link overkill and also resulted in not being able delete either of the two (without using DDL to outright modify the types) because they both linked to each other.

But then again, maybe objects should in principle almost never be deleted (it is a database, after all) and perhaps cross-linking isn't a problem then? I'm curious what best practices are regarding this.

## Dropping module default

`DROP MODULE DEFAULT;` gives this error:

```
Error: ERROR: InternalServerError: cannot drop schema edgedb_66c941f2-27b5-11eb-9e6c-cf22d7dd15f3 because other objects depend on it
  Hint: This is most likely a bug in EdgeDB. Please consider opening an issue ticket at https://github.com/edgedb/edgedb/issues/new?template=bug_report.md
  Server traceback:
      edb.errors.InternalServerError: cannot drop schema edgedb_66c941f2-27b5-11eb-9e6c-cf22d7dd15f3 because other objects depend on it
```

That makes sense because there are a lot of objects depending on it, but in the [documentation here](https://www.edgedb.com/docs/edgeql/ddl/modules#statement::DROP-MODULE) it seems like it should work:

>DROP MODULE removes an existing module from the current database. All schema items and data contained in the module are removed as well.

And I can't seem to find any command similar to `FOR object in default DELETE object` to wipe them all out and then `DROP MODULE` now that no objects depend on it.

## random()

I use random() to assign each character a strength in one part of the book and have noticed something curious about it:


```
 SELECT(UPDATE Person
   SET {
strength := <int16>(round(random() * 50))
})
  {__type__: {name},
   strength};
```

It looks like the random number is seeded just once per type that extends Person because each of them gets the same strength:

```
{
  Object {__type__: Object {name: 'default::MinorVampire'}, strength: 14},
  Object {__type__: Object {name: 'default::MinorVampire'}, strength: 14},
  Object {__type__: Object {name: 'default::MinorVampire'}, strength: 14},
  Object {__type__: Object {name: 'default::MinorVampire'}, strength: 14},
  Object {__type__: Object {name: 'default::Vampire'}, strength: 6},
  Object {__type__: Object {name: 'default::NPC'}, strength: 2},
  Object {__type__: Object {name: 'default::NPC'}, strength: 2},
  Object {__type__: Object {name: 'default::NPC'}, strength: 2},
  Object {__type__: Object {name: 'default::NPC'}, strength: 2},
  Object {__type__: Object {name: 'default::NPC'}, strength: 2},
  Object {__type__: Object {name: 'default::NPC'}, strength: 2},
  Object {__type__: Object {name: 'default::NPC'}, strength: 2},
  Object {__type__: Object {name: 'default::NPC'}, strength: 2},
}
```

Is there a way to work around this?

## enums

Enums still don't work for me as of Alpha 6 - even this generates an error:

```
START MIGRATION TO {
  module default {
    scalar type Choice extending enum<Good, Bad>;
  }
};

POPULATE MIGRATION;
COMMIT MIGRATION;
```

``` AttributeError: 'TypeName' object has no attribute 'val'```

Right now Easy EdgeDB still describes the Alpha 5 enums because of this.

What I'm most curious about is whether enums will be able to hold data (like a string or something else) that can then be accessed. One of the samples in the book generates a url for a geographical position by selecting east or west, and after Alpha 6 I changed it to a boolean instead (fortunately the only choices are east and west). Would be cool to be able to use an enum again for this.

## DELETE

I haven't written anything about using DELETE yet but it looks like there will just be a warning message, which I think is the best way to do it. Is it safe to start writing about how to use DELETE under this assumption?

## Functions returning "record"

Here's an example of a function I tried to make that resulted in an error:

```
create function find_string(input: str) -> SET OF tuple<str, int64>
  using(
    SELECT(Person.name, find(Person.name, input))
); 
```

(This one wasn't intended to be in the book, just experimenting)

Here's the error:

```
ERROR: InternalServerError: a column definition list is only allowed for functions returning "record".
```

A bit of searching shows quite a few Postgres discussions on how to get around it but I don't see anything in our docs about returning "record".

