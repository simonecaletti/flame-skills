# The CachingSystem single-access-path rule

From `GUIDELINES.md`, "Coding conventions". This is the most consequential rule in the
project and the one to check first.

## Rule

If a class is cached — there is a dedicated cache object for it in
`code/lib/include/CachingSystem` — then that class must be reached **only** through its
cache. The cache owns the sole pointer to the underlying object; no other class holds
one, and no other class calls the underlying object directly. Concretely:

- the cached object is a constructor argument of the cache and a **private member** of
  it, and the cache **exposes no getter returning it**;
- every consumer stores a pointer to the **cache**, never to the object;
- the cache's `cache_*()` / `get_*()` pair is the **only** way to obtain the quantity, so
  there is exactly one code path producing it and a cache miss cannot be silently
  bypassed by recomputing.

If a class holds both quantities that need caching and quantities that do not (for
example event-dependent values alongside plain constants), that is a sign the class
should be **split**, not a reason to reach around the cache.

## Why it matters

Not micro-optimisation. Two access paths to the same quantity means two places where the
physics can drift apart, and it makes the `DISABLE_CACHE_*` builds stop being a faithful
cross-check — which is precisely the tool used when a result looks wrong.

## How to check

Find the caches, then look for anyone holding the cached type directly:

```bash
ls code/lib/include/CachingSystem/
```

```bash
grep -rn "shared_ptr<Pdf\|shared_ptr<ScaleManager\|->pdf()\|->scales()" code/lib code/process --include=*.hh --include=*.cc | grep -v CachingSystem
```

Red flags in a diff:

- a getter on a cache that returns the cached object (`get_pdf()` returning the PDF
  rather than a cached value);
- a consumer whose member is the cached type rather than the cache type;
- a call to the underlying object's compute method from outside its cache;
- a new class that mixes cacheable event-dependent state with plain constants.

## Verification

A violation is often invisible in normal runs and shows up only as a discrepancy between
a normal build and a `DISABLE_CACHE_*` build. When in doubt, build both and compare:

```bash
cmake -S code/lib -B /tmp/flame-nocache -DDISABLE_EQUIV=1 -DDISABLE_CACHE_EQUIV=1 -DDISABLE_CACHE_PHSP_REAL=1 -DCMAKE_INSTALL_PREFIX=/tmp/flame-nocache
```

The two builds must give identical cross sections. If they do not, the cache layer and
the direct path have already drifted.
