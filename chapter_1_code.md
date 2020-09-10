Schema:

```
module default {
    type Person {
        required property name -> str;
        property places_visited -> array<str>;
}
    type City {
        required property name -> str;
        property modern_name -> str;
}
}
```


Data:

```
INSERT Person {
    name := 'Jonathan Harker',
    places_visited := ["Bistritz", "Vienna", "Buda-Pesth"],
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
```
