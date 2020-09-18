Schema:

```
START MIGRATION TO {
    module default {
abstract type Person {
  required property name -> str;
  MULTI LINK places_visited -> City;
  LINK lover -> Person;
}
type Vampire extending Person {            
  property age -> int16;
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
abstract type Place {
  required property name -> str;
  property modern_name -> str;
  property important_places -> array<str>;
}
type City extending Place;
type Country extending Place;
scalar type Transport extending enum<'feet', 'train', 'horse-drawn carriage'>;
type Date {
  required property date -> str;
  property local_time := <cal::local_time>.date;
  property hour := .date[0:2];
  property awake := 'asleep' IF <int16>.hour > 7 AND <int16>.hour < 19 ELSE 'awake';
}
}

};

POPULATE MIGRATION;
COMMIT MIGRATION;
```

Data:

```
INSERT NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};

INSERT City {
  name := 'Munich',
};

INSERT City {
  name := 'Buda-Pesth',
  modern_name := 'Budapest'
};

INSERT City {
  name := 'Bistritz',
  modern_name := 'Bistrița'
};

INSERT City {
    name := 'London',
};

INSERT PC {
  name := 'Emil Sinclair',
  places_visited := City,
  transport := <Transport>'horse-drawn carriage',
};

insert Vampire {
  name := 'Count Dracula',
  places_visited := (SELECT Place FILTER .name = 'Romania'),
};

insert NPC {
    name := 'The innkeeper',
    age := 30
};

INSERT NPC {
  name := 'Mina Murray',
  lover := (SELECT DETACHED NPC Filter .name = 'Jonathan Harker' LIMIT 1),
  places_visited := (SELECT City FILTER .name = 'London'),
};
```
