# Let's write a Brainfuck Interpreter

This is a series where we will slowly climb up to building a JIT for a brainfuck compiler. This is the first blog in the series covering the language and a naive implementation. We try to understand everything from first principles, will try to explain reason behind every step we take, so buckle up and let's get started!

## Understanding Brainfuck Language Operators and Spec

Brainfuck is a super simple language which takes the idea of a turing machine and implements it as a programming language. That means we have a tape, an array of cell where each cell contains a number, and we just move around on it operating on numbers.

```
      ----------------------------------------
Tape: |10||65||0||0||45||14||0||0||0||0||0||0|
      ----------------------------------------
                          ^
                          |
      Pointer -------------
```

Let's look at all the operators:

- `>` : move right to next cell on tape
- `<` : move left to previous cell on tape
- `+` : increment the value of current cell
- `-` : decrement the value of current cell
- `,` : take input from user and store it in current cell
- `.` : output value stored in current cell to user
- `[` : jump to matching `]`, if value is zero
- `]` : jump to matching `[`, if value is not zero

And that's it, this is the whole language, suprisingly simple right xD

There are a few properties we haven't discussed yet, they are more like implementation details, for e.g. how long should the tape be?
what to do if you cross the max size of tape? what should be initial value of cell? These things are outlined in compelte detail in this spec:
https://github.com/sunjay/brainfuck/blob/master/brainfuck.md

## Small Brainfuck Programs which we will use in tests of our interpreter

As we know about all operators let's try getting our hands dirty and write few small programs, this also would give us an additional benefit of having test
cases ready for our interpreter. We can build incrementally harder programs and use them for our interpreter, this way we also add a nice incremental debugging test suite.

Super simple program: +++ , it just adds 1 to first cell thrice, so our tape would look like:

```
       ----------....
 Tape: |3||0||0||
       ----------....
```

Next one: ++-, add 1 to first cell twice, and then subtract 1 from it.

```
       ----------....
 Tape: |2||0||0||
       ----------....
```

Let's use `<` and `>` operators now: ++>+<-, this program first adds 1 twice to first cell, then shift to second cell, adds 1 over there, comes back to first cell and does a subtract operation. Our tape is now:

```
       ----------....
 Tape: |1||1||0||
       ----------....
```

Moving to next operators `,` and `.`: ,+., this program takes input from user, stores it in first cell, adds one to it and outputs it. Let's say we pass 65 when prompted for input, our tape would look like:

```
       ----------....
 Tape: |66||0||0||
       ----------....
```

and our output would be: `B`, we print ascii representations of numbers stored in cell, that's also how we get `hello world` too later down the road xD

Side Tip: Can't remember what ascii code represents what character? Don't worry there's a man page for it, just run: `man ascii`!

Now comes the l👀ps: [++], this programmm: does nothing :P . `[` operator says jump to corresponding `]` when current cell is zero, and by default all cell
values are zero, so our program jumped to last operator of program and exited.

Okay, let's do something serious this time: +++[-] . It's a simple program, we first increment first cell to value three, next we start a loop, this time it won't jump cause we have value 3 in there, in first iteration it will decrement value by one, i.e. to 2. we then have `]` which jumps to corresponding `[` if cell has non-zero value, so it goes back to `[` and second iteration starts where decrement happens and again jump happens, this continues till zero and at that point `]` sees a zero value at it's cell and exits the program.

After exec of `+++`:

```
       ----------....
 Tape: |3||0||0||
       ----------....
```

After first iteration, we exec 4th, 5th and 6th operator in program here:

`[` -> value is 3, don't jump, go to next instruction
`-` -> decrement value to 2
`]` -> value is 2, non-zero, jump to `[`

```
       ----------....
 Tape: |2||0||0||
       ----------....
```

After second iteration, we again execute 4th, 5th and 6th operator:

`[` -> value is 2, don't jump, go to next instruction
`-` -> decrement value to 1
`]` -> value is 1, non-zero, jump to `[`

```
       ----------....
 Tape: |1||0||0||
       ----------....
```

After third iteration, we again execute 4th, 5th and 6th operator:

`[` -> value is 1, don't jump, go to next instruction
`-` -> decrement value to 0
`]` -> value is 0, don't jump, go to next instruction i.e. program end

```
       ----------....
 Tape: |0||0||0||
       ----------....
```

This seems tedious to do it by hand right? No probs some humble person on internet made this for us:
https://arkark.github.io/brainfuck-online-simulator/

Visualization of brainfuck programs, really helpful for debugging!

Okay, now let's use loops, cell movement, etc together: >+++++++++[<++++++>-]<...>++++++++++. , at this point you should try to do some brain job and figure it out on your own.

Final state of tape:

```
       ----------....
 Tape: |54||10||0||
       ----------....
```

Having difficulties understanding? Try the online playground I linked above and iterate slowly on each operator, you should be able to figure it out.
(hopefully :P just kidding xD)

Okay, one last program and we are done, I just couldn't skip this program:

++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.

this is
is
is
is:

"hello world"

lessggoo we did it! We have reached hello world finally. It's a good exercise to think about execution here too, I think spec itself does a good job at trying
to explain it, so it's better to give that a try: https://github.com/sunjay/brainfuck/blob/master/brainfuck.md#hello-world-example

## C build system setup

