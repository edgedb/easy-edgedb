# Chapter 6 Questions and Answers

1. This select is incomplete. How would you complete it so that it says "Pleased to meet you, I'm " and then the NPC's name?

You can do it with concatenation using `++`:

```
SELECT NPC {
  name,
  greeting := "Pleased to meet you, I'm " ++ .name
};
```

2. 
