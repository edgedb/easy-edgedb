```
# Schema:
START MIGRATION TO {
  module default {
    abstract type Person {
      required property name -> str;
      multi link places_visited -> Place;
      link lover -> Person;
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

    type Time {
      required property date -> str;
      property local_time := <cal::local_time>.date;
      property hour := .date[0:2];
      property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
    }
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

INSERT Country {
  name := 'Hungary'
};

INSERT Country {
  name := 'Romania'
};

INSERT NPC {
  name := 'The innkeeper',
  age := 30,
};

INSERT City {
    name := 'London',
};

INSERT NPC {
  name := 'Mina Murray',
  lover := assert_single(
    (SELECT DETACHED NPC Filter .name = 'Jonathan Harker')
  ),
  places_visited := (SELECT City FILTER .name = 'London'),
};

INSERT Vampire {
  name := 'Count Dracula',
  places_visited := (SELECT Place FILTER .name = 'Romania'),
};
```
