```
# Schema:

module default {
  type NPC {
    required property name -> str;
    multi link places_visited -> City;
  }
  
  type City {
    required property name -> str;
    property modern_name -> str;
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
  modern_name := 'Bistrița'
};

insert NPC {
  name := 'Jonathan Harker',
  places_visited := City,
};
```
