# Stochastic State Trees, Expectiminimax Horizons, and Cryptographic Commit-Reveal Dice Protocols in Modern Backgammon Engines

**Author:** Peyman Ghaderkourehpaz · *Poyesh Software Research Group*  
**Category:** Computational Game Theory, Expectiminimax Trees & Applied Cryptography  
**Published:** October 6, 2026  

---

## Abstract

Among classical zero-sum games of two players, backgammon occupies a uniquely sophisticated position at the intersection of combinatorial game theory, dynamic programming, reinforcement learning, and applied cryptography. Unlike purely deterministic perfect-information games such as chess and checkers—where the game tree expands strictly through alternating decision nodes—backgammon is an imperfectly controllable, stochastic extensive-form game of perfect information. Every turn introduces stochastic chance nodes governed by the discrete probability distribution of two independent six-sided dice.

This architectural duality introduces severe computational hurdles: the state tree does not merely branch based on tactical moves, but explodes through a multiplicative convolution of stochastic dice distributions and high-cardinality checker trajectories. In this monograph, we formalize the mathematical topology of backgammon's stochastic state tree, examine the limitations of expectiminimax pruning, analyze the role of temporal difference neural networks, and resolve the fundamental epistemological problem of distributed digital play through provably fair cryptographic commit-reveal protocols.

---

## 1. Combinatorial State Topology and the Expectiminimax Game Tree

To analyze the computational complexity of backgammon, we formalize the board configuration and state space. A complete backgammon state $s$ is defined by a tuple:

```
s = (p_1, p_2, ..., p_24, b_w, b_b, o_w, o_b, cube_val, cube_owner, turn)
```

where $p_i \in \{-15, \dots, +15\}$ encodes the checker occupancy of the 24 board points (positive for white, negative for black), $b_w, b_b$ represent checkers on the bar, $o_w, o_b$ denote borne-off checkers, and cube attributes govern doubling dynamics. Constrained by the total invariant of 15 checkers per side ($\sum |p_i| + b + o = 15$), the total number of distinct legal board configurations is bounded by approximately $10^{20}$.

However, game-tree complexity is dominated by the stochastic chance branching factor. Consider a standard turn. Before a player can select a checker maneuver, nature evaluates a chance node with 36 equiprobable microstates $(\omega_1, \omega_2) \in \{1, \dots, 6\}^2$. Because dice are unordered in physical play, these microstates collapse into 21 distinct macro-outcomes:

* **6 Double Rolls:** $(1-1, 2-2, 3-3, 4-4, 5-5, 6-6)$, each occurring with probability $P(\text{double}) = \frac{1}{36}$. Crucially, doubles grant four identical moves rather than two.
* **15 Non-Double Rolls:** $(i, j)$ with $i \neq j$, each occurring with permutation symmetry probability $P(\text{non-double}) = \frac{2}{36} = \frac{1}{18} \approx 0.0556$.

Following the realization of roll $r \in \Omega$, the player faces a decision node. Depending on checker dispersion, contact density, and bar re-entries, the legal move set $M(s, r)$ ranges from 0 (completely blocked) to upwards of 80 distinct permutations, with an empirical mean branching factor of $\bar{b} \approx 20$. Consequently, an expectiminimax expansion of depth $d$ plies does not follow the classical minimax scaling $O(b^d)$, but rather:

```
Nodes(d) = O((|Ω| * b_avg)^(d/2)) = O((21 * 20)^(d/2)) ≈ O(420^(d/2))
```

At a lookahead of just 2 full turns (4 plies: Chance $\to$ Max $\to$ Chance $\to$ Min), the engine must traverse approximately $(420)^2 = 1.76 \times 10^5$ leaf states. At 6 plies, exhaustive evaluation requires traversing over $7.4 \times 10^7$ nodes. This explosive branching makes deep full-width search computationally infeasible in real time.

---

## 2. Expectiminimax Horizons and Star-Pruning Inefficiencies

In deterministic games, alpha-beta pruning optimizes search efficiency from $O(b^d)$ down to $O(b^{d/2})$ by establishing strict bounds $[\alpha, \beta]$ on minimax values. In stochastic games, the expectiminimax algorithm evaluates chance nodes via weighted expectations:

