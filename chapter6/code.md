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
    multi link slaves -> MinorVampire;
  }

  type MinorVampire extending Person {
  }
  
  abstract type Place {
    required property name -> str;
    property modern_name -> str;
    property important_places -> array<str>;
  }

  type City extending Place;

  type Country extending Place;

  type OtherPlace extending Place;

  scalar type Transport extending enum<Feet, Train, HorseDrawnCarriage>;

  type Time {
    required property clock -> str;
    property clock_time := <cal::local_time>.clock;
    property hour := .clock[0:2];
    property sleep_state := 'asleep' if <int16>.hour > 7 and <int16>.hour < 19 else 'awake';
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

insert OtherPlace {
  name := 'Castle Dracula'
};

insert City {
    name := 'London',
};

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
      name := 'Woman 1',
  }),
    (insert MinorVampire {
     name := 'Woman 2',
  }),
    (insert MinorVampire {
     name := 'Woman 3',
  }),
 },
   places_visited := (select Place filter .name in {'Romania', 'Castle Dracula'})
};
```
