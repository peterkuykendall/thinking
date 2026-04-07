# Transaction Boundaries in a Go Monolith

Most Go service architectures push a clean model:

- services call repositories  
- repositories take a DB handle  
- `context.Context` carries metadata only  
- read and write paths are separated (CQRS-style)  
- transactions wrap writes  

It’s tidy. It’s also how you end up with broken atomicity.

## The failure mode

The common pattern looks like this:

1. Read from the read model  
2. Check invariants / permissions  
3. Start transaction  
4. Write via the write path  

It looks disciplined. It is not.

If the read happens outside the transaction, then the decision is based on an unprotected snapshot. By the time you write, the world may have changed.

This shows up as:
- duplicate assignments that “shouldn’t happen”  
- invariant violations that “we checked for”  
- race conditions that only appear under load  

The code is clean. The guarantees are gone.

## Where CQRS breaks

Separating read and write paths is useful:
- read side → optimized for queries and view composition  
- write side → optimized for mutation  

We kept that split.

But CQRS introduces a trap:

> The read model is often treated as safe input to decision-making, even when it’s outside the transaction that enforces correctness.

If the decision depends on the read, and the read is not protected, then the separation is lying about guarantees.

## What we rejected

We did not want:
- transactions everywhere  
- collapsing read and write repos into one layer  
- sagas for single-database operations  

We also rejected two common approaches:

### 1. Passing transactions explicitly everywhere

```go
func (r *Repo) UpdateThing(ctx context.Context, tx *gorm.DB, ...)
```

This quickly spreads through the entire call graph:
- every service method  
- every repo method  
- every internal helper  

Now everything depends on transaction plumbing, even when it doesn’t need to.

The result:
- bloated signatures  
- harder-to-read code  
- infrastructure concerns leaking into business logic  

### 2. Treating CQRS purity as a hard rule

Keeping all reads in the read model, even when they drive write decisions.

This preserves structure, but breaks correctness.

## The decision

We pass the active `*gorm.DB` transaction through `context.Context` for write flows that require atomicity.

And more importantly:

> When a decision requires atomicity, the read moves into the transactional write path.

## How it works

- Default:  
  - read repos are non-transactional  
  - write repos handle mutations  

- Transactional flows:  
  - service starts TX and injects into context  
  - write path consumes TX from context  
  - decision-bearing reads execute within the same TX  

In some cases, this means:
- duplicating a read query inside the write repo  
- or moving a read from the read model into the write path  

This is deliberate.

## Why this holds up

This avoids the most common “looks correct but isn’t” pattern:
- read from read model  
- write in transaction  

It preserves CQRS where it helps:
- query clarity  
- performance  
- separation of concerns  

It avoids turning every function signature into transaction plumbing.

And it keeps transactions scoped to the flows that actually need atomicity.

## What this is not

This is not:
- abandoning CQRS  
- making transactions implicit everywhere  
- hiding database behavior  

It requires being explicit about:
- which flows require atomicity  
- which reads are decision-bearing  
- when the read model is safe vs unsafe  

## The actual takeaway

CQRS is a useful structure. It is not a correctness boundary.

Transaction boundaries belong around **decisions**, not just writes.

When those conflict, correctness wins.
