```
# Schema:

module default {
  abstract type Person {
    required property name -> str;
    multi link places_visited -> Place;
    link lover -> Person;
  }

  type PC extending Person {
    required property transport -> Transport;
  }

  scalar type HumanAge extending int16 {
    constraint max_value(120);
  }

  type NPC extending Person {
    property age -> HumanAge;
  }

  type Vampire extending Person {
    property age -> int16;
  }

  type MinorVampire extending Person {
    required link master -> Vampire;
  }
  
  abstract type Place {
    required property name -> str;
    property modern_name -> str;
    property important_places -> array<str>;
  }

  type City extending Place;

  type Country extending Place;

  scalar type Transport extending enum<Feet, Train, HorseDrawnCarriage>;

  scalar type SleepState extending enum <Asleep, Awake>;
  
  type Time { 
    required property clock -> str; 
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

insert NPC {
  name := 'Jonathan Harker',
  places_visited := City,
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

insert Vampire {
  name := 'Count Dracula',
  places_visited := (select Place filter .name = 'Romania'),
};

insert NPC {
  name := 'The innkeeper',
  age := 30,
};

insert City {
    name := 'London',
};

insert NPC {
  name := 'Mina Murray',
  lover := assert_single((select detached NPC filter .name = 'Jonathan Harker')),
  places_visited := (select City filter .name = 'London'),
};

insert MinorVampire {
  name := 'Vampire Woman 1',
  master := assert_single((select Vampire Filter .name = 'Count Dracula')),
};
```
