---
layout: post
title: "Beyond Booleans: Why Enums Should Be Your Go-To for State Management"
date: 2022-06-15
categories: [programming, best-practices, software-design]
---

# Beyond Booleans: Why Enums Should Be Your Go-To for State Management

Representing state can significantly impact code clarity, maintainability, and extensibility. 
Lot of times certain attributes end up being modelled as booleans which solve the requirement temporarily but can turn out to be suboptimal and does not extend easily.
Let's explore how using enums instead of boolean flags for state management can address this issue. 
Consider a simple user entity. Imagine we want to expose a field to indicate if the user is deleted or not. By default this seems straightforward with a boolean field `is_deleted` in the user.

## The Typical Boolean Approach

Many codebases track user state with boolean flags:

```python
class User:
    def **init**(self, username, email):
        self.username = username
        self.email = email
        self.is_deleted = False
```

But as requirements evolve, problems emerge:
Now imagine we get another field to indicate if the user is locked or not. Give that we already have a field is_deleted, it creates a problem around how to best model it.

## The Problems with Boolean Flags

1. **Ambiguous state combinations**: What does it mean when both is_deleted = True and is_locked = True? Is the account recoverable?
2. **Difficult state transitions**: How do we know if a locked account can be deleted or vice versa? What about other flags?
3. **Limited extensibility for new states**: What if we need to add a new states? New boolean flags clutter the interface.
4. **Maintenance overhead**: Each new flag multiplies the possible combinations, complicating testing and validation.

## The Enum Alternative

Here's the same user model refactored to use enums:

```python
from enum import Enum

class UserAccountState(Enum):
    PENDING_ACTIVATION = "pending_activation"
    ACTIVE = "active"
    DEACTIVATED = "deactivated"  
    LOCKED = "locked"
    DELETED = "deleted"

class User:
    def **init**(self, username, email):
        self.username = username
        self.email = email
        self.account_state = UserAccountState.PENDING_ACTIVATION
        
        # Boolean for subscription - this makes sense as boolean
        self.is_premium = False
```

## When Booleans Still Make Sense

Having said this, its important to note that not everything should be an enum. Booleans remain appropriate for:

1. **Truly binary attributes** that won't need additional states:
   * is_premium - Either has premium features or doesn't
   * email_verified - Either verified or not
   * requires_2fa - Either requires two-factor authentication or doesn't
2. **Independent feature flags** that don't represent state progression:
   * marketing_emails_enabled - User preference unrelated to account state
   * dark_mode_enabled - UI preference
   * public_profile - Privacy setting
3. **Simple settings** that are unlikely to gain nuance:
   * notifications_enabled
   * auto_renewal

## Benefits of the Enum Approach

This implementation offers several advantages:

1. **Clarity**: Each user is in exactly one account state, eliminating ambiguity.
2. **Extensibility**: Adding new states (like PAYMENT_OVERDUE) requires just adding an enum value.
3. **Enforced transitions**: The implementation controls state changes through methods rather than direct property manipulation.
4. **Audit trail**: State history tracks the complete lifecycle of an account.
5. **Business logic encapsulation**: Methods like can_login() and can_recover() express rules clearly.
6. **Better queries**: Finding users in a particular state is simpler than complex boolean combinations.

## Conclusion

While booleans have their place for truly binary attributes, enums provide a more robust solution for state management. By replacing is_active and is_deleted flags with a comprehensive account state enum, we can build systems that are:
* More intuitive to understand
* Easier to extend with new requirements
* Less prone to logical errors
* Better at enforcing valid state transitions

The next time you find yourself adding an is_something flag to track a user's state, ask whether an enum might provide a clearer, more maintainable solution. Your future self—and fellow developers—will thank you.