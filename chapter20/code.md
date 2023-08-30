```
# Schema:

module default {

  # Scalar types

  scalar type Class extending enum<Rogue, Mystic, Merchant>;

  scalar type LotteryTicket extending enum <Nothing, WallChicken, ChainWhip, Crucifix, Garlic>;

  scalar type Mode extending enum<Info, Debug>;

  scalar type Money extending int64 {
    constraint min_value(0);
  }

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

  abstract type HasMoney {
    required pounds: Money {
      default := 0;
    }
    required shillings: Money {
      default := 0;
    }
    required pence: Money {
      default := 0;
    }    
    required cents: Money {
      default := 0;
    }
    required dollars: Money {
      default := 0;
    }
    property total_pence := .pounds * 240 + .shillings * 20 + .pence;
    property total_cents := .dollars * 100 + .cents;
    property approx_wealth_in_pounds := <int64>(.total_pence / 240 + .total_cents / 800);
  }

  abstract type HasNameAndCoffins {
    required coffins: int16 {
      default := 0;
    }
    required name: str {
      delegated constraint exclusive;
      constraint max_len_value(30);
    }
  }

  abstract type HasNumber {
    required number: int16;
  }

  abstract type Person extending HasMoney {
    required name: str {
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
    degrees: array<str>;
    property conversational_name := .title ++ ' ' 
      ++ .name if exists .title else .name;
    property pen_name := .name ++ ', ' 
      ++ array_join(.degrees, ', ') if exists .degrees else .name;
  }

  abstract type Place extending HasNameAndCoffins {
    modern_name: str;
    multi important_places: Landmark;
  }

  # Object types

  type Account {
    required name: str;
    required address: str;
    required username: str;
    required credit_card: CreditCardInfo {
      on source delete delete target;
    }
    multi pcs: PC;

    trigger user_info_insert after delete for each do (
      insert MinimalUserInfo {
        username := __old__.name,
        pcs := __old__.pcs,
        passcode := <int16>(random() * 400) + 100,
      }
    );
  }

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
    index on (.name ++ ': ' ++ <str>.population) {
      annotation title := 'Lists city name and population for display in Long Library stage';
    } on (.name ++ ': ' ++ <str>.population);
  }

  type Country extending Place {
    multi regions: Region;
  }

  type CreditCardInfo {
    required name: str;
    required number: str;

    link card_holder := .<credit_card[is Account]
  }

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
    index on (.location);
    property ns_suffix := '_N_' if .location.0 > 0.0 else '_S_';
    property ew_suffix := '_E' if .location.1 > 0.0 else '_W';
    property url := get_url() 
      ++ <str>(math::abs(.location.0)) ++ .ns_suffix 
      ++ <str>(math::abs(.location.1)) ++ .ew_suffix;
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

  type MinimalUserInfo {
    username: str;
    multi pcs: PC;
    passcode: int16;
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
  }

  type OtherPlace extending Place {
    annotation description := 'A place with under 50 buildings - hamlets, small villages, etc.';
    annotation warning := 'Castles and castle towns do not count! Use the Castle type for that';
  }

  type Party {
    name: str;
    link members := .<party[is PC];
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
    overloaded required name: str {
      constraint max_len_value(30);
    }
    last_updated: datetime {
      rewrite insert, update using (datetime_of_statement());
    }
    bonus_item: LotteryTicket {
      rewrite insert, update using (get_ticket());
    }
  }

  type Region extending Place {
    multi cities: City;
    multi other_places: OtherPlace;
    multi castles: Castle;
  }

  type Sailor extending Person {
    rank: Rank;
  }

  type Ship extending HasNameAndCoffins {
    multi sailors: Sailor;
    multi crew: Crewman;
  }

  type ShipVisit {
    required ship: Ship;
    required place: Place;
    required date: cal::local_date;
    clock: str;
    property clock_time := <cal::local_time>.clock;
    property hour := .clock[0:2];
    property vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
      else SleepState.Awake;
  }

  type Time { 
    required clock: str; 
    property clock_time := <cal::local_time>.clock; 
    property hour := .clock[0:2]; 
    property vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
      else SleepState.Awake;
  } 

  type Vampire extending Person {
    multi slaves: MinorVampire {
      on source delete delete target;
      property combined_strength := (Vampire.strength + .strength) / 2;
    }
    property army_strength := sum(.slaves@combined_strength);
  }

  # Aliases

  alias AllNames := (
    distinct (HasNameAndCoffins.name union
    Place.modern_name union
    Landmark.name union 
    Person.name)
  );

  alias CrewmanInBulgaria := Crewman {
    name := 'Gospodin ' ++ .name,
    strength := .strength + <int16>1,
    original_name := .name,
  };

  alias GameInfo := (
    title := ( 
      en := "Dracula the Immortal",
      fr := "Dracula l'immortel",
      no := "Dracula den udødelige",
      ro := "Dracula, nemuritorul"
    ),
    country := "Norway",
    date_published := 2023,
    website := "www.draculatheimmortal.com"
  );


  # Functions

  function can_enter(person_name: str, place: str) -> optional str
    using (
    with
      vampire := assert_single((select Person filter .name = person_name)),
      enter_place := assert_single((select HasNameAndCoffins filter .name = place))
    select vampire.name ++ ' can enter.' if enter_place.coffins > 0 else        vampire.name ++ ' cannot enter.'
    );

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

  function get_ticket() -> LotteryTicket 
    using (
      with rnd := <int16>(random() * 10),
    select(
      LotteryTicket.Nothing if rnd <= 6 else
      LotteryTicket.WallChicken if rnd = 7 else
      LotteryTicket.ChainWhip if rnd = 8 else
      LotteryTicket.Crucifix if rnd = 9 else
      LotteryTicket.Garlic
      )
    );

  function get_url() -> str
    using (
      <str>'https://geohack.toolforge.org/geohack.php?params='
    );

  function visited(person: str, city: str) -> bool
    using (
      with 
        person := (select Person filter .name = person),
      select city in person.places_visited.name
    );
}

# Data:

insert Time { clock := '09:00:00' };

for city_name in {'Munich', 'London'}
union (
  insert City {
    name := city_name
  }
);

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
  class := Class.Mystic,
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
  )};

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

update Crewman
  set { name := 'Crewman ' ++ <str>.number };

update City filter .name = 'London'
  set { coffins := 21 };

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

update NPC filter .name = 'Renfield' set {
    last_appearance := <cal::local_date>'1893-10-03'
};

update HasNameAndCoffins filter .name = 'The Demeter' set { coffins := 10 };

update HasNameAndCoffins filter .name = 'Castle Dracula' set { coffins := 50 };

  update Person filter .name in { 'Arthur Holmwood', 'Count Dracula' }
  set {
    pounds := 3000 + <int64>(random() * 3000),
    shillings := 3000 + <int64>(random() * 3000),
    pence := 3000 + <int64>(random() * 3000)
  };

  update Person filter .name not in { 'Arthur Holmwood', 'Count Dracula' }
  set {
    pence := 100 + <int64>(random() * 100),
    shillings := 20 + <int64>(random() * 100),
    pounds := 10 + <int64>(random() * 100)
  };

  update Person filter .name = 'Quincey Morris'
  set { 
    dollars := 500 + <int64>(random() * 2000),
    cents := 500 + <int64>(random() * 2000)
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

insert OtherPlace { name := 'Bosphorus' };

for visit in {
    ('The Demeter', 'Varna', '1893-07-06'),
    ('The Demeter', 'Bosphorus', '1893-07-11'),
    ('The Demeter', 'Whitby', '1893-08-08'),
    ('Czarina Catherine', 'London', '1893-10-05'),
    ('Czarina Catherine', 'Galatz', '1893-10-28')}
union (
 insert ShipVisit {
   ship := (select Ship filter .name = visit.0),
   place := (select Place filter .name = visit.1),
   date := <cal::local_date>visit.2
 });

update ShipVisit filter .place.name = 'Galatz'
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

insert Account {
  name := 'Deborah Brown',
  address := '10 Main Street',
  username := 'deb_deb_999',
  credit_card := (insert CreditCardInfo 
    {  name := 'DEBORAH LAURA BROWN',
       number := '000-000-000' }
  ),
  pcs := (insert PC 
    {  name := 'LordOfSalty',
       class := Class.Rogue
    }
  )
};

delete Account filter .username = 'deb_deb_999';
```
