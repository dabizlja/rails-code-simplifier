# Rails Code Simplifier

A Claude Code plugin that simplifies Ruby on Rails code following the rails coding philosophy.

## Philosophy

Based on the rails best practices, this plugin enforces:

- **Rich domain models over service objects**
- **Thin controllers, rich models, composable concerns**
- **Everything is CRUD** - map operations to standard REST resources
- **State as records, not booleans** - track who/when with dedicated models
- **Database-backed everything** - avoid Redis dependencies
- **Vanilla Rails is plenty** - maximize built-in framework capabilities

## Installation

### Manual Installation

```bash
# Clone the repo from github: https://github.com/dabizlja/rails-code-simplifier
claude --plugin-dir path_to_plugin
```

## Usage

Ask Claude to simplify your Rails code:

- "Simplify this controller"
- "Refactor this using rich models"
- "Clean up this Rails code"

## What It Does

### Converts Service Objects → Rich Models

```ruby
# Before: Service object
Orders::CreateService.call(params, user)

# After: Rich model
@order.place!
```

### Converts Custom Actions → CRUD Resources

```ruby
# Before: POST /cards/:id/close
def close
  @card.update!(closed: true)
end

# After: POST /cards/:id/closure
class Cards::ClosuresController
  def create
    @card.close!
  end
end
```

### Converts Boolean State → State Records

```ruby
# Before: closed: boolean
card.update!(closed: true)

# After: has_one :closure
card.close!(by: Current.user)
# Now you know WHO closed it and WHEN
```

### Extracts Shared Behavior → Concerns

```ruby
# Before: Repeated code across models
# After: Composable concerns
class Card < ApplicationRecord
  include Closeable
  include Watchable
  include Assignable
end
```

## What It Avoids

- Service objects → Use rich models
- Decorators/Presenters → Keep methods on models
- Redis/Sidekiq → Use Solid Queue
- ViewComponent → Use ERB partials
- React/Vue/SPA → Use Hotwire

## License

MIT
