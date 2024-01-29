```
# Schema:

module default {

  # Globals

  global time := assert_single((select Time));

  # Scalar types

  scalar type Class extending enum<Rogue, Mystic, Merchant>;

  scalar type HumanAge extending int16 {
    constraint max_value(120);
  }

  scalar type SleepState extending enum <Asleep, Awake>;

  # Abstract object types

  abstract type Person {
    required name: str;
    multi places_visited: Place;
    lover: Person;
    is_single := not exists .lover;
  }

  abstract type Place {
    required name: str;
    modern_name: str;
    important_places: array<str>;
  }

  # Object types

  type City extending Place;

  type Country extending Place;

  type NPC extending Person {
    age: HumanAge;
  }

  type PC extending Person {
    required class: Class;
  }

  type Time {
    required clock: str;
    clock_time := <cal::local_time>.clock;
    hour := .clock[0:2];
    vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
          else SleepState.Awake;
  }

  type Vampire extending Person {
    age: int16;
  }
}

# Data:

insert Time { clock := '09:00:00' };

insert City { name := 'Munich' };

insert City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest'
};

insert City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};

insert NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};

insert PC {
  name := 'Emil Sinclair',
  class := Class.Mystic,
};

insert Country { name := 'Hungary' };

insert Country { name := 'Romania' };

insert NPC {
  name := 'The innkeeper',
  age := 30,
};

insert City { name := 'London' };

insert NPC {
  name := 'Mina Murray',
  lover := assert_single(
    (select detached NPC Filter .name = 'Jonathan Harker')
  ),
  places_visited := (select City filter .name = 'London'),
};

insert Vampire {
  name := 'Count Dracula',
  places_visited := (select Place filter .name = 'Romania'),
};
```
