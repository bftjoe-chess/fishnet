# Protocol

![Fishnet sequence diagram](https://raw.githubusercontent.com/niklasf/fishnet/master/doc/sequence-diagram.png)

Client asks server:

```javascript
POST http://lichess.org/fishnet/acquire

{
  "fishnet": {
    "version": "1.15.7",
    "apikey": "XXX"
  }
}
```

Response with work:

```javascript
202 OK

{
  "work": {
    "type": "analysis",
    "id": "work_id",
    "nodes": { // node limit for each eval flavor
      "nnue": 2250000,
      "classical": 4050000
    }
  },
  // or:
  // "work": {
  //   "type": "move",
  //   "id": "work_id",
  //   "level": 5 // 1 to 8
  //   "clock": { // optional
  //     "wtime": 18000, // time on white clock (centiseconds)
  //     "btime": 18000, // time on black clock
  //     "inc": 2, // fisher increment (seconds)
  //   }
  // },
  "game_id": "abcdefgh", // optional, can be empty string for bc
  "position": "rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1", // start position (X-FEN)
  "variant": "standard",
  "moves": "e2e4 c7c5 c2c4 b8c6 g1e2 g8f6 b1c3 c6b4 g2g3 b4d3", // moves of the game (UCI)
  "skipPositions": [1, 4, 5] // 0 is the first position
}
```

Response with no work found:

```
204 No Content
```

Client runs Stockfish and sends the analysis to server.
The client can optionally report progress to the server, by sending null for
the pending moves in `analysis`.

```javascript
POST http://lichess.org/fishnet/analysis/{work_id}

{
  "fishnet": {
    "version": "0.0.1",
    "apikey": "XXX"
  },
  "stockfish": {
    "flavor": "nnue" // or classical
  },
  "analysis": [
    { // first ply
      "pv": "e2e4 e7e5 g1f3 g8f6",
      "depth": 18,
      "score": {
        "cp": 24
      },
      "time": 1004,
      "nodes": 1686023,
      "nps": 1670251
    },
    { // second ply (1 was in skipPositions)
      "skipped": true
    },
    // ...
    { // second last ply
      "pv": "b4d3",
      "depth": 127,
      "score": {
        "mate": 1
      },
      "time": 3,
      "nodes": 3691,
      "nps": 1230333
    },
    { // last ply
      "depth": 0,
      "score": {
        "mate": 0
      }
    }
  ]
}
```

Or the move:

```javascript
POST http://lichess.org/fishnet/move/{work_id}

{
  "fishnet": {
    "version": "0.0.1",
    "apikey": "XXX"
  },
  "move": {
    "bestmove": "b7b8q"
  }
}
```

Query parameters:

- `?slow=true`: Do not acquire user requested analysis. Speed is not important
  for system requested analysis.
- `?stop=true`: Submit result. Do not acquire next job.

Accepted:

```
204 No content
```

Accepted, with next job:

```
202 Accepted

[...]
```

## Aborting jobs

The client should send a request like the following, when shutting down instead
of completing an analysis. The server can then immediately give the job to
another client.

```
POST http://lichess.org/fishnet/abort/{work_id}

{
  "fishnet": {
    "version": "0.0.1",
    "apikey": "XXX"
  }
}
```

Response:

```
204 No Content
```

Or abort not supported:

```
404 Not found
```

## Status

Useful to monitor and react to queue status or spawn spot instances.

```
GET http://lichess.org/fishnet/status
```

```javascript
200 OK

{
  "analysis": {
    "user": { // User requested analysis (respond as soon as possible)
      "acquired": 93, // Number of jobs that are currently assigned to clients
      "queued": 1, // Number of jobs waiting to be assigned
      "oldest": 5 // Age in seconds of oldest job in queue
    },
    "system": { // System requested analysis (for example spot check for cheat
                // screening, low priority, queue may build up during peak time
                // and should be cleared once a day)
      "acquired": 128,
      "queued": 14886,
      "oldest": 7539
    }
  }
}
```

Or queue monitoring is not supported
(for example internal [lila-fishnet](https://github.com/ornicar/lila-fishnet)):

```
404 Not found
```

## Key validation

```
GET http://lichess.org/fishnet/key/XXX
```

Key valid:

```
200 Ok
```

Key invalid/inactive:

```
404 Not found
```

## Prepared changes

These experimental changes have already been implemented in the client, and
future versions of the server might start using them.

- Client version sent as header `User-Agent: fishnet-<os>-<arch>/<version>`.
- Key sent as header `Authorization: Bearer <key>`.
  In the future the key validation endpoint may be deprecated
  in favor of a `GET /fishnet/key` request, a no-op to validate the header.
- New optional `work.depth`.
- New optional `work.multipv`, to get top _multipv_ scores and pvs
  at each depth.
- Reject client until update or reconfiguration with status code:
  - 400 Bad Request (Update required due to protocol change)
  - 401 Unauthorized (Unknown key)
  - 403 Forbidden (Key disabled)
  - 406 Not Acceptable (Update required)
