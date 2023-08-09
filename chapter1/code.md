```
# Schema:

module default {
  type City {
    required name: str;
    modern_name: str;
  }
  type NPC {
    required name: str;
    multi places_visited: City;
  }
}

# Data:

insert City { name := 'Munich' };

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
