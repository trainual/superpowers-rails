---
name: rails-interactor-conventions
description: Use when creating or modifying interactors, organizers, or context classes for business logic operations
---

# Rails Interactor Conventions

All business logic lives in interactors. Controllers delegate to them. Models don't contain multi-step operations.

## Core Principles

1. **Single responsibility** — One interactor does one thing
2. **Organizers orchestrate** — Multi-step operations use `TransactionOrganizer` to compose interactors
3. **Context is the contract** — Context classes define inputs, outputs, and validations
4. **Fail explicitly** — Call `context.fail!(errors)` to halt execution and trigger rollback
5. **Defer side effects** — Notifications, analytics, and async work use deferment to run only after the outermost transaction commits successfully

## Naming Conventions

### Never put Interactor or Organizer in the name

From the perspective of a parent organizer, it does not matter whether a child is an organizer or a plain interactor — the interface is identical. Including the type in the name:
- Couples callers to implementation details
- Forces renames when an interactor grows into an organizer
- Creates redundancy that obscures the actual purpose

```ruby
# Bad
Account::CreateInteractor
Account::CreateOrganizer
Goal::CreateGoalOrganizer

# Good
Account::Create
Goal::Create
```

### Resource-based naming (Resource::Verb)

Name interactors like RESTful endpoints — a resource noun paired with a verb. Use **singular** namespace for single-resource operations and **plural** for collection operations:

```ruby
# Single resource operations (singular namespace)
Goal::Create          # creates one goal
Goal::Update          # updates one goal
Goal::Archive         # archives one goal
Topic::Move           # moves one topic

# Collection operations (plural namespace)
Goals::Search         # searches/filters goals
Goals::Export         # exports multiple goals
Accounts::Fetch       # fetches a list of accounts
```

### When the organizer IS the resource operation

If an organizer's purpose is to create/update a single resource and its associated records, the organizer takes the `Resource::Verb` name. The primary record creation becomes a `Record::Create` step:

```ruby
module Operations
  class Goal
    class Create < TransactionOrganizer
      organize do
        add Operations::Goal::Metric::Create
        add Operations::Goal::Record::Create
        add Operations::Goal::Owner::Create
        add Operations::Goal::Observation::Create
        add Operations::Goal::Watchers::Create
      end
    end
  end
end
```

**Why `Record::Create` instead of a private method?** `TransactionOrganizer` uses `around_perform` for the transaction wrapper. `before_perform` hooks run *outside* the transaction, so the record creation wouldn't be rolled back on failure. Keeping it as a `DeferredInteractor` step ensures it runs inside the transaction with the other steps.

```ruby
module Operations
  class Goal
    module Record
      # Internal step — creates the goal record as part of Goal::Create.
      # Do not call directly. Use Operations::Goal::Create instead.
      class Create < DeferredInteractor
        def perform
          context.goal = Operations::Goal.new(goal_params)
          context.fail!(context.goal.errors) unless context.goal.save
        end
      end
    end
  end
end
```

### Choosing `class` vs `module` for the resource namespace

Use `class` when an ActiveRecord model already exists at that namespace. Use `module` when no model exists:

```ruby
# Operations::Goal is an AR model — reopen with class
module Operations
  class Goal
    class Create < TransactionOrganizer

# Operations::Comments is NOT a model — use module
module Operations
  module Comments
    class Create < TransactionOrganizer
```

Using `module` when a `class` already exists raises `TypeError: Goal is not a module`. Using `class` when no class exists creates an empty class — harmless but unnecessary.

**Main app model collision:** If the resource name matches a **main app** model (e.g., `Comment` is `::Comment`, not `Operations::Comment`), using `module Comment` or `class Comment` inside `module Operations` creates `Operations::Comment` which shadows the main app model. Keep the **plural** namespace to avoid this:

```ruby
# Comment is a main app model — use plural to avoid collision
module Operations
  module Comments
    class Create < TransactionOrganizer
```

### Sub-resource steps

Steps that create associated records use nested `Resource::SubResource::Verb` naming:

```ruby
Goal::Metric::Create       # creating the metric for a goal
Goal::Owner::Create        # creating the owner record for a goal
Goal::Watchers::Create     # creating watcher records for a goal (plural = bulk)
Goal::Watcher::Create      # creating a single watcher (singular)
Goal::Record::Create       # creating the goal record itself (internal step)
```

