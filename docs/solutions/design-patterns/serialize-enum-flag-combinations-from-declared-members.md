---
module: django.db.migrations.serializer
date: 2026-10-13
problem_type: design_pattern
component: migrations
severity: medium
evidence_grounding: speculative
audience: both
related_components:
  - EnumSerializer
  - migration writer
tags:
  - enum
  - flag
  - serialization
  - migrations
  - python-version-differences
applies_when:
  - Writing an `enum.Flag` or `enum.IntFlag` value out as Python source that must rebuild the same value
  - A flag value may be a combination of members that has no declared name of its own
---

# Serialize enum flag combinations from declared members, not from Python's decomposition

## Context

Migrations write each field default as Python source. Before this change, `EnumSerializer` wrote `Class['NAME']` using `value.name`. A combined flag such as `re.UNICODE | re.IGNORECASE` has no single member name. Depending on the Python version, its name is `None` or a string like `'IGNORECASE|UNICODE'`. Either way the migration contained a lookup that fails when the migration loads.

The issue suggested the private `enum._decompose` helper. Review of this change found several ways that approach, or plain iteration of the flag value, goes wrong:

- The helper is gone in Python 3.11. There, iterating a flag gives single-bit members in a different order.
- On older versions, the helper can return cached unnamed members for bits that no member declares. Their names write out as `[None]` again.
- Passing the flag object instead of its integer to the helper can fail for a plain `enum.Flag`. The `IntFlag` tests hide this.
- Iteration looks for single bits. A declared multi-bit member such as `GROUP = 3` has no single-bit members to find.
- Iterating a named zero member such as `NONE = 0` yields nothing, so the written default becomes an empty expression.

## Solution

Use a version-independent method that only ever writes names the class declares:

1. If `self.value.name in enum_class.__members__`, keep the existing single `Class['NAME']` form. This keeps named zero members (`NONE`) and declared composite members (`BOTH = 3`) unchanged.
2. Otherwise, sort the declared members by value, largest first. Take each non-zero member whose bits all fall inside the value and that adds at least one bit not yet covered.
3. Reverse the taken list, so the output is in ascending value order whatever order the class declares its members in.
4. If the taken members do not cover the whole value, or the list is empty, raise `ValueError` instead of writing a default that loads as a different value.
5. Join each member's `module.Qualname['NAME']` with ` | `.

```python
declared = sorted(set(enum_class.__members__.values()), key=lambda m: m.value, reverse=True)
for member in declared:
    bits = member.value
    if bits and bits & value == bits and bits & ~covered:
        members.append(member)
        covered |= bits
members.reverse()
if covered != value or not members:
    raise ValueError(...)
```

## Why it matters

A migration is generated once and loaded many times, often on a different Python version. If the written expression depends on how the generating interpreter decomposes flags, the same model can produce different migrations, or a migration that does not load, on another version. Using only `__members__` means every written name is a real lookup key. The coverage check stops a bit from being silently dropped, which would otherwise change the default without any error.

The trade-off is that some values that used to be written (as a broken `Class[None]`) now make serialization raise. A clear error at `makemigrations` time is better than a migration that fails, or loads a different value, later.

## Evidence

- PR #2, head commit 3abe4741fbd5021eb6fe45e14a1e6d1931d255cc, `django/db/migrations/serializer.py`: `EnumSerializer.serialize` checks `self.value.name in enum_class.__members__` first, then splits the value against the sorted `__members__`, and raises `ValueError` when `covered != value or not members`. The code comment there says Python versions differ in how they decompose flags and may yield unnamed members.
- `tests/migrations/test_writer.py` in the same commit: `test_serialize_named_enum_flags` (named `NONE = 0` and `BOTH = 3` keep their single name), `test_serialize_enum_flags_combined_members` (`GroupedFlagEnum` declares `EXTRA = 4` before `GROUP = 3`, and the output is `GROUP | EXTRA`), and `test_serialize_enum_flags_without_members` (`IntFlagEnum(0)` and `IntFlagEnum(5)` raise).
- The specific Python 3.8-3.10 versus 3.11 behaviours listed in Context come from review notes on this pull request. They were checked only on Python 3.13, so they are not shown by the diff itself. Verification step: run the tests above on Python 3.10 and 3.11.
