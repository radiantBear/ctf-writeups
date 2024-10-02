# 09/30/24 - Arcade

## Challenge Description
> The folks over at the Geriatric Gamblers' Emporium have built an arcade for the bored 
> and retired. It seems like it might take all day to win these games, though. Can you 
> find a way to beat them before you grow old?

## Dr. B. Breaker's Barely Bestirring Brick Breaker
The first half of tonight's challenge involves a simple brick breaker game. The ball
moves incredibly slowly, meaning that anyone's reflexes should be enough to keep the game 
going! Looking at the Express backend, we can see that the only requirement to get the 
flag from this game is earning a score of at least `5 * 6`, or 30.

```js hl_lines="14 15 16"
app.get('/breakout', (req, res, next) => {
    // If we didn't get a score parameter, handle as a static file
    if (!req.query.score) {
        next();
    }

    // Score needs to be a number
    const score = parseInt(req.query.score, 10);
    if (isNaN(score)) { // (1)!
        res.status(400).send("Hmmm");
        return;
    }

    // If they've breaked all the bricks, give the flag!
    if (score >= 5 * 6) {
        res.send(process.env.BRICK_FLAG);
    } else {
        res.send("Doesn't look like a winning score to me");
    }
});
```

1.  Take note of this line! It's going to be interesting later...

This means that the slow-moving ball is actually a liability for completing the challenge
in a reasonable timeframe. Let's see if we can speed that up! We'll start by taking a peek 
at the client-side code:

```js hl_lines="14 19 20 21"
async function detectBrickCollision() {
    for (let c = 0; c < brickRowCount; c++) {
        for (let r = 0; r < brickColumnCount; r++) {
            const brick = bricks[c][r];
            if (brick.status === 1) {
                if (x + ballRadius > brick.x && x - ballRadius < brick.x + brickWidth &&
                    y + ballRadius> brick.y && y - ballRadius < brick.y + brickHeight) {
                    dy = -dy; // Bounce the ball
                    brick.status = 0; // Remove the brick
                    score++; // Increase score
                    if (score >= brickColumnCount * brickRowCount) {
                        try {
                            // Send the guess to the /raffle endpoint
                            const response = await fetch(`/breakout?score=${score}`);
                            // Check if the request was successful
                            if (!response.ok) {
                                throw new Error("Error contacting the server. Please try again.");
                            }
                            let flag = await response.text()
                            console.log(flag)
                            alert("You win!: " + flag);
                        } catch (error) {
                            alert("You were supposed to get the flag but we broke something. sorry, tell a coach lol.");
                        }
                        document.location.reload();
                    }
                }
            }
        }
    }
}
```

Right away, we can see that when the app calls `#!js fetch()` to make a GET 
request to `/breakout`, the flag should be returned! The condition when the app makes a 
GET request (`#!js if (score >= brickColumnCount * brickRowCount)`) is the same as what 
the backend is looking for: `brickColumnCount` is defined as `6`, and `brickRowCount` is 
defined as `5`. This reinforces the idea that retrieving the flag should be as simple as 
making a GET request to `/breakout` with a `score` parameter of at least 30. Let's try it!

```js
const data = await fetch('/breakout?score=31');
console.log(await data.text());
```

And with that, we have the flag! On to the next half...

## Dr. Ralph Fleur's Baffling Raffle
The second half of tonight's challenge ups the ante a bit. A second game provided to the
Geriatric Gamblers' Emporium is a raffle where the user can pick 5 numbers and roll to see
if they correctly guessed the numbers. It'll take a ***very*** long time to correctly
guess all 5 numbers, so let's take a look at the Express backend again and see if we can
find a workaround!

