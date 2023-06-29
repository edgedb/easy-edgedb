```
# Schema:

module default {

  # Globals and definitions

  required global tester_mode: Mode {
    default := Mode.Info;
  }

  global time := assert_single((select Time));

  abstract annotation warning;

  # Scalar types

  scalar type Class extending enum<Rogue, Mystic, Merchant>;

  scalar type Mode extending enum<Info, Debug>;

  scalar type PCNumber extending sequence;

  scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;

  scalar type SleepState extending enum <Asleep, Awake>;

  # Abstract object types

  abstract type HasCoffins {
    required coffins: int16 {
      default := 0;
    }
  }

  abstract type HasNumber {
    required number: int16;
  }

  abstract type Person {
    name: str {
      delegated constraint exclusive;
    }
    multi places_visited: Place;
    multi lovers: Person;
    property is_single := not exists .lovers;
    strength: int16;
    first_appearance: cal::local_date;
    last_appearance: cal::local_date;
    age: int16;
    title: str;
    degrees: str;
    property conversational_name := .title ++ ' ' ++ .name if exists .title else .name;
    property pen_name := .name ++ ', ' ++ .degrees if exists .degrees else .name;
  }

  abstract type Place extending HasCoffins {
    required name: str {
      delegated constraint exclusive;
    }
    modern_name: str;
    multi important_places: Landmark;
  }

  # Object types

  type BookExcerpt {
    required date: cal::local_datetime;
    required excerpt: str;
    index on (.date);
    required author: Person;
  }

  type Castle extending Place {
    doors: array<int16>;
  }

  type City extending Place {
    annotation description := 'A place with 50 or more buildings. Anything else is an OtherPlace';
    population: int64;
    index on (.name ++ ': ' ++ <str>.population);
  }

  type Country extending Place;

  type Crewman extending HasNumber, Person;

  type Event {
    required description: str;
    required start_time: cal::local_datetime;
    required end_time: cal::local_datetime;
    required multi place: Place;
    required multi people: Person;
    multi excerpt: BookExcerpt; # Only this is new
    location: tuple<float64, float64>;
    east: bool;
    property url := get_url() ++ <str>.location.0 ++ '_N_' ++ <str>.location.1 ++ '_' ++ ('E' if .east else 'W');
  }

  type Landmark {
    required name: str;
    multi context: str;
  }

  type Lord extending Person {
    constraint expression on (contains(__subject__.name, 'Lord')) {
      errmessage := "All lords need \'Lord\' in their name";
    }
  }

  type MinorVampire extending Person {
    former_self: Person;
    single link master := assert_single(.<slaves[is Vampire]);
    property master_name := .master.name;
  };

  type NPC extending Person {
    overloaded age: int16 {
      constraint max_value(120);
    }
    overloaded multi places_visited: Place {
      default := (select City filter .name = 'London');
    }
  }

  type OtherPlace extending Place {
    annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
    annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
  }

  type PC extending Person {
    required class: Class;
    created_at: datetime {
      default := datetime_of_statement();
    }
    required number: PCNumber {
      default := sequence_next(introspect PCNumber);
    }
    overloaded required name: str {
      constraint max_len_value(30);
    }
  }

  type Sailor extending Person {
    rank: Rank;
  }

  type Ship extending HasCoffins {
    required name: str;
    multi sailors: Sailor;
    multi crew: Crewman;
  }

  type Time { 
    required clock: str; 
    property clock_time := <cal::local_time>.clock; 
    property hour := .clock[0:2]; 
    property vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
      else SleepState.Awake;
  } 

  type Vampire extending Person {
    multi slaves: MinorVampire;
  }

  # Functions

  function can_enter(person_name: str, place: HasCoffins) -> optional str
    using (
      with vampire := (select Person filter .name = person_name),
      has_coffins := place.coffins > 0,
        select vampire.name ++ ' can enter.' if has_coffins else vampire.name ++ ' cannot enter.'
    );

  function fight(one: Person, two: Person) -> str
    using (
      (one.name ?? 'Fighter 1') ++ ' wins!'
      if (one.strength ?? 0) > (two.strength ?? 0)
      else (two.name ?? 'Fighter 2') ++ ' wins!'
    );

  function fight(people_names: array<str>, opponent: Person) -> str
    using (
      with
          people := (select Person filter contains(people_names, .name)),
      select
          array_join(people_names, ', ') ++ ' win!'
          if sum(people.strength) > (opponent.strength ?? 0)
          else (opponent.name ?? 'Opponent') ++ ' wins!'
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

insert City {
  name := 'Munich',
};

insert City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest',
  important_places := {
    (insert Landmark {name := 'Hospital of St. Joseph and Ste. Mary'}),
    (insert Landmark {name := 'Buda-Pesth University'})
  }
};

insert City {
  name := 'Bistritz',
  modern_name := 'Bistrița',
  important_places := (insert Landmark { name := 'Golden Krone Hotel'}),
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
set { title := 'Dr.' };

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
  important_places := (insert Landmark { name := 'Whitby Abbey'})
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
  degrees := 'M.D., Ph. D. Lit., etc.'
};

insert Event {
  description := "Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.",
  start_time := cal::to_local_datetime(1893, 9, 11, 18, 0, 0),
  end_time := cal::to_local_datetime(1893, 9, 11, 23, 0, 0),
  place := (select Place filter .name = 'Whitby'),
  people := (select Person filter .name ilike {'%helsing%', '%westenra%', '%seward%'}),
  location := (54.4858, 0.6206),
  east := false
};

update Person
  filter .name not in {'Jonathan Harker', 'Count Dracula', 'Renfield'}
  set {
    strength := <int16>round(random() * 5)
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
    (insert MinorVampire {
     name := 'Lucy',
     former_self := lucy,
     first_appearance := lucy.last_appearance,
     strength := lucy.strength + 5,
    }),
 },
 places_visited := (select Place filter .name in {'Romania', 'Castle Dracula'})
};

insert City {
  name := 'Exeter', 
  population := 40000
};

update Crewman
  set {
    name := 'Crewman ' ++ <str>.number
};

update City filter .name = 'London'
  set {
    coffins := 21
 };

insert BookExcerpt {
  date := cal::to_local_datetime(1893, 10, 1, 4, 0, 0),
  author := assert_single((select Person filter .name = 'John Seward')),
  excerpt := 'Dr. Seward\'s Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once..."You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night."',
};

insert BookExcerpt {
  date := cal::to_local_datetime(1893, 10, 1, 5, 0, 0),
  author := assert_single((select Person filter .name = 'Jonathan Harker')),
  excerpt := '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
};
```
