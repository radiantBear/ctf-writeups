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

1.  Make note of this line! It's going to be interesting later...

This means that the slow-moving ball is actually a liability for completing the challenge
in a reasonable timeframe. Let's see if we can speed that up!

## Raffle
Here's an explanation for the second half of the challenge
