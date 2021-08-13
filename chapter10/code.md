```
# Schema:
START MIGRATION TO {
  module default {
    abstract type Person {
      property name -> str {
        constraint exclusive;
      }
      multi link places_visited -> Place;
      multi link lover -> Person;
      property strength -> int16;
      property first_appearance -> cal::local_date;
      property last_appearance -> cal::local_date;
      property age -> int16;
      property title -> str;
      property degrees -> str;
      property conversational_name := .title ++ ' ' ++ .name IF EXISTS .title ELSE .name;
      property pen_name := .name ++ ', ' ++ .degrees IF EXISTS .degrees ELSE .name;
    }

    type PC extending Person {
      required property transport -> Transport;
    }

    type NPC extending Person {
      overloaded property age {
        constraint max_value(120)
    }
      overloaded multi link places_visited -> Place {
        default := (SELECT City filter .name = 'London');
      }
    }

    type Vampire extending Person {
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

    type City extending Place {
      property population -> int64;
    }

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

    abstract type HasNumber {
      required property number -> int16;
    }
    
    type Crewman extending HasNumber, Person {
    }

   scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;

    type Sailor extending Person {
      property rank -> Rank;
    }

    type Ship {
      required property name -> str;
      multi link sailors -> Sailor;
      multi link crew -> Crewman;
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
  lover := (SELECT DETACHED NPC Filter .name = 'Jonathan Harker'),
  places_visited := (SELECT City FILTER .name = 'London'),
};

UPDATE Person FILTER .name = 'Jonathan Harker'
  SET {
    lover := (SELECT DETACHED Person FILTER .name = 'Mina Murray')
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

INSERT Sailor {
  name := 'The Captain',
  rank := Rank.Captain
};

INSERT Sailor {
  name := 'Petrofsky',
  rank := Rank.FirstMate
};

INSERT Sailor {
  name := 'The First Mate',
  rank := Rank.SecondMate
};

INSERT Sailor {
  name := 'The Cook',
  rank := Rank.Cook
};

FOR n IN {1, 2, 3, 4, 5}
  UNION (
  INSERT Crewman {
  number := n,
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16),
});

INSERT Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};

INSERT NPC {
  name := 'Lucy Westenra',
  places_visited := (SELECT City FILTER .name = 'London')
};

FOR character_name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  UNION (
    INSERT NPC {
    name := character_name,
    lover := (SELECT Person FILTER .name = 'Lucy Westenra'),
});

UPDATE NPC FILTER .name = 'Lucy Westenra'
SET {
  lover := (
    SELECT Person FILTER .name IN {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  )
};

UPDATE NPC FILTER .name = 'Lucy Westenra'
  SET {
    lover := (SELECT NPC FILTER .name = 'Arthur Holmwood'),
};

UPDATE NPC FILTER .name in {'John Seward', 'Quincey Morris'}
  SET {
    lover := {} # 😢
};

INSERT NPC {
  name := 'Renfield',
  first_appearance := cal::to_local_date(1887, 5, 26),
  strength := 10,
};

INSERT City {
  name := 'Whitby',
  population := 14400
};

FOR data in {('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)}
  UNION (
    UPDATE City FILTER .name = data.0
    SET {
    population := data.1
});

INSERT NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := 'M.D., Ph. D. Lit., etc.'
};
```
