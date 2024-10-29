---
description: Learn how to solve the challenges from OSUSEC's Week 5 - Fall 2024 meeting!
status: new
---

# 10/28/24 - RevPaperScissors

## Description

> rock paper scissors is a lot easier if ur opponent lets you read their mind before u
> start playing

!!! note
    Tonight's challenges are a introduction to reverse engineering using Ghidra. To follow 
    along or play with the challenge yourself, you can find the first executable that I
    decompiled [here](http://chal.ctf-league.osusec.org/rev/revpaperscissors/chal1) and 
    the second one [here](http://chal.ctf-league.osusec.org/rev/revpaperscissors/chal2).

## Challenge 1

First off, let's use Ghidra to decompile the executable we were given:

```c hl_lines="27 28 29 30 31 32"
int main(void) {
    long lVar1;
    int iVar2;
    char *cVar3;
    long in_FS_OFFSET;
    size_t len;
    char name [100];
    char flag [100];
    
    lVar1 = *(long *)(in_FS_OFFSET + 0x28);
    setbuf(stdin,(char *)0x0);
    setbuf(stdout,(char *)0x0);
    get_flag(flag,100);
    puts("Welcome to Rock-Paper-Scissors! You\'ll have to beat me 5 times in a row to win!: ");
    printf("Enter your name: ");
    cVar3 = fgets(name,100,stdin);
    if (cVar3 == (char *)0x0) {
        puts("Error reading name!");
        iVar2 = 1;
    }
    else {
        len = strlen(name);
        if ((len != 0) && (name[len - 1] == '\n')) {
        name[len - 1] = '\0';
        }
        printf("Hello, %s! Let\'s start the game.\n",name);
        throw_hands("rock");
        throw_hands("scissors");
        throw_hands("paper");
        throw_hands("paper");
        throw_hands("scissors");
        printf("Wow, good job! Here\'s your flag: %s\n",flag);
        iVar2 = 0;
    }
    if (lVar1 != *(long *)(in_FS_OFFSET + 0x28)) {
                        /* WARNING: Subroutine does not return */
        __stack_chk_fail();
    }
    return iVar2;
}
```

Wow, this first challenge is quite easy! There's a bit of garbage in this view, but by 
just decompiling the executable and viewing the `#!c main` function, we can see the exact 
order that the computer will pick for the rock, paper, scissors challenges. To beat this
challenge, we just need to run the program and pick the options that beat each of the 
computer's moves! Let's see what the next challenge has in store...

## Challenge 2

Once again, let's start by decompiling the ELF file and looking at the `#!c main` function. 
I'll go ahead and clean up some of the variable names that Ghidra couldn't find symbols 
for while we're at it:

```c hl_lines="29 30 31 32 35 39"
int main(void) {
    long stack_check_magic;
    int return_status;
    char *result;
    size_t name_len;
    long in_FS_OFFSET;
    int i;
    size_t len;
    int moves [10];
    char name [100];
    char flag [100];
    
    stack_check_magic = *(long *)(in_FS_OFFSET + 0x28);
    setbuf(stdin, (char *)0x0);
    setbuf(stdout, (char *)0x0);
    get_flag(flag, 100);
    puts("Welcome to Rock-Paper-Scissors! You\'ll have to beat me 10 times in a row to win!: ");
    printf("Enter your name: ");
    result = fgets(name, 100, stdin);
    if (result == (char *)0x0) {
        puts("Error reading name!");
        return_status = 1;
    }
    else {
        name_len = strlen(name);
        if ((name_len != 0) && (name[name_len - 1] == '\n')) {
            name[name_len - 1] = '\0';
        }
        if (name_len < 0xb) {
            puts("That can\'t be right, enter a longer name.");
                            /* WARNING: Subroutine does not return */
            exit(0);
        }
        printf("Hello, %s! Let\'s start the game.\n", name);
        make_moves(name, moves);
        for (i = 0; i < 10; i = i + 1) {
            throw_hands(moves[i]);
        }
        printf("Wow, good job! Here\'s your flag: %s\n", flag);
        return_status = 0;
    }
    if (stack_check_magic != *(long *)(in_FS_OFFSET + 0x28)) {
                        /* WARNING: Subroutine does not return */
        __stack_chk_fail();
    }
    return return_status;
}
```

Interesting. This challenge is a bit harder, but there's a few things we can tell right 
away. In order they appear in the code, the name we enter will need to be at least 11 
(`#!c 0xb`) characters long. Second, we need to take a closer look at `#!c make_moves()`
to determine how the computer picks rock, paper, or scissors. Finally, we still just need
to win the game in order to get the flag.

Knowing that, let's go take a look at `#!c make_moves()`:

```c
void make_moves(char *in, int *out) {
    int delta;
    int i;
    
    delta = 0;
    for (i = 0; i < 10; i = i + 1) {
        in[i] = in[i] + (char)delta;
        out[i] = (int)(in[i] % '\x04');
        delta = (in[i] + 0x1ca3) % 0x15;
    }
    return;
}
```

Wow! There isn't any filler in this function, but it's still tough to figure out exactly 
what's going on. Essentially, though, this function just performs a bunch of math to 
scramble the character array `#!c in` and produce an array of numbers 0-3. Remembering 
that this function was called via `#!c make_moves(name, moves)`, we can see that the
name we input is directly used to create the computer's moves. 

Since the computer's moves depend on the name we pick, it's time to settle on one. For 
simplicity, let's just go with `#!c "aaaaaaaaaaa"`. `#!c make_moves()` will treat each
character as the ASCII number used to represent it, which for `#!c 'a'` is 97. A few other 
conversions that we'll find helpful are:

|   Source   | Base 10 |
|------------|---------|
|`#!c '\x04'`| 4       |
|`#!c 0x1ca3`| 7331    |
|`#!c 0x15`  | 21      |
|`#!c 'a'`   | 97      |

With our selected name and these conversions in mind, we can rewrite the move-picking 
algorithm as something like:

```py
delta = 0

for i in range(10):
    in_sub_i = 97 + delta
    delta = (in_sub_i + 7331) % 21
    print(in_sub_i % 4)
```

Still just a bunch of arbitrary math. However, we can run this to get the computer's move 
list: `#!c [1, 0, 2, 0, 3, 1, 3, 1, 0, 2]`. Now, this is strange. There are only 3 valid
moves in Rock, Paper, Scissors, but there are 4 options (`#!c 0` - `#!c 3`) that can be 
returned here. Let's take a closer look at `#!c throw_hands()` to see how the computer 
uses this to actually make moves. Or more precisely, to figure out what inputs we need in
order to beat the computer and get the flag. Once again, I'll go ahead and clean up 
Ghidra's output a bit for us to take a look at:

```c hl_lines="15 16"
int throw_hands(int choice) {
    long in_FS_OFFSET;
    int choice_local;
    int u;
    long local_10;
    
    local_10 = *(long *)(in_FS_OFFSET + 0x28);
    puts("Your move: (0: Rock, 1: Paper, 2: Scissors)");
    scanf("%d", &u);
    if ((u < 0) || (2 < u)) {
        puts("Invalid move! No cheating >:(.");
                        /* WARNING: Subroutine does not return */
        exit(0);
    }
    if ((u != 0 || choice != 1) && (u != 1 || choice != 2) && (u != 2 || choice != 0) && (choice != u)) {
        puts("You won this round!");
        if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                        /* WARNING: Subroutine does not return */
            __stack_chk_fail();
        }
        return 1;
    }
    puts("Womp womp, I win!");
                        /* WARNING: Subroutine does not return */
    exit(0);
}
```

Perfect! Now we know what comparisons the computer is making to decide if we win, but that
logic is still pretty confusing. Let's make this more readable:

```py
invalid_choice_pair = (
    (choice == 1 and u == 0) or
    (choice == 2 and u == 1) or
    (choice == 0 and u == 2)
)
same_choice = (choice == u)

if not invalid_u_choice_pair and not same_choice:
    print("You won this round!")
```

Cool! We can now see that the computer choosing `#!c 3` is basically just a free win for us!
For the others, we can see which option would be an invalid choice as well. With this in
mind, for the name `#!c "aaaaaaaaaaa"` we should be able to win with the choices 
`#!c [2, 1, 0, 1, x, 2, x, 2, 1, 0]`. After plugging those into the game, choosing any 
value for `#!c x`, we get the flag! 

Alternatively, if we didn't want to worry about the answer order, we could do the math to
pick a name that would result in all moves of `#!c 3`, allowing us to input any number for
each move and still win.