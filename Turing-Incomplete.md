# Turing Incomplete

A detailed writeup on Misc Challenge 'Turing Incomplete' from WolvCTF 2025.

By the end of WolvCTF 2025, 'Turing Incomplete' was worth 496 points (out of 500) and had 11 total solves.

## Challenge Description

![Turing Incomplete Challenge](/assets/turing_incomplete.png)

Two states and sixteen colors can do a lot of things.

`nc turingincomplete.kctf-453514-codelab.kctf.cloud 1337`

[dist.tar.gz](/assets/dist.tar.gz)

## Initial Analysis

Upon first inspection, the challenge name gave me a flashback to EECS 376 (Foundations of Computer Science). One large part of 376 involved Turing Machines -- an abstract model of computation.

After some reminsicing on what Turing Machines did again, I decided to look at the provided source codes `main.py` and `machine.py`.

### main.py

From analyzing `main.py`, I saw that the host took in a string input and created a set of "instructions" for a Machine. The machine is given the encoded instructions and its tape is modified to display [..., k, j, i, 0xa, 0xa, ...]. The goal of this machine is calculate `i + j + k` in decimal form on the tape. The rightmost 0xa value represent the most significant integer. After all possible additions for three 1-digit decimal numbers have been calculated, the flag is read!
<details>
<summary> main.py </summary>

```python
def main():
instructions = input()
encoded = []
for instruction in instructions.split():
    move, write, state = instruction 
    encoded.append(Instruction(move, int(write, base=16), int(state, base=16)))
for i in range(10):
    for j in range(10):
        for k in range(10):
            machine = Machine(encoded)
            tape = machine.tape
            tape[machine.head] = i
            tape[machine.head-1] = j
            tape[machine.head-2] = k
            tape[machine.head+1] = 0xa
            tape[machine.head+2] = 0xa
            end_location = machine.head + 1
            machine.run()
            assert (machine.tape[end_location] == i + j + k) or (machine.tape[end_location] + machine.tape[end_location + 1] * 10) == i + j + k
with open("flag.txt", "rb") as flag:
    print(flag.read())
```  
</details>

### machine.py

Next, I decided to analyze `machine.py`. Machine is basically a Turing Machine written in python. When running, the Machine reads the value at its head (defined at 0x100 initially). The Machine then looks up an instruction from `self.table` given its current state and the value it just read. Using this instruction, it writes a value to the tape, modifies the state, and moves the tape head (left/right) or halts the Machine program entirely. Note that we need 32 instructions (one instruction for each possible state and value 0-F).

<details>
<summary> machine.py </summary>

```python
class Instruction:
    def __init__(self, move: str, write: int, state: int):
        self.move = move
        self.write = write
        self.state = state

class Machine:
    def __init__(self, instructions):
        assert len(instructions) == 0x20
        self.table = [{},{}]
        self.head = 0x100
        self.tape = [0xf] * 0x200
        self.state = 0
        for i in range(0x20):
            self.table[i//0x10][i%0x10] = instructions[i]
    def run(self):
        while True:
            instruction = self.tape[self.head]
            instruction = self.table[self.state][instruction]
            self.tape[self.head] = instruction.write
            self.state = instruction.state
            if instruction.move == "H":
                break
            elif instruction.move == "R":
                self.head+=1
            elif instruction.move == "L":
                self.head-=1
```

</details>

### Analyzing Results

To get the flag, I just need to create 32 instructions that adds three numbers (0-9) in Turing Machine format.

## Figuring Out the Solution

