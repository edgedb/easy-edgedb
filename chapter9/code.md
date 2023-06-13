```
# Schema:

module default {
  abstract type Person {
    property name -> str {
      delegated constraint exclusive;
    }
    multi link places_visited -> Place;
    multi link lovers -> Person;
    property strength -> int16;
    property first_appearance -> cal::local_date;
    property last_appearance -> cal::local_date;
    property age -> int16;
  }

  type PC extending Person {
    required property class -> Class;
    property created_at -> datetime {
      default := datetime_current()
  }
  }

  type NPC extending Person {
    overloaded property age {
      constraint max_value(120)
  }
    overloaded multi link places_visited -> Place {
      default := (select City filter .name = 'London');
    }
  }

  type Vampire extending Person {
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

  scalar type Class extending enum<Rogue, Mystic, Merchant>;

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
  class := Class.Mystic,
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
  lovers := (select detached NPC filter .name = 'Jonathan Harker'),
  places_visited := (select City filter .name = 'London'),
};

update Person filter .name = 'Jonathan Harker'
  set {
    lovers := (select detached Person filter .name = 'Mina Murray')
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

insert Sailor {
  name := 'The Captain',
  rank := Rank.Captain
};

insert Sailor {
  name := 'Petrofsky',
  rank := Rank.FirstMate
};

insert Sailor {
  name := 'The First Mate',
  rank := Rank.SecondMate
};

insert Sailor {
  name := 'The Cook',
  rank := Rank.Cook
};

for n in {1, 2, 3, 4, 5}
  union (
  insert Crewman {
  number := n,
  first_appearance := cal::to_local_date(1893, 7, 6),
  last_appearance := cal::to_local_date(1893, 7, 16),
});

insert Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};

insert NPC {
  name := 'Lucy Westenra',
  places_visited := (select City filter .name = 'London')
};

for character_name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  union (
    insert NPC {
    name := character_name,
    lovers := (select Person filter .name = 'Lucy Westenra'),
});

update NPC filter .name = 'Lucy Westenra'
set {
  lovers := (
    select Person filter .name in {'John Seward', 'Quincey Morris', 'Arthur Holmwood'}
  )
};

update NPC filter .name = 'Lucy Westenra'
  set {
    lovers := (select detached NPC filter .name = 'Arthur Holmwood'),
};

update NPC filter .name in {'John Seward', 'Quincey Morris'}
  set {
    lovers := {} # 😢
};

insert NPC {
  name := 'Renfield',
  first_appearance := cal::to_local_date(1893, 5, 26),
  strength := 10,
};
```
