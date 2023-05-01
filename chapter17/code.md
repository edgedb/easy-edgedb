```
# Schema:

module default {
  abstract type HasNameAndCoffins {
    required property coffins -> int16 {
      default := 0;
    }
    required property name -> str {
      delegated constraint exclusive;
      constraint max_len_value(30);
    }
  }

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
    property title -> str;
    property degrees -> str;
    property conversational_name := .title ++ ' ' ++ .name if exists .title else .name;
    property pen_name := .name ++ ', ' ++ .degrees if exists .degrees else .name;
  }

  type PC extending Person {
    required property transport -> Transport;
      overloaded required property name -> str {
        constraint max_len_value(30);
      }
  }

  type Lord extending Person {
  constraint expression on (contains(__subject__.name, 'Lord')) {
      errmessage := "All lords need \'Lord\' in their name";
    };
  };

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
    link former_self -> Person;
    single link master := assert_single(.<slaves[is Vampire]);
    property master_name := .master.name;
  };
  
  abstract type Place extending HasNameAndCoffins {
    property modern_name -> str;
    property important_places -> array<str>;
  }

  type City extending Place {
    annotation description := 'Anything with 50 or more buildings is a city - anything else is an OtherPlace';
    property population -> int64;
    index on (.name ++ ': ' ++ <str>.population);
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
    required property clock -> str;
    property clock_time := <cal::local_time>.clock;
    property hour := .clock[0:2];
    property sleep_state := 'asleep' if <int16>.hour > 7 and <int16>.hour < 19 else 'awake';
  }

  abstract type HasNumber {
    required property number -> int16;
  }
  
  type Crewman extending HasNumber, Person {
  }
  
  alias CrewmanInBulgaria := Crewman {
      name := 'Gospodin ' ++ .name,
      strength := .strength + <int16>1,
      original_name := .name,
    };

  scalar type Rank extending enum<Captain, FirstMate, SecondMate, Cook>;

  type Sailor extending Person {
    property rank -> Rank;
  }

  type Ship extending HasNameAndCoffins {
    multi link sailors -> Sailor;
    multi link crew -> Crewman;
  }

  type BookExcerpt {
    required property date -> cal::local_datetime;
    required property excerpt -> str;
    index on (.date);
    required link author -> Person
  }

  type Event {
    required property description -> str;
    required property start_time -> cal::local_datetime;
    required property end_time -> cal::local_datetime;
    required multi link place -> Place;
    required multi link people -> Person;
    multi link excerpt -> BookExcerpt;
    property exact_location -> tuple<float64, float64>;
    property east -> bool;
    property url := 'https://geohack.toolforge.org/geohack.php?params=' ++ <str>.exact_location.0 ++ '_N_' ++ <str>.exact_location.1 ++ '_' ++ ('E' if .east else 'W');
  }

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

  function visited(person: str, city: str) -> bool
    using (
      with person := (select Person filter .name = person),
      select city in person.places_visited.name
    );

  function can_enter(person_name: str, place: str) -> optional str
    using (
    with 
      vampire := assert_single((select Person filter .name = person_name)),
      enter_place := assert_single((select HasNameAndCoffins filter .name = place))
      select vampire.name ++ ' can enter.' if enter_place.coffins > 0 else vampire.name ++ ' cannot enter.'
      );   
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

insert Country {
  name := 'Bulgaria'
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
  first_appearance := cal::to_local_date(1887, 7, 6),
  last_appearance := cal::to_local_date(1887, 7, 16),
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
  first_appearance := cal::to_local_date(1887, 5, 26),
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

insert NPC {
  name := 'Abraham Van Helsing',
  title := 'Dr.',
  degrees := 'M.D., Ph. D. Lit., etc.'
};

insert Event {
  description := "Dr. Seward gives Lucy garlic flowers to help her sleep. She falls asleep and the others leave the room.",
  start_time := cal::to_local_datetime(1887, 9, 11, 18, 0, 0),
  end_time := cal::to_local_datetime(1887, 9, 11, 23, 0, 0),
  place := (select Place filter .name = 'Whitby'),
  people := (select Person filter .name ilike {'%helsing%', '%westenra%', '%seward%'}),
  exact_location := (54.4858, 0.6206),
  east := false
};

update Person
  filter not exists .strength
  set {
    strength := <int16>round(random() * 5)
};

update Person filter .name = 'Lucy Westenra'
  set {
  last_appearance := cal::to_local_date(1887, 9, 20)
};

with lucy := assert_single((select Person filter .name = 'Lucy Westenra'))
insert Vampire {
  name := 'Count Dracula',
  age := 800,
  strength := 20,
  slaves := {
    (insert MinorVampire {
     name := 'Woman 1',
     strength := 9,
  }),
    (insert MinorVampire {
     name := 'Woman 2',
     strength := 9,
  }),
    (insert MinorVampire {
     name := 'Woman 3',
     strength := 9,
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
  date := cal::to_local_datetime(1887, 10, 1, 4, 0, 0),
  author := assert_single((select Person filter .name = 'John Seward')),
  excerpt := 'Dr. Seward\'s Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once..."You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night."',
};

insert BookExcerpt {
  date := cal::to_local_datetime(1887, 10, 1, 5, 0, 0),
  author := assert_single((select Person filter .name = 'Jonathan Harker')),
  excerpt := '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
};

select (
  update NPC filter .name = 'Renfield'
    set {
  last_appearance := <cal::local_date>'1887-10-03'
})
  {
  name, 
  last_appearance
  };
  
create function fight_2(one: Person, two: Person) -> str
  using (
    select one.name ++ ' fights ' ++ two.name ++ '. ' ++ one.name ++ ' wins!' if one.strength > two.strength 
      else 
    one.name ++ ' fights ' ++ two.name ++ '. ' ++ two.name ++ ' wins!'
);
```
