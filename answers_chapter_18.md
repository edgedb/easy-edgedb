# Chapter 18 Questions and Answers

#### 1. During the time of Dracula, the Goldmark was used in Germany. One Goldmark had 100 Pfennig. How would you make this type?

This is a lot easier than the `Pound` type, since Germany has 

```
type Goldmark extending Currency {
    overloaded required property major {
        default := 'Mark'
    }
    overloaded required property minor { 
        default := 'Pfennig'
    }
    overloaded required property minor_conversion {
        default := 100
    }
}
````

#### 2. Try adding two annotations to this type. One should be called `description` and mention that `One mark = 100 Pfennig`. The other should be called `note` and mention the types of coins there are.

Since `description` is already an option for annotations we wouldn't need to do anything for that. But for `note`, we would need to add this:

`abstract annotation note;`

After that, we add the annotations to Goldmark and it's done:

```
{
  type default::Goldmark extending Currency {
    annotation std::description := 'One Mark = 100 Pfennig';
    annotation default::note := 'Coin types: 1 Pfennig, 2 Pfennig, 5 Pfennig, 10 Pfennig, 20 Pfennig, 25 Pfennig';
    overloaded required property major {
        default := 'Mark';
    };
    overloaded required property minor {
        default := 'Pfennig';
    };
    overloaded required property minor_conversion {
        default := 100;
    };
};
```

3.

4.

5.
