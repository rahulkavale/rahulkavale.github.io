---
layout: post
title: "Tests Over Types: Why Testing Trumps Type Hints in Python"
date: 2024-09-19
categories: [python, development]
tags: [testing, type-hints, software-engineering]
---

# Tests Over Types: Why Testing Trumps Type Hints in Python

In the Python world, static type hints have gained significant popularity since their introduction in PEP 484[^1]. 

Even with all the benefits that type hints give, I strongly feel there are some aspects that need to be considered when using those.

Considering python itself is dynamically typed, type hints kind of work just as a suggestion unless enforced. It means that type hints are as good as comments in the code.

## Type Hints: The New Comments?

### 1. Type Hints Are Often Unenforced

Unlike statically-typed languages where type checking happens at compile time, Python's type hints are optional annotations that require separate tools like MyPy or Pyright to verify. In many codebases, these tools aren't consistently integrated into CI pipelines or development workflows.

Without enforcement, type hints become glorified comments—documentation that can easily drift from reality.

### 2. Type Annotations Fall Out of Sync

Just like comments, as the code evolves, type hints often aren't maintained with the same diligence as the functional code. This creates a dangerous scenario where developers trust incorrect type information:

Without tests verifying the actual behavior, incorrect type hints can lead developers astray rather than help them.

### 3. Overly Generic Types Provide Little Value

Many codebases are filled with annotations like `Dict[str, Any]`, `List[Any]`, or just `Any`. These offer minimal safety guarantees while adding visual noise:

```python
def process_config(config: Dict[str, Any]) -> Dict[str, Any]:
    # What's in this dict? What structure should it have?
    # The type hint tells us almost nothing useful
    return config
```

Such annotations provide an illusion of type safety while offering little actual protection against errors.

## Why Tests Deliver Better Value

### 1. Tests Verify Actual Behavior

While type hints can catch certain classes of errors, tests verify that your code actually does what it's supposed to do:

```python
def test_user_creation():
    user = create_user("alice", 25)
    assert user.name == "alice"
    assert user.age == 25
    assert user.is_active is True  # Default value
```

This test verifies not just types but actual behavior and business logic.

### 2. Tests Catch What Type Checkers Miss

Type hints can't easily verify:
* Business logic correctness
* Edge case handling
* Integration between components
* Performance characteristics
* Side effects

This test verifies cascading deletions—something no type checker could ever validate.

### 3. Tests Provide Living Documentation

Good tests serve as executable documentation that stays up-to-date by necessity.

This test explains the discount calculation logic better than any comment or type hint could.

## A Balanced Approach

This isn't to say type hints have no value. They can be particularly helpful in:

1. **Public APIs**: Well-maintained type hints make your library easier to use
2. **Complex data structures**: Where the shape of data matters
3. **Refactoring**: When changing interfaces across a large codebase

A pragmatic approach might include:
* Focus on writing comprehensive tests first
* Add precise type hints to public interfaces and complex functions

## Conclusion

Type hints can be valuable, but they aren't a substitute for good tests. 

Adding tests is much better investment as compared to debating whether to add type hints or not.

[^1]: https://peps.python.org/pep-0484/