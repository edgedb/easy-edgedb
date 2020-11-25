```
# Schema:
START MIGRATION TO {
  module default {
  
    type Person {
      required property name -> str;
      multi link places_visited -> City;
    }
    
    type City {
      required property name -> str;
      property modern_name -> str;
    }
  }
};

POPULATE MIGRATION;
COMMIT MIGRATION;


# Data:

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

INSERT Person {
  name := 'Jonathan Harker',
  places_visited := City,
};
```