Okay, enough brain jog, time to get hands dirty with actual interpreter implmentation, but before we start I want to clear up few things. First, we are going to implement this interpreter in C, and I am not that good at writing idiomatic C code, so if you find some weird way of doing things, that's just me being noob :P . And next thing is, we will be using meson as our build system. You can change the build system as per your convinience, I am not doing some rocket science with it, so shouldn't be hard to port from any to any.

I am going to dump the whole file at once, it's not much and easy to understand:

```
project('brenphuk', 'c',
  version : '0.1',
  default_options : ['warning_level=3', 'default_library=static'])

readline_dep = dependency('readline').as_system()

# source: https://github.com/tiernemi/meson-sample-project/blob/master/meson.build
# This adds the clang format file to the build directory
configure_file(input : '.clang-format',
               output : '.clang-format',
	       copy: true)

run_target('format',
  command : ['clang-format','-i','-style=file', ['../interpreter.c']])

run_command('clang-format','-i','-style=file', 'interpreter.c', check: true)

executable('brenphuk',
           'interpreter.c',
           install : true,
		   c_args: ['-Werror', '-Wall', '-Wextra', '-Wshadow', '-Wconversion',
					'-Wcast-align', '-Wunused', '-Wpointer-arith', '-Wold-style-cast',
					'-Wundef', '-Winit-self', '-Wredundant-decls', '-Wmissing-include-dirs',
					'-Wswitch-default', '-Wswitch-enum', '-Wfloat-equal', '-Wformat-security',
					'-Wpedantic',
					'-g'],
           dependencies : [readline_dep],
)
```

We set our project name as brenphuk, set `readline` as a dependency we need, we will be using that for REPL mode. Then we set up some formatting commands, code should look good always :) . Finally we define what our executable will be called and what files to use to make it, with some c-args, addded a bunch of them just for more strictness and help me not make mistakes, finally we pass `readline` dependency to executable to be built together.

## Implementation

For main impl, what we want to do is, take input from user, go over character by character and perform operation specified by operand mentioned on that index. Let's call this core function of ours as `exec`, which will accept an `engine` struct, this struct contains our actual tape and pointer:

```
typedef struct {
  char tape[TAPE_SIZE];
  int pointer;
} engine;
```

Along with engine, it will also accept a string, the program itself given as input from user.

```
int exec(engine *eng, char *prog) {
  size_t prog_len = strlen(prog);
  size_t i = 0;
  while (i < prog_len) {
    switch (prog[i]) {
    }
  }
}
```

Let's start adding impl of operations now, for `<`, we just want to increment engine->pointer, so it's simple as:
```
case '>':
  eng->pointer++;
  break;
```

and it's opposite `>`:
```
case '<':
  eng->pointer--;
  break;
```

Similarly for `+` and `-`, we want to increment value in tape on the index `pointer`:
```
case '+':
  eng->tape[eng->pointer]++;
  break;
case '-':
  eng->tape[eng->pointer]--;
  break;
```

For `,`, we want to take input and store it in currently pointed cell
```
case ',': {
  char ch;
  scanf("%c", &ch);
  eng->tape[eng->pointer] = ch;
  break;
}
```

and for `.`, we want to output currently pointed cell's value

```
case '.':
  printf("%c", eng->tape[eng->pointer]);
  break;
```

Now comes the slightly more interesting ones, loop constructs. For `[` and `]`, we want to jump to corresponding loop operand on certain condition.
On finding a zero on `[` operand, we have to find corresponding `]` operand, as the loops can be nested we have to keep a count of loop constructs we
have seen, so we maintain a counter, and increment it whenever we see a `[`, and decrement when we see a `]`. When we complete at the same number we started
our counter with, we have reached corresponding `]`. Impl for `[`, will look like:
```
case '[': {
  if (eng->tape[eng->pointer] == 0) {
    int brackets_depth = 0;
    while (i < prog_len) {
      if (prog[i] == '[') {
        brackets_depth++;
      } else if (prog[i] == ']') {
        brackets_depth--;
      }

      if (!brackets_depth) {
        break;
      }

      i++;
    }

    if (brackets_depth != 0) {
      ABORT("could not find matching closing square bracket");
    }
  }

  break;
}
```

here we use `bracket_depth` to keep track of the counter we discussed above, we start `brackets_depth` as 0, so at the end of transversing whole program if `brackets_depth` is not zero, we print out `brackets mismatch error`. Slightly, different impl for `]`:
```
case ']': {
  if (eng->tape[eng->pointer] != 0) {
    int brackets_depth = 0;
    while (i > 0) {
      if (prog[i] == '[') {
        brackets_depth--;
      } else if (prog[i] == ']') {
        brackets_depth++;
      }

      if (!brackets_depth) {
        break;
      }

      i--;
    }

    if (brackets_depth != 0) {
      ABORT("could not find matching opening square bracket");
    }
  }

  break;
}
```

And that's it, ofc we do need supporting code in main and things around repl, I am not covering them in blog cause they are mostly irrelevant and easy to do,
still just to whole thing together, you can check: https://github.com/feniljain/brenphuk/tree/attempt_1

This implementation also contains a benchmark suite, this will be helpful when comparing results of our different approaches in sections further.
Benchmark prorgams are present in https://github.com/feniljain/brenphuk/tree/attempt_1/programs , when we run above program with -O3, we get these execution times:

Factor: ~24-25s
Mandelbrot: ~69-70s

## Resources:

I have collected few brainfuck resources to help here: https://github.com/feniljain/knowledge-base/blob/main/programming-languages/brainfuck/README.md
