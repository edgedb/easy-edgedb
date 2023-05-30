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
    property created_at -> datetime {
      default := datetime_current()
  }
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

  type Country extending Place {
    multi link regions -> Region;
  }

  type Region extending Place {
    multi link cities -> City;
    multi link other_places -> OtherPlace;
    multi link castles -> Castle;
  }

  abstract annotation warning;

  type OtherPlace extending Place {
    annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
    annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
  }

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

  type Visit {
    link ship -> Ship;
    link place -> Place;
    required property date -> cal::local_date;
    property clock -> str;
    property clock_time := <cal::local_time>.clock;
    property hour := .clock[0:2];
    property sleep_state := 'asleep' if <int16>.hour > 7 and <int16>.hour < 19 else 'awake';
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

  abstract type Currency {
    required link owner -> Person;

    required property major -> str;
    required property major_amount -> int64 {
        default := 0;
        constraint min_value(0);
    }

    property minor -> str;
    property minor_amount -> int64 {
        default := 0;
        constraint min_value(0);
    }
    property minor_conversion -> int64;

    property sub_minor -> str;
    property sub_minor_amount -> int64 {
        default := 0;
        constraint min_value(0);
    }
    property sub_minor_conversion -> int64;
  }

  type Pound extending Currency {
    overloaded required property major {
        default := 'pound'
    }
    overloaded required property minor {
        default := 'shilling'
    }
    overloaded required property minor_conversion {
        default := 20
    }
    overloaded property sub_minor {
        default := 'pence'
    }
    overloaded property sub_minor_conversion {
        default := 240
    }
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

for city_name in {'Munich', 'London'}
union (
  insert City {
    name := city_name
  }
);

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

for country_name in {'Hungary', 'Romania', 'France', 'Slovakia', 'Bulgaria'}
  union (
    insert Country {
      name := country_name
    }
  );

insert Castle {
    name := 'Castle Dracula',
    doors := [6, 19, 10],
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

insert Ship {
  name := 'The Demeter',
  sailors := {
    (insert Sailor {
      name := 'The Captain',
      rank := Rank.Captain
    }),
    (insert Sailor {
      name := 'The First Mate',
      rank := Rank.FirstMate
    }),
    (insert Sailor {
      name := 'The Second Mate',
      rank := Rank.SecondMate
    }),
    (insert Sailor {
      name := 'The Cook',
      rank := Rank.Cook
    })
  },
  crew := (
    for n in {1, 2, 3, 4, 5}
    union (
      insert Crewman {
        number := n,
        first_appearance := cal::to_local_date(1893, 7, 6),
        last_appearance := cal::to_local_date(1893, 7, 16),
      }
    )
  )
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
  start_time := cal::to_local_datetime(1893, 9, 11, 18, 0, 0),
  end_time := cal::to_local_datetime(1893, 9, 11, 23, 0, 0),
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
     strength := 9,
  }),
    (insert MinorVampire {
     name := 'Vampire Woman 2',
     strength := 9,
  }),
    (insert MinorVampire {
     name := 'Vampire Woman 3',
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
  date := cal::to_local_datetime(1893, 10, 1, 4, 0, 0),
  author := assert_single((select Person filter .name = 'John Seward')),
  excerpt := 'Dr. Seward\'s Diary.\n 1 October, 4 a.m. -- Just as we were about to leave the house, an urgent message was brought to me from Renfield to know if I would see him at once..."You will, I trust, Dr. Seward, do me the justice to bear in mind, later on, that I did what I could to convince you to-night."',
};

insert BookExcerpt {
  date := cal::to_local_datetime(1893, 10, 1, 5, 0, 0),
  author := assert_single((select Person filter .name = 'Jonathan Harker')),
  excerpt := '1 October, 5 a.m. -- I went with the party to the search with an easy mind, for I think I never saw Mina so absolutely strong and well...I rest on the sofa, so as not to disturb her.',
};

select (
  update NPC filter .name = 'Renfield'
    set {
  last_appearance := <cal::local_date>'1893-10-03'
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

insert Pound {
 owner := (select Person filter .name = 'Count Dracula'),
   major_amount := 2500,
   minor_amount := 50,
   sub_minor_amount := 200
};

select (for character in {'Jonathan Harker', 'Mina Murray', 'The innkeeper', 'Emil Sinclair'}
  union (
    insert Pound {
      owner := assert_single((select Person filter .name = character)),
      major_amount := <int64>round(random() * 500),
      minor_amount := <int64>round(random() * 100),
      sub_minor_amount := <int64>round(random() * 500)
  })) {
  owner: {
    name
  },
  pounds := .major_amount,
  shillings := .minor_amount,
  pence := .sub_minor_amount,
  total_pounds :=
    round(<decimal>(.major_amount + (.minor_amount / .minor_conversion) + (.sub_minor_amount / .sub_minor_conversion)), 2)
};

insert Ship {
  name := 'Czarina Catherine',
  coffins := 1,
};

for city in {'Varna', 'Galatz'}
 union (
 insert City {
   name := city
});

insert OtherPlace {
  name := 'Bosphorus'
};

for visit in {
    ('The Demeter', 'Varna', '1893-07-06'),
    ('The Demeter', 'Bosphorus', '1893-07-11'),
    ('The Demeter', 'Whitby', '1893-08-08'),
    ('Czarina Catherine', 'London', '1893-10-05'),
    ('Czarina Catherine', 'Galatz', '1893-10-28')}
union (
 insert Visit {
   ship := (select Ship filter .name = visit.0),
   place := (select Place filter .name = visit.1),
   date := <cal::local_date>visit.2
 });

update Visit filter .place.name = 'Galatz'
  set {
    clock := '13:00:00'
};

insert Country {
 name := 'Germany',
 regions := {
   (insert Region {
     name := 'Prussia',
     cities := {
       (insert City {
         name := 'Berlin'
         }),
       (insert City {
         name := 'Königsberg'
         }),
    }
 }),
   (insert Region {
     name := 'Hesse',
     cities := {
       (insert City {
         name := 'Darmstadt'
         }),
       (insert City {
         name := 'Mainz'
         }),
 }
 }),
   (insert Region {
     name := 'Saxony',
     cities := {
       (insert City {
          name := 'Dresden'
          }),
       (insert City {
          name := 'Leipzig'
          }),
    }
   })
 }
};

update MinorVampire filter .name != 'Lucy'
set {
  last_appearance := <cal::local_date>'1893-11-05'
};
```
