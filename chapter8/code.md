```
# Schema:

module default {
  abstract type Person {
    property name -> str {
      delegated constraint exclusive;
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
      delegated constraint exclusive;
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

  scalar type SleepState extending enum <Asleep, Awake>;
  
  type Time { 
    required property clock -> str; 
    property clock_time := <cal::local_time>.clock; 
    property hour := .clock[0:2]; 
    property sleep_state := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
      else SleepState.Awake;
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

insert Country {
  name := 'France'
};

insert Country {
  name := 'Slovakia'
};

insert Castle {
    name := 'Castle Dracula',
    doors := [6, 19, 10],
};

insert City {
    name := 'London',
};

insert NPC {
  name := 'Jonathan Harker',
  places_visited := (select Place filter .name in {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'})
};

insert NPC {
  name := 'The innkeeper',
  age := 30,
};

insert NPC {
  name := 'Mina Murray',
  lover := assert_single((select detached NPC Filter .name = 'Jonathan Harker')),
  places_visited := (select City filter .name = 'London'),
};

update Person filter .name = 'Jonathan Harker'
  set {
    lover := assert_single((select detached Person filter .name = 'Mina Murray'))
};

insert Vampire {
  name := 'Count Dracula',
  age := 800,
  slaves := {
    (insert MinorVampire {
      name := 'Vampire Woman 1',
  }),
    (insert MinorVampire {
     name := 'Vampire Woman 2',
  }),
    (insert MinorVampire {
     name := 'Vampire Woman 3',
  }),
 },
   places_visited := (select Place filter .name in {'Romania', 'Castle Dracula'})
};

update Person filter .name = 'Jonathan Harker'
  set {
    strength := 5
};

with next_number := count(Crewman) + 1,
  insert Crewman {
  number := next_number
};
with next_number := count(Crewman) + 1,
  insert Crewman {
  number := next_number
};
with next_number := count(Crewman) + 1,
  insert Crewman {
  number := next_number
};
with next_number := count(Crewman) + 1,
  insert Crewman {
  number := next_number
};
with next_number := count(Crewman) + 1,
  insert Crewman {
  number := next_number
};

insert Sailor {
  name := 'The Captain',
  rank := 'Captain'
};

insert Sailor {
  name := 'Petrofsky',
  rank := 'FirstMate'
};

insert Sailor {
  name := 'The First Mate',
  rank := Rank.SecondMate
};

insert Sailor {
  name := 'The Cook',
  rank := Rank.Cook
};

insert Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};
```
