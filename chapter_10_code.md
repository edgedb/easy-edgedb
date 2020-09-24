Schema:

```
START MIGRATION TO {
  module default {

  abstract type Person {
    property first -> str;
    property last -> str;
    property title -> str;
    property degrees -> str;
    property name -> str {
        constraint exclusive
    }
    property title -> str;
    property degrees -> str;
    property conversational_name := .title ++ ' ' ++ .name IF EXISTS .title ELSE .name;
    property pen_name := .name ++ ', ' ++ .degrees IF EXISTS .degrees ELSE .name;

    property strength -> int16;
    MULTI LINK places_visited -> Place;
    MULTI LINK lover -> Person;
    property first_appearance -> cal::local_date;
    property last_appearance -> cal::local_date;
  }

    type Vampire extending Person {    
    property age -> int16;
    MULTI LINK slaves -> MinorVampire;
    }

    type MinorVampire extending Person {
    }

    abstract type HasNumber {
        required property number -> int16;
    }

    type Crewman extending HasNumber, Person {

    }

    scalar type Rank extending enum<'Captain', 'First mate', 'Second mate', 'Cook'>;

    type Sailor extending Person {
        property rank -> Rank;
    }

    type Ship {
        property name -> str;
        MULTI LINK sailors -> Sailor;
        MULTI LINK crew -> Crewman;
    }

    type PC extending Person {
      required property transport -> Transport;
    }

    scalar type HumanAge extending int16 {
      constraint max_value(120);
    }

    type NPC extending Person {
      property age -> HumanAge;
      OVERLOADED MULTI LINK places_visited -> Place {
        default := (SELECT City filter .name = 'London');
      }
    }

    abstract type Place {
      required property name -> str {
          constraint exclusive;
      };
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

    scalar type Transport extending enum<'feet', 'train', 'horse-drawn carriage'>;

    type Date {
      required property date -> str;
      property local_time := <cal::local_time>.date;
      property hour := .date[0:2];
      property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
    }
  }
};

POPULATE MIGRATION;
COMMIT MIGRATION;
```

Data:

```
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := (SELECT Place FILTER .name in {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'}),
  strength := 5,
};

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
  population := 9100,
};

INSERT City {
    name := 'London',
};

INSERT Country {
    name := 'Romania',
};

INSERT Country {
    name := 'Slovakia'
};

INSERT Castle {
    name := 'Castle Dracula',
    doors := [6, 19, 10],
};

INSERT PC {
  name := 'Emil Sinclair',
  places_visited := City,
  transport := <Transport>'horse-drawn carriage',
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
 }
};

INSERT NPC {
    name := 'The innkeeper',
    age := 30
};

INSERT NPC {
  name := 'Mina Murray',
  lover := (SELECT DETACHED NPC Filter .name = 'Jonathan Harker' LIMIT 1),
  places_visited := (SELECT City FILTER .name = 'London'),
};

INSERT Crewman {  # Repeat this insert five times
  number := count(DETACHED Crewman) +1,
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16),
};

INSERT Sailor {
  name := 'The Captain',
  rank := 'Captain'
};

INSERT Sailor {
  name := 'The First Mate',
  rank := 'First mate'
};

INSERT Sailor {
  name := 'The Second Mate',
  rank := 'Second mate'
};

INSERT Sailor {
  name := 'The Cook',
  rank := 'Cook'
};

INSERT Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};

FOR character_name IN {'John Seward', 'Quincey Morris'}
  UNION (
    INSERT NPC {
    name := character_name,
    first_appearance := cal::to_local_date(1887, 5, 26),
});

INSERT NPC {
  name := 'Lucy Westenra',
  first_appearance := cal::to_local_date(1887, 5, 26),
};

INSERT NPC {
  name := 'Arthur Holmwood',
  first_appearance := cal::to_local_date(1887, 5, 26),
  lover := (SELECT DETACHED NPC FILTER .name = 'Lucy Westenra'),
};

UPDATE NPC FILTER .name = 'Lucy Westenra'
SET {
  lover := (
    SELECT Person FILTER .name = 'Arthur Holmwood'
  )
};

INSERT NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := 'M.D., Ph. D. Lit., etc.'
};
```
