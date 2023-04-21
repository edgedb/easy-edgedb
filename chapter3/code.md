```
# Schema:
start migration to {
  module default {
    abstract type Person {
      required property name -> str;
      multi link places_visited -> Place;
    }

    type PC extending Person {
      required property transport -> Transport;
    }

    scalar type HumanAge extending int16 {
      constraint max_value(120);
    }

    type NPC extending Person {
      property age -> HumanAge;
    }

    type Vampire extending Person {
      property age -> int16;
    }
    
    abstract type Place {
      required property name -> str;
      property modern_name -> str;
      property important_places -> array<str>;
    }

    type City extending Place;

    type Country extending Place;

    scalar type Transport extending enum<Feet, Train, HorseDrawnCarriage>;
  }
};

populate migration;
commit migration;


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
  transport := Transport.HorseDrawnCarriage,
};

insert Country {
  name := 'Hungary'
};

insert Country {
  name := 'Romania'
};

insert Vampire {
  name := 'Count Dracula',
  places_visited := (SELECT Place FILTER .name = 'Romania'),
};

insert NPC {
  name := 'The innkeeper',
  age := 30,
};
```
