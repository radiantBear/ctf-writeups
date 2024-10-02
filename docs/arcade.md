# Arcade

## Challenge Description
> The folks over at the Geriatric Gamblers' Emporium have built an arcade for the bored 
> and retired. It seems like it might take all day to win these games, though. Can you 
> find a way to beat them before you grow old?

## Dr. B. Breaker's Barely Bestirring Brick Breaker!
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

Right away, we can see that when the app makes a `#!js fetch()` call and make a GET 
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

## Raffle
Here's an explanation for the second half of the challenge
