Each settlement maintains a **Trust Score** toward the player.

The trust system determines dialogue access, quest availability, and settlement privileges.

### Trust Scale

```
-100 to -50 → Hostile
-49 to -10 → Distrustful
-9 to 29 → Neutral
30 to 69 → Trusted
70+ → Allied
```

---

### Trust Modifiers

Trust changes through player behavior.

**Positive Actions**

- completing settlement tasks
- delivering supplies
- rescuing settlement members
- defending settlement from raiders

**Negative Actions**

- stealing
- attacking NPCs
- failing critical tasks
- assisting rival groups

---

### Trust Effects

Trust level influences:

- dialogue options
- trading prices
- quest availability
- settlement movement freedom

Higher trust may unlock:

- protected shelter
- rare equipment
- narrative progression

---

## Exit / Join Branching Logic

Player interaction with settlements may result in several long-term relationship outcomes.

### Temporary Visitor

Default state.

Player may:
- trade
- receive small quests
- gather information

Settlement remains cautious.

---

### Trusted Ally

Unlocked through positive trust progression.

Benefits may include:
- safe resting locations
- storage access
- advanced trade options
- settlement defense missions

---

### Settlement Member (Rare)

Some settlements may allow the player to temporarily join their operations.

Effects may include:

- access to restricted areas
- participation in group missions
- additional narrative dialogue

However, joining one settlement may damage relations with others.

---

### Expelled

Triggered if the player repeatedly violates settlement rules.

Consequences:

- settlement guards hostile
- trade blocked
- quests unavailable

Some settlements may allow reputation recovery through specific actions.

---

## Implementation Notes (for Programming)

Dialogue logic should support:

- state-based dialogue branching
- trust variable storage per settlement
- event triggers modifying trust values
- settlement access flags tied to trust thresholds

The system should remain modular so that individual settlements can override behavior or introduce unique dialogue rules.
