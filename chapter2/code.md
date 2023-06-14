```
# Schema:

module default {
  abstract type Person {
    required name: str;
    multi places_visited: City;
  }

  type PC extending Person {
    required class: Class;
  }

  type NPC extending Person {
  }
  
  type City {
    required name: str;
    modern_name: str;
    important_places: array<str>;
  }

  scalar type Class extending enum<Rogue, Mystic, Merchant>;
}


# Data:

insert City {
  name := 'Munich',
};

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
  places_visited := City,
  class := Class.Mystic,
};

insert City {
  name := ''
};
```