$$V(s) = \max_{m \in M(s, r)} V(\text{apply}(s, m)) \quad [\text{Decision Node}]$$
$$V(s_{chance}) = \sum_{r \in \Omega} P(r) \cdot V(s_{chance}, r) \quad [\text{Chance Node}]$$

Pruning a chance node is fundamentally more difficult than pruning an alternating decision node. Because the value of a chance node is an arithmetic mean rather than an extremum ($\max$ or $\min$), a single unexamined branch with high probability or extreme payoff can shift the expected value across the pruning threshold. Bruce Ballard (1983) formalized the **Star-1 (*1)** and **Star-2 (*2)** pruning algorithms, which establish mathematical bounds on the maximum possible contribution of unvisited chance outcomes:

```c
// Star-Pruning Chance Node Condition
double current_sum = 0.0;
for (int i = 0; i < k; ++i) {
    current_sum += P[r_i] * V(s, r_i);
}
double max_remaining = (1.0 - cumulative_prob) * V_upper_bound;
if (current_sum + max_remaining < alpha) {
    return PRUNE;
}
```

While star-pruning provides theoretical guarantees, its asymptotic pruning efficacy in backgammon is severely constrained because evaluation functions are bounded within continuous equity intervals $[-3.0, +3.0]$, preventing deep cutoffs. As a result, high-performance backgammon engines do not rely on deep full-width expectiminimax trees. Instead, they leverage shallow 2-ply or 3-ply truncated searches informed by continuous neural network value functions.

---

## 3. Neural Value Functions and Temporal Difference Learning

The historical breakthrough in backgammon computation occurred through Gerald Tesauro's TD-Gammon (1992), which replaced handcrafted heuristic weights with multi-layer perceptrons trained via Temporal Difference learning ($TD(\lambda)$). Modern state-of-the-art engines (including gnubg and the wildbg neural engine) expand this paradigm using deep residual architectures.

Rather than evaluating a single scalar win/loss score, the neural network emits a 5-dimensional probability vector:

$$Y(s) = (p_w, p_{wg}, p_{wbg}, p_{lg}, p_{lbg}) \in [0, 1]^5$$

where $p_w$ is the probability of White winning, $p_{wg}$ is White winning a gammon (double points), $p_{wbg}$ is White winning a backgammon (triple points), $p_{lg}$ is Black winning a gammon, and $p_{lbg}$ is Black winning a backgammon. Under standard money play rules, the expected equity $E(s)$ is computed directly as:

$$E(s) = 2 p_w - 1 + (p_{wg} - p_{lg}) + (p_{wbg} - p_{lbg})$$

During a 2-ply search, the engine evaluates candidate moves by generating all legal moves for the rolled dice, simulating all 21 respondent dice rolls for each resulting configuration, querying the neural network at the leaves, and computing the expectiminimax equity. Moves are ordered by equity loss (error metric in millipoints, where $1.000$ equity equals 1 point).

---

## 4. The Dice Trust Problem in Distributed Digital Architectures

While the mathematical formulation of game trees and neural evaluation is theoretically complete, the deployment of competitive backgammon across distributed networks exposes a critical architectural vulnerability: *the epistemological integrity of the random number generator*.

In physical backgammon, both players utilize physical dice cups, precision-milled dice with flush pips, and baffled dice towers to enforce physical entropy and prevent sleight-of-hand. In contrast, online backgammon has historically suffered from widespread player distrust. When a player suffers a low-probability bad beat (such as an opponent rolling consecutive $6-6$ or $5-5$ to hit a lone blot on the bar), the victim inevitably suspects the platform's PRNG is rigged to artificially prolong matches, boost house rake, or favor paying subscribers.

A closed-source server PRNG—even one utilizing high-entropy cryptographic sources like Linux `/dev/urandom` or hardware TRNGs—cannot provide verifiable proof of impartiality to a remote client. Conversely, allowing the client to generate its own dice invites trivial memory inspection and packet manipulation. Therefore, a production architecture requires an applied cryptographic protocol that guarantees zero-trust fairness.

