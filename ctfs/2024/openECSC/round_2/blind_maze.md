# Blind maze

## Description

```
Welcome to the blind maze, you move without knowing the outcome and maybe you will reach the end, good luck.

A previous winner left us a strange file. Maybe it will help you.

Site: http://blindmaze.challs.open.ecsc2024.it
```

## Categories

- `misc`

## Provided Files

- `capture.pcap`

## Solve Process

This challenge was later found out to be created with a mistake in it.

All you had to do was to basically open the `.pcap` file in WireShark and analyze the content in the `HTML` packets.

One of the last packets in the capture contained the flag `openECSC{xxx}` in cleartext.

One could also use the Search Function of WireShark to search for the flag prefix.
