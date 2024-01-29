```
# Schema:

module default {

  # Scalar types

  scalar type Class extending enum<Rogue, Mystic, Merchant>;

  scalar type Mode extending enum<Info, Debug>;

  scalar type PCNumber extending sequence;

  scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;

  scalar type SleepState extending enum <Asleep, Awake>;

  # Globals and definitions

  required global tester_mode: Mode {
    default := Mode.Info;
  }

  global time := assert_single((select Time));

  abstract annotation warning;

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
    conversational_name := .title ++ ' ' 
      ++ .name if exists .title else .name;
    pen_name := .name ++ ', ' 
      ++ array_join(.degrees, ', ') if exists .degrees else .name;
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
    annotation description := 'A place with 50 or more buildings. Anything else is an OtherPlace';
    population: int64;
  }

  type Country extending Place;

  type Crewman extending HasNumber, Person {
    overloaded name: str {
      default := 'Crewman ' ++ <str>.number;
    }
  }

  type Event {
    required description: str;
    required start_time: cal::local_datetime;
    required end_time: cal::local_datetime;
    required multi place: Place;
    required multi people: Person;
    location: tuple<float64, float64>;
    ns_suffix := '_N_' if .location.0 > 0.0 else '_S_';
    ew_suffix := '_E' if .location.1 > 0.0 else '_W';
    url := get_url() 
      ++ <str>(math::abs(.location.0)) ++ .ns_suffix 
      ++ <str>(math::abs(.location.1)) ++ .ew_suffix;
  }

  type MinorVampire extending Person {
    former_self: Person;
  }

  type NPC extending Person {
    overloaded age: int16 {
      constraint max_value(120);
    }
  }

  type OtherPlace extending Place {
    annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
    annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
  }

  type Party {
    name: str;
  }

  type PC extending Person {
    required class: Class;
    required created_at: datetime {
      default := datetime_of_statement();
    }
    required number: PCNumber {
      default := sequence_next(introspect PCNumber);
    }
    multi party: Party {
      on source delete delete target if orphan;
      on target delete allow;
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
    multi slaves: MinorVampire {
      on source delete delete target;
      property combined_strength := (Vampire.strength + .strength) / 2;
    }
    army_strength := sum(.slaves@combined_strength);
  }

  # Functions

  function fight(one: Person, two: Person) -> str
  using (
    one.name ++ ' wins!' if (one.strength ?? 0) > (two.strength ?? 0)
    else two.name ++ ' wins!'
  );

  function fight(people_names: array<str>, opponent: Person) -> str
    using (
      with
          people := (select Person filter contains(people_names, .name)),
      select
          array_join(people_names, ', ') ++ ' win!'
          if sum(people.strength) > (opponent.strength ?? 0)
          else opponent.name ++ ' wins!'
  );

  function get_url() -> str
    using (
      <str>'https://geohack.toolforge.org/geohack.php?params='
    );

  function visited(person: str, city: str) -> bool
    using (
      with person := (select Person filter .name = person),
      select city in person.places_visited.name
    );
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
#  rank := Rank.Cook
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

insert Event {
  description := "Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.",
  start_time := cal::to_local_datetime(1893, 9, 11, 18, 0, 0),
  end_time := cal::to_local_datetime(1893, 9, 11, 23, 0, 0),
  place := (select Place filter .name = 'Whitby'),
  people := (select Person filter .name ilike 
    {'%helsing%', '%westenra%', '%seward%'}),
  location := (54.4858, -0.6206),
};

with 
  ship_people := (select Ship.sailors union Ship.crew filter Ship .name = 'The Demeter'),
  dracula := (select Vampire filter .name = 'Count Dracula'),
insert Event {
  description := "On 11 July at dawn entered Bosphorus. Boarded by Turkish Customs officers. Backsheesh. All correct. Under way at 4 p.m.",
  start_time := cal::to_local_datetime(1893, 7, 11, 7, 0, 0),
  end_time := cal::to_local_datetime(1893, 7, 11, 16, 0, 0),
  place := (insert OtherPlace {name := 'Rumeli Feneri'}),
  people := ship_people union dracula,
  location := (41.2350, 29.1100)
};

update Person filter .name = 'Lucy Westenra'
  set {
  last_appearance := cal::to_local_date(1893, 9, 20)
};

with lucy := assert_single((select Person filter .name = 'Lucy Westenra'))
insert Vampire {
  name := 'Count Dracula',
  age := 800,
  strength := 20,
  places_visited := (select Place filter .name in {'Romania', 'Castle Dracula'}),
  slaves := {
    (insert MinorVampire { name := 'Vampire Woman 1'}),
    (insert MinorVampire { name := 'Vampire Woman 2'}),
    (insert MinorVampire { name := 'Vampire Woman 3'}),
    (insert MinorVampire {
     name := "Lucy",
     former_self := lucy,
     first_appearance := lucy.last_appearance,
     strength := lucy.strength + 5,
    }),
 }
};

update Person
  filter .name not in {'Jonathan Harker', 'Count Dracula', 'Renfield'}
  set {
    strength := <int16>round(random() * 5)
  };

update MinorVampire
  set {
    strength := <int16>round(random() * 5) + 5
  };

insert City {
  name := 'Exeter', 
  population := 40000
};

update City filter .name = 'Buda-Pesth' 
  set { important_places := [
    'Hospital of St. Joseph and Ste. Mary',
    'Buda-Pesth University'
    ] 
  };

update Crewman
  set { name := 'Crewman ' ++ <str>.number };
```
