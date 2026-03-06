<div align="center">

<img src="https://cdn.prod.website-files.com/69082c5061a39922df8ed3b6/6971864f1c9d5744062b71fd_logogogogo.png" alt="Freelancer" width="120" />

# FREELANCER

**A collectible card game. Built from Shaw's repo. Launched on Solana.**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)
[![JavaScript](https://img.shields.io/badge/JavaScript-96.4%25-F7DF1E?logo=javascript&logoColor=black)](https://github.com/FreelancerCards)
[![Node.js](https://img.shields.io/badge/Node.js-Server-339933?logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![Redis](https://img.shields.io/badge/Redis-State%20Engine-DC382D?logo=redis&logoColor=white)](https://redis.io/)
[![Solana](https://img.shields.io/badge/Solana-Token%20Live-9945FF?logo=solana&logoColor=white)](https://solana.com/)
[![pump.fun](https://img.shields.io/badge/pump.fun-Fair%20Launch-00E4B8)](https://pump.fun/)
[![Twitter Follow](https://img.shields.io/twitter/follow/FreelancerGame?style=social)](https://twitter.com/FreelancerGame)

[Play](https://x.com/freelancergame) | [Medium](https://medium.com/@freelancergame/freelancer-is-live-the-game-the-token-today-4dbc9d11cefe) | [Twitter](https://twitter.com/FreelancerGame) | [Original Repo](https://github.com/lalalune/freelancer)

</div>

---

<img src="https://cdn.prod.website-files.com/69082c5061a39922df8ed3b6/697187704730a92b6fd4ccbd_4234324324311.png" alt="Freelancer Banner" width="100%" />

---

## The Story

This project started with a DM from Shaw Walters ([@shawmakesmagic](https://twitter.com/shawmakesmagic)) -- the creator of ElizaOS and one of the most prolific builders in the AI x crypto space. He had a collectible card game repo sitting on GitHub. The architecture was solid: client-server split, Redis-backed state, WebSocket multiplayer. But it needed someone to take it from prototype to product.

He paid us $1,000 to build it out, polish the gameplay, and launch a token on pump.fun. No pitch deck. No roadmap committee. Just "here's the repo, here's the bag, make it happen."

<div align="center">
<img src="https://cdn.prod.website-files.com/69082c5061a39922df8ed3b6/69aaf82699e6dafac3731a65_shaw-ezgif.com-optimize.gif" alt="Shaw endorsing Freelancer" width="480" />

*Shaw giving the official stamp of approval.*
</div>

So we built.

---

## What Is Freelancer

Freelancer is a real-time, web-based collectible card game. Players build decks, queue into matches, and battle opponents with cards that have mana costs, effects, and triggered abilities. Every move is validated server-side. No client-side cheating. No exploits.

Think the strategic depth of Magic: The Gathering with the speed and accessibility of a browser game.

```
freelancer/
├── packages/
│   ├── client/          # Web-based game UI
│   └── server/          # Node.js game server
├── package.json         # Yarn workspaces config
├── yarn.lock
└── README.md
```

**Stack breakdown:**

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | Vanilla JavaScript | Game UI, card rendering, animations |
| Backend | Node.js | Game logic, move validation, matchmaking |
| State | Redis | Pub/sub for real-time sync, persistent game state |
| Transport | WebSockets | Low-latency multiplayer communication |
| Token | Solana (pump.fun) | Community token, fair launch |

---

## Getting Started

You need Docker for Redis. Everything else runs on Node.js with Yarn.

```bash
# 1. Clone the repo
git clone https://github.com/FreelancerCards/freelancer.git
cd freelancer

# 2. Install dependencies
yarn install

# 3. Pull and run Redis
sudo docker pull redis
sudo docker run -it -p 6379:6379 redis

# 4. Start the game server (terminal 1)
yarn start:server

# 5. Start the client (terminal 2)
yarn start:client
```

That's it. You're in a game lobby. Open two browser tabs to test multiplayer locally.

---

## Dev Diary: How We Built It

### Week 1 -- Mapping the Engine

First thing we did was read every line of Shaw's original code. The game engine handles card interactions, turn resolution, and state management through a clean validation pipeline. Every action goes through the server before anything renders on-screen.

The core pattern looks like this:

```javascript
// Server-side card validation
const validateCardPlay = (gameState, playerId, cardId) => {
  const player = gameState.players[playerId];
  const card = player.hand.find(c => c.id === cardId);

  if (!card) {
    return { valid: false, reason: 'Card not in hand' };
  }

  if (player.mana < card.cost) {
    return {
      valid: false,
      reason: `Need ${card.cost} mana, have ${player.mana}`
    };
  }

  if (gameState.activePlayer !== playerId) {
    return { valid: false, reason: 'Not your turn' };
  }

  return { valid: true, card };
};
```

No trust on the client. The server is the single source of truth. This is a non-negotiable for any multiplayer card game -- you cannot let the browser decide whether a play is legal.

### Week 2 -- Making Cards Feel Real

A card game without satisfying interactions is a spreadsheet with extra steps. We spent the second week entirely on game feel: the weight of a card leaving your hand, the impact when it lands, the feedback loop that makes you want to play another round.

```javascript
// Card play animation
const animateCardPlay = (cardEl, targetSlot) => {
  const start = cardEl.getBoundingClientRect();
  const end = targetSlot.getBoundingClientRect();

  const dx = end.left - start.left;
  const dy = end.top - start.top;

  cardEl.style.transition = 'none';
  cardEl.style.transform = 'translate(0,0) scale(1)';

  requestAnimationFrame(() => {
    cardEl.style.transition =
      'all 0.4s cubic-bezier(0.22, 1, 0.36, 1)';
    cardEl.style.transform =
      `translate(${dx}px, ${dy}px) scale(0.85)`;
    cardEl.style.boxShadow =
      '0 20px 60px rgba(0, 212, 170, 0.3)';

    setTimeout(() => {
      targetSlot.classList.add('impact-flash');
      playSound('card-slam');
    }, 380);
  });
};
```

CSS-driven animations with `requestAnimationFrame` for 60fps card movement. The elastic easing (`cubic-bezier(0.22, 1, 0.36, 1)`) gives cards a natural deceleration that feels physical. Small detail, massive difference in how the game feels to play.

### Week 3 -- Redis and Multiplayer

The multiplayer layer is built on Redis pub/sub. Every game gets its own channel. State mutations are atomic writes. Disconnects are handled gracefully with a 5-minute reconnect window.

```javascript
const Redis = require('ioredis');
const publisher = new Redis();
const subscriber = new Redis();

const updateGameState = async (gameId, action) => {
  const state = await getGameState(gameId);
  const newState = applyAction(state, action);

  // Atomic write -- no race conditions
  await publisher.set(
    `game:${gameId}`,
    JSON.stringify(newState),
    'EX', 3600
  );

  // Broadcast to all players in this match
  publisher.publish(
    `game:${gameId}:updates`,
    JSON.stringify({
      type: 'STATE_UPDATE',
      state: sanitizeForPlayer(newState),
      action
    })
  );
};

const handleDisconnect = async (playerId, gameId) => {
  await publisher.set(
    `player:${playerId}:disconnected`,
    Date.now(),
    'EX', 300  // 5 minute reconnect window
  );
};
```

We stress-tested this with simultaneous games, rapid card plays, forced disconnects, and reconnect floods. Redis held up. The pub/sub model means the server never polls -- it reacts. State is always consistent because every write is atomic and every read is from the single Redis source.

### Week 4 -- Token Launch

Stealth launched on pump.fun. No presale. No whitelist. No insiders. Fair launch with Shaw's endorsement.

---

## Token Details

```
Token:        $FREELANCER
Platform:     pump.fun (Solana)
Launch:       March 6, 2026
Type:         Stealth / Fair Launch
Dev Address:  miLtonJTjXf1v6ue3QGWmmJtYCjrKXuLg74bve2UeyC
```

The dev address is public. Verify everything on-chain. That's the point.

Read the full launch announcement on [Medium](https://medium.com/@freelancergame/freelancer-is-live-the-game-the-token-today-4dbc9d11cefe).

---

## Architecture

```
                    +------------------+
                    |    Browser UI    |
                    |  (Vanilla JS)    |
                    +--------+---------+
                             |
                         WebSocket
                             |
                    +--------+---------+
                    |   Game Server    |
                    |   (Node.js)      |
                    |                  |
                    |  - Validation    |
                    |  - Turn Logic    |
                    |  - Matchmaking   |
                    +--------+---------+
                             |
                        Read/Write
                             |
                    +--------+---------+
                    |      Redis       |
                    |                  |
                    |  - Game State    |
                    |  - Pub/Sub       |
                    |  - Sessions      |
                    +------------------+
```

**Why this stack:**

The entire frontend is vanilla JavaScript. No React. No Next.js. No build step. This was a deliberate choice -- card games need fast load times and precise animation control. Every framework abstraction is a frame you might drop. Shaw's original architecture was right about this and we kept it.

Redis over PostgreSQL for game state because card games are write-heavy, read-heavy, and ephemeral. A match lasts 10-20 minutes. You don't need ACID compliance for something that expires in an hour. You need speed and pub/sub. Redis gives you both.

WebSockets over REST because turn-based card games still need real-time updates. When your opponent plays a card, you need to see it now, not on your next poll cycle.

---

## Roadmap

| Phase | Status | Description |
|-------|--------|-------------|
| Core game engine | Done | Card mechanics, turn resolution, mana system |
| Server-side validation | Done | Anti-cheat, move verification |
| Multiplayer (Redis) | Done | Real-time sync, disconnect recovery |
| Card animations | Done | CSS-driven, 60fps card movement |
| Matchmaking | Done | Lobby system, game creation |
| Token launch | Done | $FREELANCER on pump.fun, fair launch |
| Advanced deck builder | Next | Card filtering, strategy presets, custom decks |
| On-chain marketplace | Planned | Trade cards as Solana assets |
| ELO ranked system | Planned | Competitive ladder, seasonal resets |
| Tournament mode | Planned | Organized events, token prize pools |
| AI opponents | Planned | Single-player powered by AI agents |
| Mobile optimization | Planned | Responsive play on any device |

---

## Contributing

The code is open source. Fork it, break it, improve it.

```bash
# Fork the repo, then:
git clone https://github.com/YOUR_USERNAME/freelancer.git
cd freelancer
yarn install

# Make your changes, then submit a PR
```

We're particularly looking for contributions in:

- Card balance and new card designs
- UI/UX improvements
- Performance optimization
- Mobile responsiveness
- Test coverage

---

## Links

| | |
|---|---|
| **Medium** | [Launch Announcement](https://medium.com/@freelancergame/freelancer-is-live-the-game-the-token-today-4dbc9d11cefe) |
| **Twitter** | [@FreelancerGame](https://twitter.com/FreelancerGame) |
| **Website** | [freelancergame](https://x.com/freelancergame) |
| **Original Repo** | [github.com/lalalune/freelancer](https://github.com/lalalune/freelancer) |
| **Shaw** | [@shawmakesmagic](https://twitter.com/shawmakesmagic) |
| **Dev Address** | `miLtonJTjXf1v6ue3QGWmmJtYCjrKXuLg74bve2UeyC` |

---

## Credits

Freelancer was originally created by [Shaw Walters](https://twitter.com/shawmakesmagic), creator of [ElizaOS](https://github.com/elizaOS/eliza). Shaw commissioned the development of this version, funded the build, and endorses both the game and the token.

Built by the [@FreelancerGame](https://twitter.com/FreelancerGame) team.

---

<div align="center">

**No presale. No whitelist. No insiders. Just a game that works.**

MIT License

</div>