```js
app.get('/raffle', (req, res) => {
    const guesses = req.query;
    let digits = Array.from({length: 5}, () => Math.floor(Math.random() * 10));

    // Make sure that the guess exists and isn't too high or too low
    if (!guesses.field1 || parseInt(guesses.field1) > digits[0] || parseInt(guesses.field1) < digits[0]) {
        res.send( { winningDigits: digits, flag: 'No flag 4 u!'})
    }
    else if (!guesses.field2 || parseInt(guesses.field2) > digits[1] || parseInt(guesses.field2) < digits[1]) {
        res.send( { winningDigits: digits, flag: 'No flag 4 u!'})
    }
    else if (!guesses.field3 || parseInt(guesses.field3) > digits[2] || parseInt(guesses.field3) < digits[2]) {
        res.send( { winningDigits: digits, flag: 'No flag 4 u!'})
    }
    else if (!guesses.field4 || parseInt(guesses.field4) > digits[3] || parseInt(guesses.field4) < digits[3]) {
        res.send( { winningDigits: digits, flag: 'No flag 4 u!'})
    }
    else if (!guesses.field5 || parseInt(guesses.field5) > digits[4] || parseInt(guesses.field5) < digits[4]) {
        res.send( { winningDigits: digits, flag: 'No flag 4 u!'})
    }
    else {
        res.send( { winningDigits: digits, flag: process.env.SLOTS_FLAG})
    }
});
```

That's a lot of conditional statements! What's going on with them? They all follow the 
same pattern, so let's break down each condition:

1. `#!js !guesses.field_`: This checks if the user submitted a "falsy" (invalid) guess for
    this field. Values like `false`, `""`, `null`, and `0` will make this condition true, 
    causing the response "No flag 4 u" to be returned.

2. `#!js parseInt(guesses.field_) > digits[_]`: This checks if the user guessed too large
    of a value for this field. If they did, the response "No flag 4 u" will also be 
    returned.

3. `#!js parseInt(guesses.field_) < digits[_]`: This checks if the user guessed too small
    of a value for this field. If they did, the response "No flag 4 u" will still be 
    returned.
    
In summary, the response "No flag 4 u" will be returned any time the user submits a
"falsy" value like an empty field or guesses the wrong number for any of the 5 fields. But
what happens if the user submits a truthy answer that's *not a number at all?* 

Remember the note from the first half that I said to remember? If not, it was 
`#!js if (isNaN(score))`. That kind of check isn't happening anywhere in this function! If
the developer here had followed the KISS principle and performed inequality checks with 
the `#!js !=` operator, this wouldn't matter. However, the `>` and `<` comparisons give us
a loophole to exploit.

How is that? Well, if the `#!js parseInt()` function is given an input that cannot be 
converted into an integer (say, `#!js "@#$%"`), it will return `NaN`. `NaN` is falsy, but
the falsy check is performed on the raw input, which could be a truthy string, so that 
doesn't stop us. The raw input only becomes `NaN` when it is put through the 
`#!js parseInt()` function. Now for the catch! As explained on 
[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/NaN),

> When `NaN` is one of the operands of any relational comparison (`>`, `<`, `>=`, `<=`), 
> the result is always `false`.

This means that if we provide an input for one of the fields that consists of non-numeric
characters (but isn't empty, since `""` is falsy!), the conditionals will evaluate as 
such:

1. `#!js !guesses.field_` is `#!js false` (string isn't a falsy value)

2. `#!js parseInt(guesses.field_) > digits[_]` is `#!js false` (relational comparison with
    `NaN`)

3. `#!js parseInt(guesses.field_) < digits[_]` is `#!js false` (relational comparison with
    `NaN`)

Since these conditions are all `OR`d together, the entire condition in the `#!js if`
statement will evaluate to `false` as well, meaning we found a way to bypass the "No flag 
4 u!" message without needing to guess all the correct numbers! If we do the same thing 
for each field, we should make it to the `#!js else` block and get the flag returned to 
us!

Unfortunately, the app frontend will only let us submit numeric input, so let's take a 
look at its source and see if we can find a way to send a spoofed GET request like we did
last time.

```js hl_lines="2 3 11"
async function loadSlots(a, b, c, d, e) {
    let response = await fetch(`/raffle?field1=${a}&field2=${b}&field3=${c}&field4=${d}&field5=${e}`);
    let results = await response.json()
    let slots = Array(numSlots).fill().map((element, index) => ({
        yOffset: 0,
        targetIcon: results.winningDigits[index],
        spinning: true,
        speed: slotSpeed,
        spinCount: index
    }));
    flag = results.flag
    return slots;
}
```

Sweet! It looks like if we just make a GET request to `/raffle` with non-numeric strings
for each field and parse the response as JSON, we should get our flag in the response's 
`.flag` property. Let's try it out: 

```js
const data = await fetch('/raffle?field1=@&field2=@&field3=@&field4=@&field5=@');
console.log(await data.json());
```

And with that, we completed tonight's challenge! It was a great refresher after taking the
summer off.