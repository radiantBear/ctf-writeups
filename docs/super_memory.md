---
description: Learn how to solve the challenges from OSUSEC's third 2024-2025 school year meeting!
status: new
---

# 10/14/24 - Super Memory

## Description
> Welcome to super memory world! Each level has two flags - one for the normies, and one 
> for the leet... Can you beat them all?

!!! note
    Tonight's challenges both include C source files in the hundreds of LOC. To minimize
    clutter in this writeup, I will only include small snippets of the code. To follow 
    along with the full code, download challenge 1's source 
    [here](https://chal.ctf-league.osusec.org/pwn/super_memory/level1.c) and challenge 2's 
    source [here](https://chal.ctf-league.osusec.org/pwn/super_memory/level2.c). Line 
    numbers are set to match with the original source.

## Level 1
To start, we're given a C source file for the first game level. To play the game, we can
SSH into a CTF League server that only allows us to execute the compiled code. It's 
unlikely that we're supposed to break this limitation to, say, use GDB to jump to the
flag printing code. Instead, let's take a look at the source code. 

A few notes that will be helpful:

- `#!c read_flag()` is defined to fetch and print out the flag. Its implementation 
    details are straightforward & irrelevant; there's nothing strange happening in it.
- The game will continually loop, collecting input if any given and updating the state 
    accordingly. The display will also be updated with the current state, using ASCII art
    to provide a simple GUI. Details here (including `#!c main()`, `#!c buildLevel()`, and `#!c renderLevel()`) are similarly irrelevant.

On the other hand, there's some of interesting stuff happening in `#!c handleMovement()`. 
There's a fair bit of noise I'll filter out again, but let's take a closer look at the 
middle of the function:

```c linenums="101" hl_lines="10 11"
    switch (level[*ypos][*xpos]) {
        case ' ':
            break;
        case 'f':
            return "Does this feel like the end?\n"
                "Are you proud of yourself?\n"
                "Question: Should Magellan have been proud of himself had he turned around after leaving the harbor?\n"
                "Answer: No.";
            break;
        case '1':
            read_flag();
            break;
        default:
            if (input == 'w')
                (*ypos)++;
            else if (input == 'a')
                (*xpos)++;
            else if (input == 'd')
                (*xpos)--;
            else if (input == 's')
                (*ypos)--;
    }
```

This is the key! It looks like we need to find a way to maneuver our character over a 
number `1` somewhere on the map. 

Let's try playing the game to see where that might be... Here's the view from the end of 
the level:

```
++++++++++++++++++                       +++++++++++++++++                       
+                                        +                                       
+                                        +                                       
+                                        +                                       
+                                        +                                       
+                                        +                                       
+                                        +                                       
+                                        +                                       
+                                        +                                       
+                                        +                                       
+            _                           +                _                      
+           \_/                          +               \_/                     
+            |._                         +                |._                    
+            |'."-._.-""--.-"-.__.'/     +                |'."-._.-""--.-"-.__.'/
+            |  \                 /      +                |  \       .-.       / 
+            |   |                (      +                |   |     (@.@)      ( 
+            |   |                 )     +                |   |   '=.|m|.='     )
+            |  /                 /      +                |  /    .='`"``=.    / 
+            |.'                 (       +                |.'                 (  
+            |.-"-.__.-""-.__.-"-.)      +                |.-"-.__.-""-.__.-"-.) 
+            |                           +                |                      
+            |                           +                |                      
+            |                           +                |                      
+            |                           +                |                      
+            f                  p        +                1                      
---------------------------------------------------------------------------------
```

There's the flag! So close, and yet so far... our character is the `p` and the `+` symbols
represent a solid wall. There's no way we can make the jump required at the top of the 
screen, either. :frowning:

There must be some weakness that we're missing in the code. Let's go back and take a 
closer look at the beginning of the function:

```c linenums="83" hl_lines="5 6 7 8 9 13 14 17 18"
char *handleMovement(unsigned char *xpos, unsigned char *ypos, char level[LEVELHEIGHT][LEVELWIDTH], int input) {
    if (level[*ypos+1][*xpos] != ' ') {
        jumpCount = 0;
    }
    if (input == 'd')
        (*xpos) ++;
    if (input == 'a')
        (*xpos) --;
    if (input == 'w') {
        if (level[*ypos+1][*xpos] != ' ') {
            jumpCount = 3;
        }
        if (jumpCount > 0) {
            (*ypos) --;
        }
    }
    if (input == 's')
        (*ypos) ++;
```

This is interesting! It doesn't appear that there's any bounds checks occurring before 
players are moved, meaning the player could end up with negative coordinates. What does
that mean? Well, let's take a look at the places these coordinates are used in this 
function:

- Line 84: `#!c if (level[*ypos+1][*xpos] != ' ')`
- Line 92: `#!c if (level[*ypos+1][*xpos] != ' ')`
- Line 101: `#!c switch (level[*ypos][*xpos])`
- Line 123: `#!c if (level[*ypos+1][*xpos] == ' ')`

In all these cases, the position variables are used to index into the `#!c level` array.
These coordinates are used in a similar fashion in the rendering code to determine where 
to print the player's avatar:

```c linenums="74"
if (x == xpos && y == ypos)
    printf("p");
else
    printf("%c", level[y][x]);
```

In C, array indexing is essentially syntax sugar for pointer arithmetic and dereferencing,
so we could rewrite the oft-used `#!c level[ypos][xpos]` as 
`#!c *(level + ypos * LEVELWIDTH + xpos)`. This makes it easier to understand what happens
if we move off of the screen's bounds. 

For example, let $x = -1$ and $y = 15$. This will result in dereferencing the address


$\textrm{level} + 15 * 256 - 1$.

This is the same as 

$\textrm{level} + 14 * 256 + 255$.

In other words, everywhere that the player's coordinates are used, they will be 
indistinguishable from the coordinates $x = 255$ and $y = 14$ -- on the other end of the 
screen, one row higher. This means we have another way over to the skull flag where we
get the challenge's flag! 

The whole map looks like this:
```
                                                                                        +++++++++++++++++++++++++++++++++                                                                                                                                        
                                                                                                                                                                                                                                                                 
                                                                                   +++++                              +++++++++++++++++++++++++++                                                                                                                
                                                                                                                                                                                                                                                                 
                                                                                      +++++                                                    +++++++++++++++++++++++                                                                                           
                                                                                                                                                                                                                                                                 
                                                                                         ++++++                                                                                                                                                                  
                                                                                                                                            ++++++++++++++++++                                                                                                   
                                                                                      ++++++                                                                                                                                                                     
                                                                                                                                                               +++++                                                                                             
                                                                                            +                                                                                                                                                                    
                                                                                                                                                                                                                                                                 
                                                                                           + +                                                                                                                                                                   
                                                                                                                                    .')                                                                                                                          
              .                                                                             +                                      (_  )                                  ++++++++++++++++++                       +++++++++++++++++                             
                                                                                                                                                                          +                                        +                                             
                |                                                                          + +                                                                            +                                        +                                             
       .               /                                                                                                                                                  +                                        +                                             
        \       I                                                                           +                                _                                            +                                        +                                             
                    /                                                                                                    .+(`  )`.                                        +                                        +                                             
          \  ,g88R_                                                                       ++ ++                         :(   .    )                                       +                                        +                                             
            d888(` `).                   _                                                                         .--  `.  (    ) )                                      +                                        +                                             
   -  --==  888(     ).=--           .+(` ')`.                                            +++++                 .=(   )   ` _`  ) )                                       +                                        +                                             
  )         Y8P(       '`.          :(   .    )                                                                 (   .  )     (   )  ._                                    +                                        +                                             
          .+(`(      .    )    .--  `.  (    ) )                                        +++++++++              (   (   ))     `-'.:(`  )                                  +            _                           +                _                            
         ((    (..__.:'---'..=(   )   ` _`  ) )                                      ++                         `- __.'         :(      ))                                +           \_/                          +               \_/                           
  `.     `(       ) )       (   .  )     (   )  ._                                ++                                            `(    )  ))                               +            |._                         +                |._                          
    )      ` __.:'   )     (   (   ))     `-'.:(`  )                   ++++++++++                                                 ` __.:'                                 +            |'."-._.-""--.-"-.__.'/     +                |'."-._.-""--.-"-.__.'/      
  )  )  ( )       --'       `- __.'         :(      ))                                                                                                                    +            |  \                 /      +                |  \       .-.       /       
  .-'  (_.'          .')                    `(    )  ))          +++++++                                                                                                  +            |   |                (      +                |   |     (@.@)      (       
