```
# Schema:

module default {
  abstract type Person {
    required name: str;
    multi places_visited: Place;
  }

  type PC extending Person {
    required class: Class;
  }

  scalar type HumanAge extending int16 {
    constraint max_value(120);
  }

  type NPC extending Person {
    age: HumanAge;
  }

  type Vampire extending Person {
    age: int16;
  }
  
  abstract type Place {
    required name: str;
    modern_name: str;
    important_places: array<str>;
  }

  type City extending Place;

  type Country extending Place;

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

insert Country {
  name := 'Hungary'
};

insert Country {
  name := 'Romania'
};

insert Vampire {
  name := 'Count Dracula',
  places_visited := (select Place filter .name = 'Romania'),
};

insert NPC {
  name := 'The innkeeper',
  age := 30,
};
```
