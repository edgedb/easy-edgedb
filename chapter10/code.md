``` 
# Schema:

module default {

  # Globals

  global time := assert_single((select Time));

  # Scalar types

  scalar type Class extending enum<Rogue, Mystic, Merchant>;

  scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;

  scalar type SleepState extending enum <Asleep, Awake>;

  # Abstract object types

  abstract type HasNumber {
    required number: int16;
  }

  abstract type Person {
    required name: str {
      delegated constraint exclusive;
    }
    multi places_visited: Place;
    multi lovers: Person;
    is_single := not exists .lovers;
    strength: int16;
    first_appearance: cal::local_date;
    last_appearance: cal::local_date;
    age: int16;
    title: str;
    degrees: array<str>;
    conversational_name := .title ++ ' ' ++ .name if exists .title else .name;
    pen_name := .name ++ ', ' ++ array_join(.degrees, ', ') if exists .degrees 
       else .name;
  }

  abstract type Place {
    required name: str {
      delegated constraint exclusive;
    }
    modern_name: str;
    important_places: array<str>;
  }

  # Object types

  type Castle extending Place {
    doors: array<int16>;
  }

  type City extending Place {
    population: int64;
  }

  type Country extending Place;

  type Crewman extending HasNumber, Person {
    overloaded name: str {
      default := 'Crewman ' ++ <str>.number;
    }
  }

  type MinorVampire extending Person;

  type NPC extending Person {
    overloaded age: int16 {
      constraint max_value(120);
    }
  }

  type OtherPlace extending Place;

  type PC extending Person {
    required class: Class;
    required created_at: datetime {
      default := datetime_of_statement()
    }
  }

  type Sailor extending Person {
    rank: Rank;
  }

  type Ship {
    required name: str;
    multi sailors: Sailor;
    multi crew: Crewman;
  }

  type Time { 
    required clock: str; 
    clock_time := <cal::local_time>.clock; 
    hour := .clock[0:2]; 
    vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
      else SleepState.Awake;
  }

  type Vampire extending Person {
    multi slaves: MinorVampire;
  }
}

# Data:

insert Time { clock := '09:00:00' };

insert City { name := 'Munich' };

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
  class := Class.Mystic,
};

insert PC {
 name := 'Max Demian',
 class := Class.Mystic
};

insert Country { name := 'Hungary' };

insert Country { name := 'Romania' };

insert Country { name := 'France' };

insert Country { name := 'Slovakia' };

insert Castle {
    name := 'Castle Dracula',
    doors := [6, 19, 10],
};

insert City { name := 'London' };

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

update NPC filter .name = 'John Seward'
set { 
  title := 'Dr.',
  degrees := ['M.D.']
};

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
  population := 14400,
  important_places := ['Whitby Abbey']
};

for data in {('Buda-Pesth', 402706), ('London', 3500000), ('Munich', 230023), ('Bistritz', 9100)}
  union (
    update City filter .name = data.0
    set {
    population := data.1
});

insert NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := ['M.D.', 'Ph. D. Lit.', 'etc.']
};

insert City {
  name := 'Munich',
  population := 261023,
} unless conflict on .name
else (
  update City
  set {
    population := 261023,
  }
);
```