+                   (_  )                     ` __.:'                                                                                                                     +            |   |                 )     +                |   |   '=.|m|.='     )      
+                                                             +++                                                                                                         +            |  /                 /      +                |  /    .='`"``=.    /       
+                                                                                                                                                                         +            |.'                 (       +                |.'                 (        
+ p                                                             ++                                                                                                        +            |.-"-.__.-""-.__.-"-.)      +                |.-"-.__.-""-.__.-"-.)       
++++                                                                                                                                                                      +            |                           +                |                            
++++++                                    +                  +++                                                                                                          +            |                           +                |                            
++++++++                                + +   +  +  +                                                                                                                     +            |                           +                |                            
++++++++++                  ++         ++ +   +  +  +     +++                                                                                                             +            |                           +                |                            
++++++++++++        ++++    ++      ++              +                                                                                                                     +            f                           +                1                            
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

We can't immediately walk off the left boundary, but after climbing a bit, we can jump off
the upper left edge get warped around to the right side! Then, we simply need to walk over
to the `1` to complete the level and get the flag!

## Level 2
The next level is pretty cool looking, too! Let's start off by taking a look at the whole
map this time. Once again, we start in the bottom left corner:

```
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
+                                                                                                                                                                                                                                                              +
+                                                                                                                                                                                                                                                              +
+                                                                                                                                                                                                                                                              +
+                                                                                                                                                                                                                                                              +
+                                                                                                                                                                                                                                                              +
+                                            _                                                                                                                                                                                                                 +
+                                        .+(` ')`.                                                                                                                                                                                                             +
+                                       :(   .    )                                                                                                                                                                                                            +
+                                       `.  (    ) )                                                                                                                                                                                                           +
+                                         ` _`  ) )                                                                                                                                                                                                            +
+                                            (   )                                                                                                                                                                                                             +
+                                             `-'                                                                                                                                                                                                              +
+                                                                                    ~~.                                                                                                                                                                       +
+                                                                                   (~  )                                                                                                                                                                      +
+                                                                                  ( _.)_)                                          (`~                                                                                                                        +
+                                                                                                                                  (. _)                                                                                                                       +
+                                                                                                                                                                                                                                                              +
+               |                                                                                                                                                                                                                                              +
+      .               /                                                                                                                                                                           _                                                           +
+       \       |                                                                        .--._  _                            _                                                                 .+(` ')`. _  _                                                  +
+                   /                                                              .-..=(   . '` ')`.                    .+(`  )`.                                                          .=:(   .    ) '` ')`.                                              +
+         \  ,g88R_                                                             .=(   (   .:(   .    )                  :(   .    )                                                         (   .( (   :(   .    )                                             +
+           d888(` `).                   _                                      (   .(   ( `.  (    ) )            .-.,_`.    (  ) )                                                       (   (  ` `-'`.  (    ) )                                            +
+  -  --==  888(     ).=--           .+(` ')`.                                 (   (  `-.__..` _`-,) )          .=(    )    (   ) )                                                         `- __._;:__..` _`-,) )                                             +
+ )         Y8P(       '`.          :(   .    )                                 `- __.'                         (   .- ( ` (    .--._  _                                                                                                                       +
+         .+(`(      .    )    .--  `.  (    ) )                                                               (   (  ( ) ((...(   . '` ')`.                      _                                                  _                                         +
+        ((    (..__.:'---'..=(   )   ` _`  ) )                                                                 `- _-( .=(   (   . (   .    )                    \e/                                                \e/                                        +
+ `.     `(       ) )       (   .  )     (   )  ._                                                                    ((   .(     `.  (    ) )                    |._                                                |._                                       +
+   )      ` __.:'   )     (   (   ))     `-'.:(`  )                                                                   `- __.'-.__..` _`-,) )                     |'."-._.-""--.-"-.__.'/                            |'."-._.-""--.-"-.__.'/                   +
+ )  )  ( )       --'       `- __.'         :(      ))                                                                                                            |  \                 /                             |  \       .-.       /                    +
+ .-'  (_.'          .')                    `(    )  ))                                                                                                           |   |                (                             |   |     (@.@)      (                    +
+                   (_  )                     `-+-.:'                                                                                                             |   |                 )                            |   |   '=.|m|.='     )                   +
+                                               +     -------           +                                                                                         |  /                 /                             |  /    .='`"``=.    /                    +
+               b $                             +      $$$  C   d       +                 ++++++                                                                  |.'                 (                              |.'                 (                     +
+               -----                         --+---  ------------      +                             +++++++++                                                   |.-"-.__.-""-.__.-"-.)                             |.-"-.__.-""-.__.-"-.)                    +
+                     +                      ---+----                   +             +++++    ++++++                                                             |                                                  |                                         +
+                    +++                  -------------                 +                                                                                         |                                                  |                                         +
+a       +          +  c+                                +              +          ++++                                       +++++++++                           |                                 +                |                                         +
++  p    +         +++B+++                             + ++             +                              ++++++                                        +            |                                 +                |                                         +
++$ +    A                                             + ++++           D        +++                                                                 +            f                                 [                1                                         +
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

It's immediately clear that we won't be able to exploit the lack of bounds checking this
time since there's now walls all the way around the map. There are also some new 
mechanics shown below the map in a new status printout:

```
---------------------------------
+       +       +       +       +
+       +       +       +       +  $0
+       +       +       +       +
---------------------------------
```

The grid on the left is an inventory. There are lowercase letters `a`, `b`, `c`, and `d`
scattered across the map. These represent keys that we can add to our inventory by walking
over them and pressing <kbd>s</kbd>. Once in our inventory, we can unlock and walk past 
the corresponding gates - uppercase `A`, `B`, `C`, and `D`.

There's also a money display in the status printout. Scattered across the map are a 
handful of `$` that each increment our wallet by $1 when we walk across them!

Finally, near the far right of the map, between the false flag and the skull flag, there's
another gate that looks like this:

```
+
+
[
```

We must pay a fee to unlock this gate and reach the skull flag, as can be seen in the 
`#!c paid_unlock()` function:

```c linenums="37" hl_lines="3 5 6"
char* paid_unlock(char key, struct PlayerInfo *player_info) {
    if (blocks[key].solid == 1) {
        if (player_info->dollars == 1650549605) {
            player_info->dollars = 0;
            blocks[key].solid = 0;
            return "Unlocked!";
        }
        else if (player_info->dollars > 1650549605) {
            return "Your excessive wealth disgusts me.";
        }
        else {
            return "Sorry! You need $1650549605 to unlock me!";
        }
    }
    return "";
};
```

How do we get that much money?? There's only 5 `$` available to pick up in the game! There
must be another way... Let's take a look at where the player's wallet data is stored and
see if there's anything interesting around that:

```c linenums="13" hl_lines="6 7"
struct PlayerInfo {
    int jumpCount;
    int collected_items;
    unsigned char xpos;
    unsigned char ypos;
    char inventory[4];
    uint dollars;
};
```

Nothing seems to be too special here, but the `#!c dollars` attribute is defined directly
after the new `#!c inventory` feature. Let's go take a look at how that inventory array is
used:

```c linenums="155" hl_lines="6 7 18 19"
    if (input == 's' && current_block_properties.mustInteract) {
        if (current_block_properties.specialProperty) {
            message = (*current_block_properties.specialProperty)(current_level_location, player_info);
        }
        if (current_block_properties.collectable) {
            player_info->inventory[player_info->collected_items] = current_level_location;
            player_info->collected_items ++;
        }
        if (current_block_properties.disappears) {
            current_level_location = ' ';
        }
    }
    else if (!current_block_properties.mustInteract) {
        if (current_block_properties.specialProperty) {
            message = (*current_block_properties.specialProperty)(current_level_location, player_info);
        }
        if (current_block_properties.collectable) {
            player_info->inventory[player_info->collected_items] = current_level_location;
            player_info->collected_items ++;
        }
        if (current_block_properties.disappears) {
            current_level_location = ' ';
        }
    }
```

To understand this code snippet, it's helpful to know that `#!c collectible` means that 
the user is able to pick up an item from the space -- in this level, only keys are 
collectible. It's also helpful to know that `#!c current_level_location` is a macro that
gets the character the player is standing on (e.g. a `$` for money or an `a` for the A 
key). Something interesting is going on with this collection code, though! As the 
highlighted lines show, the game doesn't perform bounds checking for the `#!c inventory` 
array before or after allowing the user to pick up an item. Keys also aren't removed after 
they're collected because their `#!c disappears` attribute is false. This means there is 
no limit to the number of keys we can pick up!

How is that useful? Well, the location the collectibles are stored at is incremented each
time one is picked up. This means that we can continue writing into memory past the end of
the array allocated for `#!c player.inventory[]`. As we saw in the `#!c PlayerInfo` 
struct, this allows us to overwrite the previous value in `#!c player_info->dollars` and
gain significantly more money! 

Even better, we just saw that when keys are added to the (potentially overflowed) 
`#!c inventory` array, the value actually stored is the ASCII value for character shown on
screen, e.g. an `a` for the A key. These ASCII values take 1 byte each, so we can control 
how memory is overwritten on a per-byte basis.

Going back to `#!c paid_unlock()`, we need to get $1650549605 in order to beat the level. 
Let's take a look at some alternate representations of 1650549605 to figure out the order
we need to pick up extra keys in!

|             |    Big-endian   |  Little-endian  |
|-------------|-----------------|-----------------|
| Hexadecimal | `6261 6365`     | `6563 6162`     |
| ASCII       | `b` `a` `c` `e` | `e` `c` `a` `b` |


We're on a little-endian system, meaning that we need to overwrite the 
4 bytes in the `#!c uint` player wallet with keys in the order `e`, `c`, `a`, `b` to get
$1650549605. Unfortunately, there's no key `e`!

Thankfully, we don't need one. The `e` is needed for the least significant byte of
the dollar amount, meaning it translates to $101 (the ASCII code for `e` is `101`). The
ASCII code for `d` (a valid key!) is `100`, which would give us $100 if we used it first 
with the extra keys bug. We can then pick up a single `$` to get the extra dollar needed
to unlock the system! This means our new sequence for picking up extra keys is `d`, `c`, 
`a`, `b`, plus a `$` that we pick up after starting to exploit this bug.

!!! warning
    Remember that you need to leave at least 1 `$` behind to pick up after you start 
    exploiting the extra keys bug to gain extra money! The first time you exploit the bug
    and pick up another `d`, your wallet will be set to $100, regardless of what was there
    before!

Let's go back to the game and carry this out! We can jump over the `$` next to key `b`,
saving it for after we've reached all the keys and can start overwriting the wallet. After
picking up a second `d` key, our wallet should jump to $100. After this, we can grab the 
last `$` at any time and pick up the remaining 3 keys in the correct order. With that, we 
can finally make it past the toll booth to get the second flag!

??? tip "Not seeing your wallet increase the first time you exploit the bug?"
    Something interesting can happen sometimes... the first time (or usually two times!) 
    you try to exploit the bug, you may not see your wallet's value increase. This is 
    because of how the compiler lays out structs in memory. Even though `#!c uint dollars` 
    immediately follows `#!c char inventory[4]` in the struct definition, the compiler may
    add additional space in a process known as struct padding. This helps speed up 
    operations that reference struct elements, and is even required on some architectures!
    Look up struct padding and packing to learn more.

Wow, tonight's challenges were a lot of fun! It was a big step up from the first two 
weeks, for sure. It was very cool to work on challenges that require exploiting some of 
C's quirks that are usually such a pain to mitigate. The game level design and gameplay
implementation was super cool to see, too! I'm looking forward to see what's coming next 
week.