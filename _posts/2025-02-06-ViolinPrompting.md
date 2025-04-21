---
title: LLM and Music Theory
date: 2025-02-06 00:00:00 +0000
tags: [ai, music]
description: Trying to get an LLM to teach me violin
---

I recently got back into playing the violin. Joined an orchestra and started rehearsing [**Symphonie fantastique**](https://en.wikipedia.org/wiki/Symphonie_fantastique).

It’s not the easiest piece to dive into after a long break. I found myself struggling with the higher notes - particularly the position transitions. Movement five the sequence leading up to measure 82 - utter chaos. 

My experience with higher positions has been limited. The orchestras I have played in thus far have been... casual. I was able to get away with playing in the first and third positions and very occasionally the fifth. Second and fourth positions, I barely touched. There were pieces in the ABRSM syllabus that called for the occasionally second and fourth positions, however I never felt as if I had a good grasp on them.

So, I wanted a shortcut. I looked for software that would guide me on the position transitions. The two main points the problem boils down to are:

1. Minimise string crossings in higher positions 
2. Minimise position shifts as much as possible

Surprisingly, I was not able to find much on this. The closest was a [blog](https://www.violinist.com/blog/pooispoois/201612/20903/) from 2016 describing an algorithm that a violinist had come up with, however it was hyper-parameterised for his own personal use. Of course it would be - players have different preferences, and I wanted a general solution that could be adapted to different players.

So I tried to pass this problem through an LLM. 

This was the starting point - this is a passage from the symphony, I would play each four note segment in a single position and then shift up by one for the next four: 

## Prompt 1
    
    ```jsx
    for the below sequence of notes played on the violin:
    ["Db6", "C6", "Bb5", "Db6", "D6", "C6", "B5", "D6", "Eb6", "D6", "C6", "Eb6", "E6", "D6", "C#6", "E6", "F6", "E6", "D6", "F6", "G6", "F6", "E6", "G6"]
    
    list the position each note should be played in, along with the finger used, that would make it easy for the violinist to play
    ```
    

## Output
    
    ```jsx
    Db6 - Position 5, 4th finger (E string)
    
    C6 - Position 5, 3rd finger (E string)
    
    Bb5 - Position 3, 3rd finger (E string)
    
    Db6 - Position 5, 4th finger (E string)
    
    D6 - Position 6, 4th finger (E string)
    
    C6 - Position 5, 3rd finger (E string)
    
    B5 - Position 4, 4th finger (E string)
    
    D6 - Position 6, 4th finger (E string)
    
    Eb6 - Position 7, 4th finger (E string)
    
    D6 - Position 6, 3rd finger (E string)
    
    C6 - Position 5, 3rd finger (E string)
    
    Eb6 - Position 7, 4th finger (E string)
    
    E6 - Position 8, 3rd finger (E string)
    
    D6 - Position 6, 3rd finger (E string)
    
    C#6 - Position 5, 4th finger (E string)
    
    E6 - Position 8, 3rd finger (E string)
    
    F6 - Position 8, 4th finger (E string)
    
    E6 - Position 8, 3rd finger (E string)
    
    D6 - Position 6, 3rd finger (E string)
    
    F6 - Position 8, 4th finger (E string)
    
    G6 - Position 9, 1st finger (E string)
    
    F6 - Position 8, 4th finger (E string)
    
    E6 - Position 8, 3rd finger (E string)
    
    G6 - Position 9, 1st finger (E string)
    ```
    

To its credit, it was able to recognise the goal of minimising shifting and minimal string crossings. However, the suggested fingerings contain notes that would not be played in the listed position and the position shifting frequency is far from ideal.

Seeing as the model did not seem to understand the position to notes mapping, I provided this in the prompt. Additionally, the problem statement was condensed/hyper-simplified. I did not consider:

- tempo - gaps in a sequence or between sequences that would allow for position shifting
- range of notes a single finger could play - half a semitone lower/higher - meaning each finger was mapped to a singular note for the sake of this problem
- extended fourth finger notes, and harmonics

Thus, it was framed as an optimisation problem with some constraints on what notes can be played in which positions. For testing I’m still using the same sequence.

## Prompt 2

    ```jsx
    You are a professional violinist, and you are given a list of notes in treble clef. You will need to come up with the best possible fingerings for a given sequence.
    
    The notes on the violin are as follows: G3, G#3/Ab3, A3, A#3/Bb3, B3, C4, C#4/Db4, D4, D#4/Eb4, E4, F4, F#4/Gb4, G4, G#4/Ab4, A4, A#4/Bb4, B4, C5, C#5/Db5, D5, D#5/Eb5, E5, F5, F#5/Gb5, G5, G#5/Ab5, A5, A#5/Bb5, B5, C6, C#6/Db6, D6, D#6/Eb6, E6, F6, F#6/Gb6, G6, G#6/Ab6, A6, A#6/Bb6, B6, C7, C#7/Db7, D7, D#7/Eb7, E7, F7, F#7/Gb7, G7, G#7/Ab7, A7, A#7/Bb7, B7.
    
    The default position is first position. You may also use 2nd, 3rd, 4th, 5th, 6th, and 7th position fingerings.
    
    First Position:
    G String: G3 (open), A3 (1st finger), B3 (2nd finger), C4 (3rd finger), D4 (4th finger)
    D String: D4 (open), E4 (1st finger), F#4 (2nd finger), G4 (3rd finger), A4 (4th finger)
    A String: A4 (open), B4 (1st finger), C#5 (2nd finger), D5 (3rd finger), E5 (4th finger)
    E String: E5 (open), F#5 (1st finger), G5 (2nd finger), A5 (3rd finger), B5 (4th finger)
    
    Second Position:
    G String: B3 (1st finger), C4 (2nd finger), D4 (3rd finger), E4 (4th finger)
    D String: F#4 (1st finger), G4 (2nd finger), A4 (3rd finger), B4 (4th finger)
    A String: C#5 (1st finger), D5 (2nd finger), E5 (3rd finger), F#5 (4th finger)
    E String: G5 (1st finger), A5 (2nd finger), B5 (3rd finger), C#6 (4th finger)
    
    Third Position:
    G String: C4 (1st finger), D4 (2nd finger), E4 (3rd finger), F4 (4th finger)
    D String: G4 (1st finger), A4 (2nd finger), B4 (3rd finger), C5 (4th finger)
    A String: D5 (1st finger), E5 (2nd finger), F#5 (3rd finger), G5 (4th finger)
    E String: A5 (1st finger), B5 (2nd finger), C#6 (3rd finger), D6 (4th finger)
    
    Fourth Position:
    G String: D4 (1st finger), E4 (2nd finger), F4 (3rd finger), G4 (4th finger)
    D String: A4 (1st finger), B4 (2nd finger), C5 (3rd finger), D5 (4th finger)
    A String: E5 (1st finger), F#5 (2nd finger), G5 (3rd finger), A5 (4th finger)
    E String: B5 (1st finger), C#6 (2nd finger), D6 (3rd finger), E6 (4th finger)
    
    Fifth Position:
    G String: E4 (1st finger), F4 (2nd finger), G4 (3rd finger), A4 (4th finger)
    D String: B4 (1st finger), C5 (2nd finger), D5 (3rd finger), E5 (4th finger)
    A String: F#5 (1st finger), G5 (2nd finger), A5 (3rd finger), B5 (4th finger)
    E String: C#6 (1st finger), D6 (2nd finger), E6 (3rd finger), F#6 (4th finger)
    
    Sixth Position:
    G String: F4 (1st finger), G4 (2nd finger), A4 (3rd finger), B4 (4th finger)
    D String: C5 (1st finger), D5 (2nd finger), E5 (3rd finger), F#5 (4th finger)
    A String: G5 (1st finger), A5 (2nd finger), B5 (3rd finger), C#6 (4th finger)
    E String: D6 (1st finger), E6 (2nd finger), F#6 (3rd finger), G6 (4th finger)
    
    Each finger can play the note on its default position, and also one semitone lower or higher.
    
    Position shifts should be worthwhile, it should benefit the following sequence. Otherwise, position shifting should be avoided. For example, you should not shift from 1st to 2nd position then back to 1st position.
    
    Minimise string crossings in high positions. For example, if the transition is A4 to B4, it is better to play A4 on the G string and B4 on the A string, rather than crossing the D string.
    
    If there are clear segments in the given sequence, you may prefer to play each segment in a single position.
    
    You will be given a list of notes. You must output the suggestions in the below JSON format:
    
    ```
    [
      {
        "note": string,
        "finger": "open" | "1st" | "2nd" | "3rd" | "4th",
        "position": "first" | "second" | "third" | "fourth" | "fifth" | "sixth",
        "string": "G" | "D" | "A" | "E"
      }
    ]
    ```
    
    Try on:
    ["Db6", "C6", "Bb5", "Db6", "D6", "C6", "B5", "D6", "Eb6", "D6", "C6", "Eb6", "E6", "D6", "C#6", "E6", "F6", "E6", "D6", "F6", "G6", "F6", "E6", "G6"]
    
    ```
    
## Output 2
    
    ```jsx
    [
      {
        "note": "Db6",
        "finger": "2nd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "C6",
        "finger": "2nd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "Bb5",
        "finger": "1st",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "Db6",
        "finger": "2nd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "D6",
        "finger": "3rd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "C6",
        "finger": "2nd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "B5",
        "finger": "1st",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "D6",
        "finger": "3rd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "Eb6",
        "finger": "3rd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "D6",
        "finger": "3rd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "C6",
        "finger": "2nd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "Eb6",
        "finger": "3rd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "E6",
        "finger": "4th",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "D6",
        "finger": "3rd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "C#6",
        "finger": "2nd",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "E6",
        "finger": "4th",
        "position": "fourth",
        "string": "E"
      },
      {
        "note": "F6",
        "finger": "3rd",
        "position": "sixth",
        "string": "E"
      },
      {
        "note": "E6",
        "finger": "2nd",
        "position": "sixth",
        "string": "E"
      },
      {
        "note": "D6",
        "finger": "1st",
        "position": "sixth",
        "string": "E"
      },
      {
        "note": "F6",
        "finger": "3rd",
        "position": "sixth",
        "string": "E"
      },
      {
        "note": "G6",
        "finger": "4th",
        "position": "sixth",
        "string": "E"
      },
      {
        "note": "F6",
        "finger": "3rd",
        "position": "sixth",
        "string": "E"
      },
      {
        "note": "E6",
        "finger": "2nd",
        "position": "sixth",
        "string": "E"
      },
      {
        "note": "G6",
        "finger": "4th",
        "position": "sixth",
        "string": "E"
      }
    ]
    ```
    

This is better than before (at least the notes can be played) but still quite far from ideal - it's not able to see patterns.

I also tried to get it to write a program. R1 approached it by transforming notes into their MIDI equivalent. In MIDI notation, each semitone is assigned a number, for each semitone increase the MIDI number increases by one. This is actually a nice way to deal with confusing flats and sharps in a passage - focusing on the notes and not so much the notation. Cognitively, C sharp and D flat may feel different, but mechanically they are not. Although one could argue there is purpose behind the composer's choice and flattening it into MIDI notation loses some of this intent. The program then penalises for each string crossing, and each position shift. The general idea is fine but parameterising will be tricky.

LLMs are said to be bad at interacting with the physical world (at the time of writing). However I still believe it could work with a better prompt... but back to practicing for now.