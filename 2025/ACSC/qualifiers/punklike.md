# Punklike

## Description

```
Hey Roguelikes are pretty popular, can you beat this one?
```

## Categories

- `pwn`

## Provided Files

- `punklike.zip`

## Solve Process

This was a cool challenge with demonstrated the functioning of a [ROP-Chain](https://en.wikipedia.org/wiki/Return-oriented_programming#Attacks) attack in a "playful" manner.

The terminal allowed you to fight against NPCs utilizing different "attacks" which peformed some other tasks under the hood.

```plain
"right" => cost=5 func=pop_rdi_func
"left" => cost=2 func=pop_rsi_func
"tape" => cost=2 func=add_func
"fuse" => cost=10 func=multi_func
"secret" => cost=20 func=flag_door_func
"knife" => cost=2 value=9
"pistol" => cost=5 value=16
"machine gun" => cost=10 value=64
"nuke" => cost=25 value=1024
```

Each attack also has a `cost` and the last four a `value` attached. The cost is subtracted from our `STAMINA`. The code also expects a target value to reach for sucessfully accessing `flag_door_func`.

```c
// our target
#define LEET 1337
// our max cost
#define STAMINA 100
// --- snip ---
void flag_door_func(int key) {
  fputs(CLEAR, stdout);

  if (key == LEET)
    fprintf(stdout, "Flag: %-28s \n\r", getenv("FLAG"));

  fputs(" " TOP_LEFT_CORNER, stdout);
  for (int i = 0; i < 36; ++i)
    fputs(HORIZONTAL_LINE, stdout);

  fputs(TOP_RIGHT_CORNER "\n\r", stdout);

  if (key != LEET)
    fprintf(stdout,
            " " VERTICAL_LINE " %12sWrong Key%13s " VERTICAL_LINE "\n\r", "",
            "");

  fprintf(stdout,
          " " VERTICAL_LINE " Press any key to continue!%8s " VERTICAL_LINE
          "\n\r",
          "");

  fputs(" " BOT_LEFT_CORNER, stdout);

  for (int i = 0; i < 36; ++i)
    fputs(HORIZONTAL_LINE, stdout);
  fputs(BOT_RIGHT_CORNER "\n\r", stdout);
  fputs(FOOTER, stdout);
  fflush(stdout);
  fgetc(stdin);
}
// --- snip ---
```

So we need to find a combination that uses `add_func` and `multi_func` correctly to add up the values using the last four items. Keeping in mind our stamina limit of `100` that we can't breach. There may be more than one possible combination, but below is the one I found and used.

```plain
TARGET => ((16 * 9) * 9) + 16 + 16 + 9 = 1337
COST => (pistol * 3) + (knife * 3) + (tape * 3) + (fuse * 2) + (left * 5) + (right * 1) = 62
```

We can the use this in our attack chain in the correct menu of the game.

```plain
right => 5
pistol => 5
left => 2
knife => 5
fuse => 10
left => 2
knife => 2
fuse => 10
left => 2
pistol => 5
tape => 2
left => 2
pistol => 5
tape => 2
left => 2
knife => 2
tape => 2
secret => 5
```

When executing this, the game prints out the flag, which is another solve.
