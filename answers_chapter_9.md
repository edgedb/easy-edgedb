# Chapter 9 Questions and Answers

#### 1. Why doesn't this insert work and how can it be fixed?

It needs to be a set instead of an array, so change the brackets to `{}` instead:

```
FOR castle IN ['Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle']
  UNION(
    INSERT Castle {
      name := castle
});
```

#### 2. How would you do the same insert while displaying the castle's name at the same time?

It looks like this:

```
SELECT(
 FOR castle IN {'Windsor Castle', 'Neuschwanstein', 'Hohenzollern Castle'}
   UNION(
     INSERT Castle {
       name := castle
 })) { name };
 ```
 
#### 3. How would you change the `Vampire` type if all vampires needed a minimum strength of 10?

Since `strength` comes from `abstract type Person`, you would need to overload it and give it a constraint. The `Vampire` type would then look like this:

```
type Vampire extending Person {    
  multi link slaves -> MinorVampire;
  overloaded property strength {
    constraint min_value(10)
  }
}
```
