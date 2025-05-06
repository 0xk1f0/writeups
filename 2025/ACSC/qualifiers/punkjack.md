# Punkjack

## Description

```
99% of pwners quit before they can afford the flag
```

## Categories

- `pwn`

## Provided Files

- `Punkjack.zip`

## Solve Process

Would Rust have fixed this? Yeah probably, because this challenge is an example of a UaF ord [User After Free](https://owasp.org/www-community/vulnerabilities/Using_freed_memory).

This occurs when a program continues to use a memory location or pointer after it has already been freed once.

```c
// --- snip ---
struct user {
  char name[4];
  int coins;
};
// --- snip ---
const struct item items[] = {
    {.name = "delete data",
     .price = 0,
     .func = delete_user}, // DSGVO compliancy
    {.name = "sell kidney", .price = -25000, .func = sell_kidney},
    {.name = "buy kidney", .price = 50000, .func = buy_kidney},
    {.name = "buy flag", .price = LEET, .func = buy_flag}};
// --- snip ---
void delete_user() {
  bzero(user, sizeof(struct user));
  free(user);
}
// --- snip ---
int kidney_count = 2;
void sell_kidney() {
  kidney_count--;
  if (kidney_count <= 0) {
    error("you died");
  }
}
void buy_kidney() { kidney_count++; }
// --- snip ---
void play_blackjack() {
  if (user->coins <= 0)
    error("you are broke");
  int seed;
  getrandom(&seed, sizeof(int), 0);
  srand(seed);
  struct card *deck = generate_deck();
  int dealer_cards = 2;
  struct card *dealer = malloc(dealer_cards * sizeof(struct card));
  int player_cards = 2;
  struct card *player = malloc(player_cards * sizeof(struct card));
// --- snip ---
void start_game() {
    user = malloc(sizeof(struct user));
    bzero(user, sizeof(struct user));
    fputs(ENABLE_CURSOR, stdout);
    set_name();
    fputs(DISABLE_CURSOR, stdout);
    int selected = 0;
    forever {
      draw_game(selected);
      switch (UPPER(fgetc(stdin))) {
      case 'S':
        if (selected > 0)
          --selected;
        break;
      case 'D':
        if (selected < 2)
          ++selected;
        break;
      case '\r':
      case '\n':
        switch (selected) {
        case 0: // play
          play_blackjack();
          break;
        case 1: // buy
          shop();
          break;
        case 2: // quit
          return;
        }
        break;
      default:
        break;
      }
    }
}
// --- snip ---
```

In essence, we can see that the game allocates the `user` variable at the beginning of the game using `malloc`. This variable is then used throughout the program to keep track of the players money.

In the `items` struct we can spot that the user has the option to `buy_kidney()`, `sell_kidney()` and also to `delete_user()`. The latter is especially interesting because it simply calls `free` on the `user` variable. But the program itself does not exit or end the game.

At this point we don't have a correct user struct assigned to `user`, so the next time a function tries to access it, the only thing it can read is corrupted data which was never meant to be read because it was already overwritten or used elsewhere.

When trying this in practice, as soon as we try to play a game with the previous call to `delete_user()`, we get a coin score of `10720762` after finishing a game of blackjack. Now this is not nearly enough to get to the `26666674` that we need to buy the flag.

This is where the second aspect of this challenge's code comes into play. The human body has two kidneys, defined by `int kidney_count = 2;`. But wait a second, that integer is defined globally, so it doesn't reset when we `delete_user()` right?

So in theory we can just do the same sequence over and over again, utilizing `delete_user()`, `kidney_count`, `buy_kidney` and `sell_kidney` to our advantage.

```text
while (kidney_count * kidney_sell_value < flag_price) {
    go_to_shop();
    while (user_coins > kidney_price) {
        buy_kidney();
    }
    delete_user();
    exit_to_menu();
    play_game();
}

// finally
while user_coins < flag_price {
    sell_kidney();
}
buy_flag();
```

We just need to be cautious about the fact that reaching a `kidney_count` of zero will kill us and end the game, so we have to stay in the above pseudo-code `while` loop for long enough.

This can surely be coded out with `pwntools` or similar, but I took the time to do this by hand.

As soon as we gather enough coins to buy the flag, it prints correctly and we can mark this challenge as solved.
