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
      link former_self -> Person;
    }
    
    abstract type Place {
      required property name -> str {
        constraint exclusive;
      }
      property modern_name -> str;
      property important_places -> array<str>;
    }

    type City extending Place {
      annotation description := 'Anything with 50 or more buildings is a city - anything else is an OtherPlace';
      property population -> int64;
    }

    type Country extending Place;

    abstract annotation warning;

    type OtherPlace extending Place {
      annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
      annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
    }

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

    type Event {
      required property description -> str;
      required property start_time -> cal::local_datetime;
      required property end_time -> cal::local_datetime;
      required multi link place -> Place;
      required multi link people -> Person;
      property exact_location -> tuple<float64, float64>;
      property east -> bool;
      property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ '_N_' ++ <str>.exact_location.1 ++ '_' ++ ('E' if .east = true else 'W');
    }
  
    function fight(one: Person, two: Person) -> str
      using (
        (one.name ++ ' wins!') IF one.strength > two.strength ELSE (two.name ++ ' wins!')
      );

    function fight(names: str, one: int16, two: Person) -> str
      using (
        (names ++ ' win!') IF one > two.strength ELSE (two.name ++ ' wins!')
      );

    function visited(person: str, city: str) -> bool
      using (
        WITH person := (SELECT Person FILTER .name = person),
        SELECT city IN person.places_visited.name
      );
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
#  rank := Rank.Cook
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

INSERT Event {
  description := "Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.",
  start_time := cal::to_local_datetime(1887, 9, 11, 18, 0, 0),
  end_time := cal::to_local_datetime(1887, 9, 11, 23, 0, 0),
  place := (SELECT Place FILTER .name = 'Whitby'),
  people := (SELECT Person FILTER .name ILIKE {'%helsing%', '%westenra%', '%seward%'}),
  exact_location := (54.4858, 0.6206),
  east := false
};

UPDATE Person
  FILTER NOT EXISTS .strength
  SET {
    strength := <int16>round(random() * 5)
};

UPDATE Person filter .name = 'Lucy Westenra'
  SET {
  last_appearance := cal::to_local_date(1887, 9, 20)
};

WITH lucy := assert_single(
    (SELECT Person filter .name = 'Lucy Westenra')
)
INSERT Vampire {
  name := 'Count Dracula',
  age := 800,
  strength := 20,
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
    (INSERT MinorVampire {
     name := 'Lucy',
     former_self := lucy,
     first_appearance := lucy.last_appearance,
     strength := lucy.strength + 5,
    }),
 },
 places_visited := (SELECT Place FILTER .name in {'Romania', 'Castle Dracula'})
};

INSERT City {
  name := 'Exeter', 
  population := 40000
};

UPDATE Crewman
  SET {
    name := 'Crewman ' ++ <str>.number
};
```
