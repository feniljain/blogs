# Let's write a Brainfuck Interpreter: Optimizations

In last part, we wrote a naive implementation of brainfuck, which is pain-stakingly slow, let's try to optimize it, we will majorly discuss two major
optimizations in this blog. We will end up with really nice speedups at the end, so buckle up and let's go!

## First optimization

One of the best things about implementing brainfuck is it's implementation is simple and straightforward and hence one can find optimization opportunities realtively easily. We don't try to plot a flamegraph, cause we know most of the time is spent in `exec` function, that's where we execute all of our operations, so any optimizations done in that flow would give us direct noticeable speedups.

Let's look at implementations of our operands again, this is the core loop: https://github.com/feniljain/brenphuk/blob/6b00f84be79c00679dc28ba917b853ff2e18beea/interpreter.c#L66-L142 right now. There's not much to see, implementations for `>`, `<`, `+`, `-`, `.`, `,` are pretty simple and one liner even :P . So let's have a look at multi-liners i.e. loop implementations, here most hot path would definitely will be finding it's corresponding loop operand, let's say we have a program like:

```
[:::[::]:::[::[::]::]]
^1  ^2     ^3 ^4
```

(`:` here means any random operand), we have 4 loops in total, 1 being the parent loops of all, containing 2 and 3 as their immediate child loop and finally 4 inside 3. Let's say 1 repeats 5 times. In a single interation of loop1 we will be finding end of loop2 once, which would make this find operation happen 5 times. Now let's say loop3 executes 10 times, for loop4 we will execute find operation 10 * 5 = 50 times, this is wasted computation. We can do this computation once and store it for whole execution of program.

So do we make a kind of caching mechanism to store just for the inner loops? Technically we also have to jump for outer loops, so we do need jumping index for them too, but only once for most parent loop, and fewer times for depth one loops. What if we precompute all bracket locations? We as such do it while executing, maybe do it before execution starts, and then reference them to jump easily around. Let's give it a try and see our benchmark results.

We make an array as big as program size and fill it in with -1 values, at exact index of loop operands we will fill in it's corresponding loop operands index. So we create two arrays: `open_brackets_loc` and `close_brackets_loc`. Now just before entering the core loop of `exec` we call a new function called `fill_brackets_loc`, this takes in program and it's length and calculates all brackets location along with filling them in our arrays. Implementation is simple, we find a `[` and maintain a counter till we find corresponding `]`, same as what we did in last blogpost, but we will only do it once this time, at the very start. Code looks like this:

```C
void fill_brackets_loc(char *prog, int prog_len) {
  int i = 0, next_open_bracket_loc = -1;

  while (i < prog_len) {
    switch (prog[i]) {
    case '[': {
      int brackets_depth = 0;
      for (int j = i; j < prog_len; j++) {
        if (prog[j] == '[') { // found a new loop start operand
          if (next_open_bracket_loc == -1 && j != i) {
            next_open_bracket_loc = j;
          }
          brackets_depth++; // increase the counter
        } else if (prog[j] == ']') { // found a new loop end operand
          brackets_depth--; // decrease the counter
        }

        if (brackets_depth == 0) {
          open_brackets_loc[i] = j; // filling in our arrays
          close_brackets_loc[j] = i;
          break;
        }
      }

      if (brackets_depth != 0) {
        ABORT("brackets mismatch"); // oops didn't find corresponding loop operand
      }

      break;
    }
    default:
      break;
    }

    if (next_open_bracket_loc != -1) {
      i = next_open_bracket_loc;
      next_open_bracket_loc = -1;
    } else {
      i++;
    }
  }
}
```

We can even do one better by also storing each `[]` identified when transversing nested loops. But for now, this works :P

Now our `[` handler in `exec` looks like this:

```C
case '[':
  if (tape[pointer] == 0) {
    int idx = open_brackets_loc[i];
    if (idx == -1) {
      DBG_PRINTF("[: got bracket_loc as -1 for i: %d", i);
      ABORT("invalid state");
    }
    i = idx;
    continue;
  }

  break;
```

We directly look up the location of corresponding loop operanding and jump!

same for `]`:

```C
case ']': {
  if (tape[pointer] != 0) {
    int idx = close_brackets_loc[i];
    if (idx == -1) {
      DBG_PRINTF("]: got bracket_loc as -1 for i: %d", i);
      ABORT("invalid state");
    }
    i = idx;
    continue;
  }

 break;
}
```

Running our benchmarks now gives us:
Factor: ~7s
Mandelbrot: ~22s

That's some big gains from a simple observation! But wait we have more :)

## Second optimization

Before this occurs, I made a small change to our core `exec`, instead of using characters I am using enum variants for identifying each character, it's essentially the same thing as before just different representation. For the coversion between character operations and enum variants I wrote a simple `parse` function:

```C
enum Op_type {
  INVALID = 0,
  FWD,
  BWD,
  INCREMENT,
  DECREMENT,
  OUTPUT,
  INPUT,
  JMP_IF_ZERO,
  JMP_IF_NOT_ZERO,
};

void parse(char *prog, int prog_len) {
  int i = 0;

  while (i < prog_len) {
    enum Op_type op_type = INVALID;
    switch (prog[i]) {
    case '>':
      op_type = FWD;
    case '<':
      if (op_type == INVALID)
        op_type = BWD;
    case '+':
      if (op_type == INVALID)
        op_type = INCREMENT;
    case '-': {
      if (op_type == INVALID)
        op_type = DECREMENT;
      break;
    }
    case '.':
      op_type = OUTPUT;
      break;
    case ',':
      op_type = INPUT;
      break;
    case '[':
      op_type = JMP_IF_ZERO;
      break;
    case ']':
      op_type = JMP_IF_NOT_ZERO;
      break;
    default:
      break;
    }

    if (op_type == INVALID) {
      i++;
      continue; // this can happen when there are comments which are supposed to
                // be ignored
    }
      i++;
  }
}
```