I used a [Turing Machine Simulator](https://turingmachine.io/) that was given as a resource to understand how Turing Machines worked. The code provided is meant for use in this simulator :D

### Challenge 1: Moving 1 Number

Moving one number seems simple until you realize you only have two states.

The first issue was keeping track of whether I had already accounted for moving a `0` over. To resolve this, I used `B` to represent that the Machine should 'ignore' this value.

The second issue was figuring out the starting value `A`. Logically an `A` should become `0` on its first update.

From here, I used the two states to keep track of whether the program should *subtract* (`state 0`) or *add* (`state 1`) to the value that it is at.

<details>
  <summary> Moving 1 Number Over Turing Machine</summary>
  
  This Turing Machine moves the value (0-9) over to the `A` position in the tape. 

  ```
  input: '9A'
  blank: 'F'
  start state: sub
  table:
      # subtract
      sub:
          0: {write: B, R: add}
          1: {write: 0, R: add}
          2: {write: 1, R: add}
          3: {write: 2, R: add}
          4: {write: 3, R: add}
          5: {write: 4, R: add}
          6: {write: 5, R: add}
          7: {write: 6, R: add}
          8: {write: 7, R: add}
          9: {write: 8, R: add}
          B: {write: B, L: sub}
      # add
      add:
          0: {write: 1, L: sub}
          1: {write: 2, L: sub}
          2: {write: 3, L: sub}
          3: {write: 4, L: sub}
          4: {write: 5, L: sub}
          5: {write: 6, L: sub}
          6: {write: 7, L: sub}
          7: {write: 8, L: sub}
          8: {write: 9, L: sub}
          A: {write: 0, L: sub}
  ```
</details>

### Challenge 2: Moving 2 Numbers

First, I had to add a `B` instruction for the other state so that when the Machine reads `B`, it passes the state along. The Machine moves left for `state 0` and moves right for `state 1`.

This addition almost worked for adding two numbers (**without** carry). However, when adding `4 + 4`, the Machine resulted in `9`. So what happened?

When the Machine (in `state 0`) evaluates `0`, it adds `+1` to the sum. This is not a problem when adding one number because our code accounted for this (turning `A` -> `0`). However, when adding two numbers, the process of turning `0` to `B` occurs twice! This causes an off-by-one error. To account for this, I added an additional state so that `A` -> `C` -> `0`. 

<details>
  <summary> Moving 2 Numbers Over (Without Carry) Turing Machine</summary>

  This Turing Machine adds two values, `i` and `j`, over to the `A` position in the tape. Note that `0 <= i + j < 10`.

  ```
  input: '44A'
  blank: 'F'
  start state: start
  table:
      # Adjusts the tape position to be in the correct spot
      start:
          [0,1,2,3,4,5,6,7,8,9,A,B,C,D,E,F]: {R: sub}
      # subtract
      sub:
          0: {write: B, R: add}
          1: {write: 0, R: add}
          2: {write: 1, R: add}
          3: {write: 2, R: add}
          4: {write: 3, R: add}
          5: {write: 4, R: add}
          6: {write: 5, R: add}
          7: {write: 6, R: add}
          8: {write: 7, R: add}
          9: {write: 8, R: add}
          B: {write: B, L: sub}
      # add
      add:
          0: {write: 1, L: sub}
          1: {write: 2, L: sub}
          2: {write: 3, L: sub}
          3: {write: 4, L: sub}
          4: {write: 5, L: sub}
          5: {write: 6, L: sub}
          6: {write: 7, L: sub}
          7: {write: 8, L: sub}
          8: {write: 9, L: sub}
          A: {write: C, L: sub}
          B: {write: B, R: add} # new
          C: {write: 0, L: sub} # new
  ```

    
</details>

Now, the hard part: moving two numbers over **with** carry.

The main issue with solving this part was that the `A` with the most significant bit needs to be initialized to `0` while the least significant `A` needs to be initialized to a "delayed" value (in my case, `C`). 

To solve this, I used the first rightmost `F` to switch the state so that after the first pass/addition, the states would be initialized as such: `j i C 0`, with `i` and `j` representing some arbitrary `0-9` value.

<details>
<summary> Moving 2 Numbers Over (With Carry) Turing Machine</summary>

  This Turing Machine adds two values, `i` and `j`, over to the `AA` position in the tape.
  
  ```
  input: '11AA'
  blank: 'F'
  start state: start
  table:
      # Adjusts the tape position to be in the correct spot
      start:
          [0,1,2,3,4,5,6,7,8,9,A,B,C,D,E,F]: {R: sub}
      # subtract
      sub:
          0: {write: B, R: add}
          1: {write: 0, R: add}
          2: {write: 1, R: add}
          3: {write: 2, R: add}
          4: {write: 3, R: add}
          5: {write: 4, R: add}
          6: {write: 5, R: add}
          7: {write: 6, R: add}
          8: {write: 7, R: add}
          9: {write: 8, R: add}
          B: {write: B, L: sub}
          C: {write: C, L: sub} # new
      # add
      add:
          0: {write: 1, L: sub}
          1: {write: 2, L: sub}
          2: {write: 3, L: sub}
          3: {write: 4, L: sub}
          4: {write: 5, L: sub}
          5: {write: 6, L: sub}
          6: {write: 7, L: sub}
          7: {write: 8, L: sub}
          8: {write: 9, L: sub}
          9: {write: C, R: add} # new
          A: {write: C, R: add} # modified
          B: {write: B, R: add}
          C: {write: 0, L: sub}
          F: {write: F, L: add} # new
  ```

</details>

### Challenge 3: Moving 3 Numbers

Modifying the Turing Machine for 3 numbers was quite simple. I just needed to add some values to create additional delay for the 3rd `0 -> B` conversion. To do this, I used `D` and `E` to separate what the two `A`s should eventually evaluate to after the first pass.

<details>
  <summary> Moving 3 Numbers Over (With Carry) Turing Machine</summary>
  
  This Turing Machine adds 3 numbers (with carry) as wanted.

  ```
  input: '999AA'
  blank: 'F'
  start state: start
  table:
  # Adjusts the tape position to be in the correct spot
  start:
      [0,1,2,3,4,5,6,7,8,9,A,B,C,D,E,F]: {R: start1}
  start1:
      [0,1,2,3,4,5,6,7,8,9,A,B,C,D,E,F]: {R: sub}
  # subtract
  sub:
      0: {write: B, R: add}
      1: {write: 0, R: add}
      2: {write: 1, R: add}
      3: {write: 2, R: add}
      4: {write: 3, R: add}
      5: {write: 4, R: add}
      6: {write: 5, R: add}
      7: {write: 6, R: add}
      8: {write: 7, R: add}
      9: {write: 8, R: add}
      B: {write: B, L: sub}
      C: {write: 0, L: sub} # modified
      D: {write: C, L: sub} # new
  # add
  add:
      0: {write: 1, L: sub}
      1: {write: 2, L: sub}
      2: {write: 3, L: sub}
      3: {write: 4, L: sub}
      4: {write: 5, L: sub}
      5: {write: 6, L: sub}
      6: {write: 7, L: sub}
      7: {write: 8, L: sub}
      8: {write: 9, L: sub}
      9: {write: C, R: add}
      A: {write: D, R: add} # modified
      B: {write: B, R: add}
      C: {write: E, L: sub} # modified
      D: {write: 0, L: sub}
      E: {write: 0, L: sub} # new
      F: {write: F, L: add}
  ```
</details>

## Final Solution

From all of this work, I used the final instruction set to create my final input for the challenge:

```RB1 R01 R11 R21 R31 R41 R51 R61 R71 R81 RE1 LB0 L00 LC0 H00 H00 L10 L20 L30 L40 L50 L60 L70 L80 L90 RC1 RD1 RB1 LE0 L00 L00 H00```

I've copied the same instructions into the Turing Machine simulator website [here](turingmachine.io/?import-gist=3b269e9d65d942c9b678981722db37d1).

And at around 4am, I got the flag :D

<details>
  <summary> Flag </summary>

  wctf{jU5t_a_b1T_0f_s7At3}

</details>
