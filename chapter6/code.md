```
# Schema:

module default {

  # Globals

  global time := assert_single((select Time));
  
  # Scalar types

  scalar type HumanAge extending int16 {
    constraint max_value(120);
  }
  scalar type Class extending enum<Rogue, Mystic, Merchant>;

  scalar type SleepState extending enum <Asleep, Awake>;

  # Abstract object types

  abstract type Person {
    required name: str;
    multi places_visited: Place;
    lover: Person;
    is_single := not exists .lover;
  }

  abstract type Place {
    required name: str;
    modern_name: str;
    important_places: array<str>;
  }

  # Object types

  type Castle extending Place;

  type City extending Place;

  type Country extending Place;

  type MinorVampire extending Person;

  type NPC extending Person {
    age: HumanAge;
  }

  type OtherPlace extending Place;

  type PC extending Person {
    required class: Class;
  }

  type Time { 
    required clock: str; 
    clock_time := <cal::local_time>.clock; 
    hour := .clock[0:2]; 
    vampires_are := SleepState.Asleep if <int16>.hour > 7 and <int16>.hour < 19
      else SleepState.Awake;
  }

  type Vampire extending Person {
    age: int16;
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

insert Country { name := 'Hungary' };

insert Country { name := 'Romania' };

insert Country { name := 'France' };

insert Country { name := 'Slovakia' };

insert Castle {
  name := 'Castle Dracula'
};

insert City { name := 'London' };

insert NPC {
  name := 'Jonathan Harker',
  places_visited := (
    select Place filter .name in {'Munich', 'Buda-Pesth', 'Bistritz', 'London', 'Romania', 'Castle Dracula'}
  )
};

insert NPC {
  name := 'The innkeeper',
  age := 30,
};

insert NPC {
  name := 'Mina Murray',
  lover := assert_single((select detached NPC filter .name = 'Jonathan Harker')),
  places_visited := (select City filter .name = 'London'),
};

update Person filter .name = 'Jonathan Harker'
  set {
    lover := assert_single(
      (select detached Person filter .name = 'Mina Murray')
    )
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
```