Now an interesting optimization I have seen done in Bytecode Interpreters is combining instructions when they occur together way too often. This could happen with same or different instructions too. I learnt about this first time while completing (Crafting interpreters)[https://craftinginterpreters.com/] an amazing book by Bob Nystorm. So let's try to find if it is possible to combine any instructions in our case. We add an array with size of `[number of instructions][number of instructions]`. This is because we want to check how each instruction relates with other ones. At the end of parse function we add this code to make it record op_assoc:

```C
ops[++ops_len] = op;
if (ops_len > 0) {
    // This logic simply tries to unite op_assoc[1][5]
    // and op_assoc[5][1] into one single field
    int op_type_1 = (int)ops[ops_len].op_type;
    int op_type_2 = (int)ops[ops_len - 1].op_type;
    if (op_type_1 >= op_type_2) {
      op_assoc[op_type_2][op_type_1]++;
    } else {
      op_assoc[op_type_1][op_type_2]++;
    }
}
```

We try to unite results of form `op_assoc[i][j]` and `op_assoc[j][i]` into one field `op_assoc[i][j]`, cause we don't want to see associativity of `+` and `[` and `[` and `+` as separate results. With this done, let's try to get output of it for mandelbrot:

```
DEBUG: op_assoc[1][1]: 3506
DEBUG: op_assoc[1][3]: 438
DEBUG: op_assoc[1][4]: 337
DEBUG: op_assoc[1][5]: 3
DEBUG: op_assoc[1][7]: 498
DEBUG: op_assoc[1][8]: 568
DEBUG: op_assoc[2][2]: 3604
DEBUG: op_assoc[2][3]: 386
DEBUG: op_assoc[2][4]: 246
DEBUG: op_assoc[2][5]: 3
DEBUG: op_assoc[2][7]: 362
DEBUG: op_assoc[2][8]: 521
DEBUG: op_assoc[3][3]: 224
DEBUG: op_assoc[3][7]: 30
DEBUG: op_assoc[3][8]: 86
DEBUG: op_assoc[4][4]: 2
DEBUG: op_assoc[4][7]: 462
DEBUG: op_assoc[4][8]: 133
DEBUG: op_assoc[5][7]: 1
DEBUG: op_assoc[5][8]: 1
DEBUG: op_assoc[7][7]: 10
DEBUG: op_assoc[8][8]: 32
```

Highest oens are (2, 2), (1, 1), so repeating instructions, specifically `>`, `<`, these should be easy to club. Let's do just that, we will add a `repeat` field for each operation which will store how many times does the operation repeat. After this we can make exec function increment values by `repeat`'s value instead of just 1, after this change our `exec` function looks like this:

```C
int exec(char *prog, int prog_len) {
  DBG_PRINT(prog);
  int i = 0, val;

  parse(prog, prog_len);
  // print_op_assoc(); // This is for checking which all ops occur together
  fill_brackets_loc();

  while (i <= ops_len) {
    // start = clock();
    switch (ops[i].op_type) {
    case FWD:
      pointer += ops[i].repeat; // We increment by `repeat` now
      break;
    case BWD:
      pointer -= ops[i].repeat; // We increment by `repeat` now
      break;
    case INCREMENT:
      val = (int)tape[pointer];
      val += ops[i].repeat; // We increment by `repeat` now
      tape[pointer] = (char)val;
      break;
    case DECREMENT:
      val = (int)tape[pointer];
      val -= ops[i].repeat; // We increment by `repeat` now
      tape[pointer] = (char)val;
      break;
    case OUTPUT:
      printf("%c", tape[pointer]);
      break;
    case INPUT: {
      char ch = (char)getchar();
      tape[pointer] = ch;
      break;
    }
    case JMP_IF_ZERO:
      if (tape[pointer] == 0) {
        int idx = open_brackets_loc[i];
        if (idx == -1) {
          DBG_PRINTF("[: got bracket_loc as -1 for i: %d", i);
          ABORT("invalid state");
        }
        i = idx;
        continue;
      }

      break;
    case JMP_IF_NOT_ZERO: {
      if (tape[pointer] != 0) {
        int idx = close_brackets_loc[i];
        if (idx == -1) {
          DBG_PRINTF("]: got bracket_loc as -1 for i: %d", i);
          ABORT("invalid state");
        }
        i = idx;
        continue;
      }

      break;
    }
    case INVALID:
      ABORT("INVALID shouln't have leakded till here, there's a bug in parsing "
            "code");
    default:
      break;
    }

    i++;
  }

  return 0;
}
```

Simple and easy, let's benchmark this change:

Factor: ~2.16s
Mandelbrot: ~5.9s

And we get another round of massive speedups! Whole code is available at: https://github.com/feniljain/brenphuk/tree/attempt_3

This is where halt our efforts for optimizations, next we are going to learn about JITs from systems perspective, how do we leverage kernel APIs to achieve JITting.
