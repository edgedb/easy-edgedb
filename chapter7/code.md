```
# Schema:

module default {
  abstract type Person {
    required name: str {
      delegated constraint exclusive;
    }
    multi places_visited: Place;
    lover: Person;
    property is_single := not exists .lover;
    strength: int16;
  }

  type PC extending Person {
    required class: Class;
  }

  scalar type HumanAge extending int16 {
    constraint max_value(120);
  }

  type NPC extending Person {
    age: HumanAge;
  }

  type Vampire extending Person {
    age: int16;
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

  type City extending Place;

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

update Castle filter .name = 'Castle Dracula'
  set {
    doors := [6, 9, 10]
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
```
