```
# Schema:

module default {
  abstract type Person {
    name: str {
      delegated constraint exclusive;
    }
    multi places_visited: Place;
    multi lovers: Person;
    strength: int16;
    first_appearance: cal::local_date;
    last_appearance: cal::local_date;
    age: int16;
    title: str;
    degrees: str;
    property conversational_name := .title ++ ' ' ++ .name if exists .title else .name;
    property pen_name := .name ++ ', ' ++ .degrees if exists .degrees else .name;
  }

  type PC extending Person {
    required class: Class;
    created_at: datetime {
      default := datetime_current()
  }
  }

  type NPC extending Person {
    overloaded age {
      constraint max_value(120)
  }
    overloaded multi places_visited: Place {
      default := (select City filter .name = 'London');
    }
  }

  type Vampire extending Person {
    multi slaves: MinorVampire;
  }

  type MinorVampire extending Person {
  }
  
  abstract type Place {
    required name: str {
      delegated constraint exclusive;
    }
    modern_name: str;
    important_places: array<str>;
  }

  type City extending Place {
    population: int64;
  }

  type Country extending Place;

  type OtherPlace extending Place;

  type Castle extending Place {
    doors: array<int16>;
  }

  scalar type Class extending enum<Rogue, Mystic, Merchant>;

  scalar type SleepState extending enum <Asleep, Awake>;
  
  type Time { 
    required clock: str; 
    property clock_time := <cal::local_time>.clock; 
    property hour := .clock[0:2]; 
    property sleep_state := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
      else SleepState.Awake;
  } 

  abstract type HasNumber {
    required number: int16;
  }
  
  type Crewman extending HasNumber, Person {
  }

  scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;

  type Sailor extending Person {
    rank: Rank;
  }

  type Ship {
    required name: str;
    multi sailors: Sailor;
    multi crew: Crewman;
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

insert PC {
 name := 'Max Demian',
 class := Class.Mystic
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
  lovers := (select detached NPC Filter .name = 'Jonathan Harker'),
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

 Ship {
  name := 'The Demeter',
  sailors := Sailor,
  crew := Crewman
};

 NPC {
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

insert City {
  name := 'Whitby',
  population := 14400
};

for data in {('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)}
  union (
    update City filter .name = data.0
    set {
    population := data.1
});

 NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := 'M.D., Ph. D. Lit., etc.'
};
```
