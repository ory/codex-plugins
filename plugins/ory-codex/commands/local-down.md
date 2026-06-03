---
description: Stop the local Ory dev stack, preserving data volumes.
---

# Stop Local Ory Environment

Stop the local Ory development environment. This gracefully shuts down
all Docker containers (Kratos, Keto, Hydra, Nginx gateway) while
preserving data volumes so you can restart later without losing state.

Run:

```bash
npx -y -p @ory/codex ory-codex local down
```

To restart later, use `ory-local-up`.

To stop **and** remove all data (full reset), run:

```bash
npx -y -p @ory/codex ory-codex local reset
```