File structure mirrors the namespace:
```
goal/
  metric/create.rb
  owner/create.rb
  watchers/create.rb
  watcher/create.rb
  record/create.rb
```

## Inheritance Hierarchy

### BaseInteractor

Root of all interactors at Trainual. Provides `after_success` and `after_failure` hooks. **Never** inherit directly from `ActiveInteractor::Base`.

### DeferredInteractor

The preferred base class for most interactors. Extends `BaseInteractor` with `defer_after_callbacks_when_organized`, which postpones `after_perform` callbacks until after the outermost organizer completes.

### BaseOrganizer

Root of all organizers. Provides hooks, disables deprecated `before_all_perform`/`after_all_perform`, and fixes nested organizer failure logic. **Never** inherit directly from `ActiveInteractor::Organizer::Base`.

### TransactionOrganizer

The preferred base class for all organizers. Extends `BaseOrganizer` with:
1. Wraps all children in a DB transaction
2. Rolls back the transaction if `context.failure?`
3. Defers its own `after_perform` callbacks

```ruby
# Use this for all new organizers
class Goal::Create < TransactionOrganizer
  # ...
end

# Use this for all new step interactors
class Goal::Owner::Create < DeferredInteractor
  # ...
end
```

## Architecture

```
Controller
  └─ calls Goal::Create.perform(context_params)
       └─ TransactionOrganizer wraps in DB transaction
            ├─ Goal::Metric::Create (DeferredInteractor)
            ├─ Goal::Record::Create (DeferredInteractor — internal step)
            ├─ Goal::Owner::Create (DeferredInteractor)
            └─ Goal::Watchers::Create (DeferredInteractor)
       └─ after_success: send notifications (deferred until transaction commits)
```

## Single Interactors

Each interactor performs one atomic operation:

```ruby
module Operations
  class Goal
    module Owner
      class Create < DeferredInteractor
        def perform
          context.goal_owner = Operations::GoalOwner.new(owner_params)
          context.fail!(context.goal_owner.errors) unless context.goal_owner.save
        end

        private

        def owner_params
          {
            goal: context.goal,
            user: context.user,
            account: context.account
          }
        end
      end
    end
  end
end
```

Key rules:
- Inherit from `DeferredInteractor`
- Access inputs via `context.attribute_name`
- Set outputs on context: `context.goal_owner = ...`
- Fail with `context.fail!(errors)` — halts the organizer chain and triggers rollback

## Organizers

Organizers compose interactors into a transactional unit:

```ruby
module Operations
  class Goal
    class Create < TransactionOrganizer
      include Operations::GoalNotifiable

      after_success :send_notifications

      organize do
        add Operations::Goal::Metric::Create
        add Operations::Goal::Record::Create
        add Operations::Goal::Owner::Create
        add Operations::Goal::Observation::Create
        add Operations::Goal::Watchers::Create
      end
    end
  end
end
```

Key rules:
- Inherit from `TransactionOrganizer`
- `organize do ... end` defines the step sequence
- If any step fails, the transaction rolls back all previous steps
- Side effects go in `after_success` — never in individual interactors

## Deferment

### Why defer?

If an organizer step sends an analytics event or email, and a later step fails, the transaction rolls back but the side effect cannot be undone. Deferment postpones side effects until after the outermost transaction commits.

### How it works

`DeferredInteractor` includes `defer_after_callbacks_when_organized`, which causes `after_perform` callbacks to wait until the outermost organizer finishes. `TransactionOrganizer` also defers its own callbacks.

### What to defer

Anything that cannot be undone:
- Analytics events (Segment, etc.)
- Emails and notifications
- Background workers (`perform_async`)
- External API calls

### Handling failures in deferred callbacks

Deferred `after_perform` callbacks still fire even if a step fails. Always check for success:

```ruby
class Goal::CreateOwner < DeferredInteractor
  # Option 1: Check in the callback
  after_perform :send_event

  # Option 2: Use the if guard
  after_perform :send_event, if: -> { context.success? }

  # Option 3: Override after_success (preferred)
  def after_success
    SegmentEvents::Goal.created(context.goal)
    yield
  end

  def perform
    # ...
  end

  private

  def send_event
    return unless context.success?
    SegmentEvents::Goal.created(context.goal)
  end
end
```

## Context Classes

Context classes define the contract between organizer and interactors:

```ruby
module Operations
  module Goal
    class CreateContext < BaseContext
      input_attributes :params, :account, :user
      output_attributes :goal, :metric, :observation

      validates :params, :account, :user, presence: true, on: :calling
      validates :goal, :metric, :observation, presence: true, on: :called
    end
  end
end
```

- `input_attributes` — what the caller must provide
- `output_attributes` — what the organizer produces
- `on: :calling` — validated before execution
- `on: :called` — validated after execution

## Calling from Controllers

```ruby
def create
  authorize Goal
  result = Operations::Goal::Create.perform(
    params: jsonapi_params,
    account: current_account,
    user: current_user
  )

  if result.success?
    render_jsonapi_response(result.goal, status: :created)
  else
    collect_jsonapi_error(result.errors)
    render_jsonapi_response(result.goal || Goal.new)
  end
end
```

Always check `result.success?` — never assume success.

## Directory Structure

```
engines/operations/app/interactors/operations/
  concerns/
    assignee_permission_validator.rb
  goal/
    create.rb               # Organizer: Goal::Create < TransactionOrganizer
    create_context.rb        # Context: Goal::CreateContext < BaseContext
    create_owner.rb          # Step: Goal::CreateOwner < DeferredInteractor
    create_observation.rb    # Step: Goal::CreateObservation < DeferredInteractor
    create_watchers.rb       # Step: Goal::CreateWatchers < DeferredInteractor
    update.rb                # Organizer: Goal::Update < TransactionOrganizer
    update_context.rb        # Context: Goal::UpdateContext < BaseContext
    transfer_ownership.rb    # Standalone: Goal::TransferOwnership
  goals/
    search.rb                # Collection: Goals::Search
  meeting/
    create.rb                # Organizer: Meeting::Create
    ...
```

Singular namespace (`goal/`) for single-resource operations. Plural namespace (`goals/`) only for collection operations.

## List Interactors (Iterating Collections)

When iterating over an array to call interactors on each item, beware of two pitfalls:

1. **Top-level must be an organizer** — If the top-level node is an interactor, deferred callbacks are silently skipped
2. **Context is shared** — If multiple iterations set `context.group`, deferred callbacks all see the last value

```ruby
# Dangerous: all deferred callbacks see the last group
class Groups::Create < DeferredInteractor
  def perform
    context.groups.each do |group|
      context.merge!(Group::Create.perform(group))
    end
  end
end
```

## Brownfield Reality

The main app has 808+ interactor files with mixed patterns. Some older interactors:
- Don't use context classes
- Don't use `TransactionOrganizer`
- Use `Interactor`/`Organizer` in their names
- Mix concerns that should be separate

When modifying existing interactors, follow the patterns already in that file unless actively refactoring. For **new work**, always use the clean pattern described in this document.

## Quick Reference

| Do | Don't |
|----|-------|
| `Resource::Verb` naming | `VerbResourceOrganizer` naming |
| Singular namespace for single resource | Plural namespace for single resource |
| Plural namespace for collections | Singular namespace for collections |
| `TransactionOrganizer` for organizers | `ActiveInteractor::Organizer::Base` |
| `DeferredInteractor` for steps | `ActiveInteractor::Base` |
| Context classes with `input_attributes`/`output_attributes` | Untyped context bags |
| `after_success` for side effects with deferment | Side effects inside `perform` |
| `context.fail!(errors)` to halt | Raise exceptions for business logic |
| Check `result.success?` in controller | Assume organizer succeeded |
| Private methods for simple primary record creation | Separate interactor for trivial operations |

## Common Mistakes

1. **Putting Interactor/Organizer in the name** — Creates coupling and forces renames when implementation changes
2. **Using plural namespace for single-resource operations** — `Goal::Create` not `Goals::Create` for creating one goal
3. **Fat interactors** — If an interactor does more than one thing, split it
4. **Side effects in interactors** — Notifications, emails, analytics go in organizer `after_success` hooks with deferment
5. **Missing context class** — New organizers must define input/output contracts
6. **Not using TransactionOrganizer** — Multi-step writes must be transactional
7. **Inheriting from ActiveInteractor base classes** — Always use `DeferredInteractor`, `TransactionOrganizer`, or `BaseInteractor`/`BaseOrganizer`
8. **Not deferring irreversible side effects** — Analytics, emails, and async jobs must be deferred

**Remember:** Interactors are the logic layer. Models own data. Controllers delegate. Names describe the resource and action, not the implementation type.
