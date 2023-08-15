```
# Schema:

module default {
  # Scalar types
  scalar type Class extending enum<Rogue, Mystic, Merchant>;
  
  scalar type HumanAge extending int16 {
    constraint max_value(120);
  }

  # Abstract object types

  abstract type Person {
    required name: str;
    multi places_visited: Place;
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

  type Vampire extending Person {
    age: int16;
  }
}

# Data:

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

insert Vampire {
  name := 'Count Dracula',
  places_visited := (select Place filter .name = 'Romania'),
};

insert NPC {
  name := 'The innkeeper',
  age := 30,
};
```
