```
# Schema:
START MIGRATION TO {
  module default {
    abstract type Person {
      required property name -> str {
        constraint exclusive;
      }
      multi link places_visited -> Place;
      link lover -> Person;
      property strength -> int16;
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
      multi link slaves -> MinorVampire;
    }

    type MinorVampire extending Person {
    }
    
    abstract type Place {
      required property name -> str {
        constraint exclusive;
      }
      property modern_name -> str;
      property important_places -> array<str>;
    }

    type City extending Place;

    type Country extending Place;

    type OtherPlace extending Place;

    type Castle extending Place {
      property doors -> array<int16>;
    }

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

INSERT Country {
  name := 'France'
};

INSERT Country {
  name := 'Slovakia'
};

INSERT Castle {
    name := 'Castle Dracula',
    doors := [6, 19, 10],
};

INSERT City {
    name := 'London',
};

INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := (SELECT Place FILTER .name IN {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'})
};

INSERT NPC {
  name := 'The innkeeper',
  age := 30,
};

INSERT NPC {
  name := 'Mina Murray',
  lover := assert_single((SELECT DETACHED NPC Filter .name = 'Jonathan Harker')),
  places_visited := (SELECT City FILTER .name = 'London'),
};

UPDATE Person FILTER .name = 'Jonathan Harker'
  SET {
    lover := assert_single((SELECT DETACHED Person FILTER .name = 'Mina Murray'))
};

INSERT Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (INSERT MinorVampire {
      name := 'Woman 1',
  }),
    (INSERT MinorVampire {
     name := 'Woman 2',
  }),
    (INSERT MinorVampire {
     name := 'Woman 3',
  }),
 },
   places_visited := (SELECT Place FILTER .name in {'Romania', 'Castle Dracula'})
};

UPDATE Person FILTER .name = 'Jonathan Harker'
  SET {
    strength := 5
};
```