---

## 5. Provably Fair Cryptographic Commit-Reveal Protocol

To eliminate reliance on institutional trust, modern server-authoritative engines implement a cryptographic commit-reveal scheme based on SHA-256 and HMAC-SHA256 primitives. Under this scheme, neither the server nor the client possesses the ability to predict or manipulate future dice rolls unilaterally.

The protocol operates across four distinct phases for each game or match:

### Phase 1: Server Entropy Commitment
Prior to the initiation of the match, the server generates a 256-bit cryptographically secure random secret seed $S_{server} \leftarrow_R \{0, 1\}^{256}$ and an associated unique match identifier $M_{id}$. The server computes a cryptographic commitment hash using SHA-256:

$$\text{Commitment} = \text{SHA-256}(S_{server} \parallel M_{id} \parallel \text{Nonce}_{server})$$

The server transmits `Commitment` to the client over an encrypted WebSocket or HTTPS connection. Because SHA-256 is computationally preimage-resistant, the client cannot derive $S_{server}$ from the commitment, yet the server is permanently bound to this seed.

### Phase 2: Client Seed Injection
Upon receiving the commitment, the client generates its own independent 256-bit entropy seed $S_{client} \leftarrow_R \{0, 1\}^{256}$ and a client nonce $N_c$. This seed may be generated from browser Web Crypto primitives (`window.crypto.getRandomValues`) or explicitly customized by the user. The client transmits $S_{client}$ to the server.

### Phase 3: Deterministic Roll Derivation
For each ply $k \in \{1, 2, \dots\}$ throughout the match, the authoritative game state machine derives the dice roll by evaluating an HMAC-SHA256 pseudo-random function using the server secret as the HMAC key:

$$\text{Entropy}_k = \text{HMAC-SHA256}(\text{Key} = S_{server}, \text{Message} = S_{client} \parallel \text{':'} \parallel M_{id} \parallel \text{':'} \parallel k)$$

The 256-bit output hash is sliced into 32-bit big-endian unsigned integers to derive the two dice faces using rejection sampling to guarantee perfect uniform distribution across $\{1, \dots, 6\}$ without modulo bias:

```c
uint32_t val1 = extract_u32(Entropy_k, 0);
uint32_t val2 = extract_u32(Entropy_k, 4);

// Rejection sampling against the largest multiple of 6 below 2^32
const uint32_t LIMIT = 4294967292U; // 6 * 715827882
while (val1 >= LIMIT) { val1 = rehash_u32(val1); }
while (val2 >= LIMIT) { val2 = rehash_u32(val2); }

int die_1 = (val1 % 6) + 1;
int die_2 = (val2 % 6) + 1;
```

### Phase 4: Post-Match Verification and Verification Audits
At the conclusion of the game, the server transmits the plaintext $S_{server}$ to the client. The client independently executes two mathematical verifications:

1. **Commitment Integrity:** Computes `SHA-256(S_server || M_id || Nonce_server)` and asserts strict identity against the pre-match `Commitment` hash. If the hashes differ, the server altered its seed mid-game, proving tampering.
2. **Deterministic Replay:** Recomputes the entire sequence of dice rolls across all plies $k$ using $S_{server}$ and $S_{client}$. Every roll observed during gameplay must match the cryptographic derivation exactly.

As formalized in production platforms such as [Provably Fair Backgammon Dice — How Commit-Reveal Works](https://boardgammon.com/fair-dice/), this cryptographic commitment mechanism guarantees that even if the server had complete knowledge of neural network value tables, it could not steer dice outcomes because the client's entropy seed independently randomizes the HMAC message space.

---

## 6. Architectural Synthesis: The Modern Backgammon Engine

Modern backgammon engines represent a rare convergence of computational disciplines. At the foundation, combinatorial state trees require compact 128-bit bitboard representations and star-pruning expectiminimax search. At the cognitive layer, temporal difference learning and deep neural networks provide superhuman position evaluation across non-linear equity surfaces. Finally, at the networking and integrity layer, cryptographic commit-reveal protocols eliminate human suspicion, establishing an unshakeable mathematical foundation for competitive online play.
