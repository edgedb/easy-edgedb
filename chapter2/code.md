```
# Schema:

module default {
  # Scalar types

  scalar type Class extending enum<Rogue, Mystic, Merchant>;

  # Abstract object types

  abstract type Person {
    required name: str;
    multi places_visited: City;
  }

  # Object types

  type City {
    required name: str;
    modern_name: str;
    important_places: array<str>;
  }

  type NPC extending Person;

  type PC extending Person {
    required class: Class;
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

insert City { name := '' };
```
