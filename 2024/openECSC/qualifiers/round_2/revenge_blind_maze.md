# Revenge of the Blind maze

## Description

```
Welcome back to the blind maze, this time you'll have a harder time finding the flag, good luck.

Site: http://blindmazerevenge.challs.open.ecsc2024.it
```

## Categories

- `misc`

## Provided Files

- `capture.pcap`

## Solve Process

This being the evolved (or intended) version of the beforementioned [Blind maze](./blind_maze.md) challenge, we can analyze the `HTML` packets again.

They seem to use the provided site to solve a maze by going in the directions `up`, `down`, `left` and `right`.

We can parse these directions out of the pcap file into a text file.

```bash
cat full_stream.txt | grep Last | grep -v FAILED
```

Using this command we get:

```text
    <h4>Last Move: start</h4>
    <h4>Last Move: right</h4>
    <h4>Last Move: down</h4>
    <h4>Last Move: down</h4>
    <h4>Last Move: right</h4>
    <h4>Last Move: right</h4>
    <h4>Last Move: right</h4>
    <h4>Last Move: right</h4>
    <h4>Last Move: up</h4>
    <h4>Last Move: up</h4>
    <h4>Last Move: right</h4>
#  --- snip ---
```

Now, some people might try to automate this, but there is a catch to this. Because some of the moves are indicated as `FAILED`. This is why the `grep` call above filters these.

So a solver script would need to know when to retry a move. Since I was under time pressure at the beginning of the second round because of internet problems, I decided to manually solve the maze by hand instead. Which I did manage on the first try.

When submitting the last `right` move, the website prints the flag and we can consider it solved.
