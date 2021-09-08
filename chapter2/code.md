```
# Schema:
START MIGRATION TO {
  module default {
    abstract type Person {
      required property name -> str;
      multi link places_visited -> City;
    }

    type PC extending Person {
      required property transport -> Transport;
    }

    type NPC extending Person {
    }
    
    type City {
      required property name -> str;
      property modern_name -> str;
      property important_places -> array<str>;
    }

    scalar type Transport extending enum<Feet, Train, HorseDrawnCarriage>;
  }
};

POPULATE MIGRATION;
COMMIT MIGRATION;


# Data:

INSERT City {
  name := 'Munich',
};

INSERT City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest'
};

INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := ['Golden Krone Hotel'],
};

INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};

INSERT PC {
  name := 'Emil Sinclair',
  places_visited := City,
  transport := Transport.HorseDrawnCarriage,
};
```
