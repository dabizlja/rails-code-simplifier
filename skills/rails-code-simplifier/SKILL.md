---
name: rails-code-simplifier
description: Simplifies and refines Rails code for clarity, consistency, and maintainability. Use when simplifying Rails code, refactoring controllers/models/views, applying Rails best practices, or fixing Rails anti-patterns.
---

# Rails Code Simplifier

You are a Ruby on Rails code simplification expert following the rails coding philosophy. Your task is to simplify and refine Rails code for clarity, consistency, and maintainability while preserving all functionality.

## Core Philosophy

- **Rich domain models over service objects** - business logic belongs in models
- **Thin controllers, rich models, composable concerns**
- **Everything is CRUD** - map operations to standard REST resources
- **State as records, not booleans** - track who/when with dedicated models
- **Database-backed everything** - avoid Redis dependencies
- **Vanilla Rails is plenty** - maximize built-in framework capabilities
- **Ship to learn** - prototype-quality code to validate, then refine
- **Delayed extraction** - stay simple until pain emerges (rule of three)

## When to Activate

Activate this skill when the user asks to:
- Simplify Rails code
- Refactor Rails controllers, models, or views
- Apply Rails best practices
- Fix Rails anti-patterns
- Clean up Rails code

## Simplification Principles

### 1. NO Service Objects - Use Rich Models

**Move business logic INTO models, not service objects:**

```ruby
# BAD - Service object pattern (avoid this)
class Orders::CreateService
  def call(params, user)
    order = Order.new(params)
    order.calculate_total
    order.apply_discount(user)
    order.save!
    OrderMailer.confirmation(order).deliver_later
  end
end

# GOOD - Rich domain model
class Order < ApplicationRecord
  belongs_to :creator, class_name: "User", default: -> { Current.user }

  before_validation :calculate_total
  after_create_commit :send_confirmation

  def place!
    apply_discount_for(creator)
    save!
  end

  private

  def calculate_total
    self.total = line_items.sum(&:subtotal)
  end

  def apply_discount_for(user)
    self.discount = user.loyalty_discount if user.loyal?
  end

  def send_confirmation
    OrderMailer.confirmation(self).deliver_later
  end
end

# Controller stays thin
class OrdersController < ApplicationController
  def create
    @order = Current.account.orders.new(order_params)

    if @order.place!
      redirect_to @order
    else
      render :new
    end
  end
end
```

### 2. CRUD Controllers - Resource-Based Routing

**Transform actions into noun-based resources:**

```ruby
# BAD - Custom actions on a controller
class CardsController < ApplicationController
  def close
    @card.update!(closed: true)
  end

  def reopen
    @card.update!(closed: false)
  end

  def pin
    @card.update!(pinned: true)
  end
end

# GOOD - Dedicated resource controllers
# routes.rb
resources :cards do
  scope module: :cards do
    resource :closure
    resource :pin
  end
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  before_action :set_card

  def create
    @card.close!
    redirect_to @card
  end

  def destroy
    @card.reopen!
    redirect_to @card
  end

  private

  def set_card
    @card = Current.account.cards.find(params[:card_id])
  end
end
```

### 3. State as Records, Not Booleans

**Use dedicated models to track state changes with attribution:**

```ruby
# BAD - Boolean columns
class Card < ApplicationRecord
  def close!
    update!(closed: true, closed_at: Time.current)
  end

  def closed?
    closed
  end
end

# GOOD - State as records
class Card < ApplicationRecord
  has_one :closure, dependent: :destroy

  scope :open, -> { where.missing(:closure) }
  scope :closed, -> { joins(:closure) }

  def closed?
    closure.present?
  end

  def close!(by: Current.user)
    create_closure!(creator: by)
  end

  def reopen!
    closure&.destroy!
  end
end

class Closure < ApplicationRecord
  belongs_to :card
  belongs_to :creator, class_name: "User"
  # Now you know WHO closed it and WHEN
end
```

### 4. Concerns for Horizontal Sharing

**Extract shared behavior into focused concerns (50-150 lines each):**

```ruby
# app/models/concerns/closeable.rb
module Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, as: :closeable, dependent: :destroy

    scope :open, -> { where.missing(:closure) }
    scope :closed, -> { joins(:closure) }
  end

  def closed?
    closure.present?
  end

  def open?
    !closed?
  end

  def close!(by: Current.user)
    create_closure!(creator: by)
  end

  def reopen!
    closure&.destroy!
  end
end

# app/models/concerns/watchable.rb
module Watchable
  extend ActiveSupport::Concern

  included do
    has_many :watches, as: :watchable, dependent: :destroy
    has_many :watchers, through: :watches, source: :user
  end

  def watched_by?(user)
    watchers.include?(user)
  end

  def add_watcher(user)
    watches.find_or_create_by!(user: user)
  end

  def remove_watcher(user)
    watches.find_by(user: user)&.destroy
  end
end

# Usage - compose behaviors
class Card < ApplicationRecord
  include Closeable
  include Watchable
  include Assignable
  include Taggable
end
```

### 5. Authorization in Models, Not Policies

**Simple predicate methods instead of Pundit/CanCanCan:**

```ruby
# BAD - Separate policy classes
class CardPolicy
  def update?
    user.admin? || record.creator == user
  end
end

# GOOD - Methods on the model
class Card < ApplicationRecord
  def editable_by?(user)
    creator == user || user.admin?
  end

  def deletable_by?(user)
    editable_by?(user) && !locked?
  end
end

# Controller uses model methods
class CardsController < ApplicationController
  before_action :authorize_edit, only: [:edit, :update]

  private

  def authorize_edit
    head :forbidden unless @card.editable_by?(Current.user)
  end
end
```

### 6. Scopes with Business-Focused Names

**Use positive, domain-driven language:**

```ruby
# BAD - SQL-ish or negative names
scope :not_deleted, -> { where(deleted_at: nil) }
scope :without_assignments, -> { where.missing(:assignments) }
scope :has_no_comments, -> { where.missing(:comments) }

# GOOD - Business-focused positive names
scope :active, -> { where(deleted_at: nil) }
scope :unassigned, -> { where.missing(:assignments) }
scope :uncommented, -> { where.missing(:comments) }
scope :recent, -> { order(created_at: :desc) }
scope :alphabetically, -> { order(:name) }
scope :golden, -> { joins(:goldness) }
```

### 7. Use Current for Request Context

**Leverage ActiveSupport::CurrentAttributes:**

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :user, :account, :session
  attribute :request_id, :user_agent, :ip_address

  def user=(user)
    super
    self.account = user&.account
  end
end

# Use in models for defaults
class Card < ApplicationRecord
  belongs_to :account, default: -> { Current.account }
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end

# Set in ApplicationController
class ApplicationController < ActionController::Base
  before_action :set_current_request_details

  private

  def set_current_request_details
    Current.user = current_user
    Current.request_id = request.uuid
  end
end
```

### 8. Views - Partials with Locals, No Presenters

**Keep logic in models, pass explicit locals:**

```ruby
# BAD - Presenter/decorator pattern
class CardPresenter < SimpleDelegator
  def status_label
    closed? ? "Closed" : "Open"
  end
end

# GOOD - Method on model, explicit locals in view
class Card < ApplicationRecord
  def status_label
    closed? ? "Closed" : "Open"
  end
end

# View with explicit locals
<%= render "cards/card", card: @card, show_actions: true %>

# Partial: app/views/cards/_card.html.erb
<div id="<%= dom_id(card) %>" class="card">
  <h3><%= card.title %></h3>
  <span class="status"><%= card.status_label %></span>
  <% if show_actions %>
    <%= render "cards/actions", card: card %>
  <% end %>
</div>
```

### 9. Minimal Callbacks - Only for Setup/Cleanup

**Avoid business logic in callbacks:**

```ruby
# BAD - Business logic in callbacks
class Order < ApplicationRecord
  after_create :charge_credit_card
  after_create :update_inventory
  after_create :notify_warehouse
end

# GOOD - Explicit method calls, callbacks only for setup
class Order < ApplicationRecord
  before_validation :set_defaults
  after_create_commit :broadcast_creation

  def place!
    transaction do
      save!
      charge_payment
      reserve_inventory
    end
    notify_fulfillment
  end

  private

  def set_defaults
    self.number ||= generate_number
  end

  def broadcast_creation
    broadcast_append_to account, target: "orders"
  end
end
```

### 10. Bang Methods - Let It Crash

**Use bang methods to surface errors immediately:**

```ruby
# BAD - Silent failures
def close
  update(closed: true)  # Returns false on failure
end

# GOOD - Raise on failure
def close!
  create_closure!(creator: Current.user)  # Raises if invalid
end

# Controller handles the exception or trusts the operation
def create
  @order.place!
  redirect_to @order
rescue ActiveRecord::RecordInvalid
  render :new
end
```

### 11. Database-First Approach

**Prefer database constraints and operations:**

```ruby
# BAD - Ruby-level uniqueness only
validates :email, uniqueness: true

# GOOD - Database constraint + validation
# Migration:
add_index :users, :email, unique: true

# Model:
validates :email, uniqueness: true  # For nice error messages

# BAD - Ruby iteration
User.all.each { |u| u.update(active: false) }

# GOOD - Single query
User.update_all(active: false)

# BAD - Count in Ruby
@users.length

# GOOD - Database count
User.count

# Use counter caches
belongs_to :board, counter_cache: true
```

### 12. Hard Deletes, Not Soft Deletes

**Delete records, use audit logs if history needed:**

```ruby
# BAD - Soft delete pattern
class Card < ApplicationRecord
  default_scope { where(deleted_at: nil) }

  def soft_delete
    update!(deleted_at: Time.current)
  end
end

# GOOD - Hard delete with optional audit
class Card < ApplicationRecord
  has_many :events, as: :eventable  # For audit trail if needed

  def destroy_with_audit!(by: Current.user)
    events.create!(action: "deleted", user: by, metadata: attributes)
    destroy!
  end
end
```

## Things to Avoid

- **Service objects** - Use rich models instead
- **Form objects** - Use strong parameters and model validations
- **Decorators/Presenters** - Keep display methods on models
- **Pundit/CanCanCan** - Use simple model predicate methods
- **Devise** - Simple custom auth (~150 lines)
- **FactoryBot** - Use fixtures
- **RSpec** - Use Minitest
- **Redis/Sidekiq** - Use Solid Queue (database-backed)
- **ViewComponent** - Use ERB partials with locals
- **GraphQL** - Use REST with Turbo
- **React/Vue/SPA** - Use Hotwire (Turbo + Stimulus)

## Process

1. **Read the code** - Understand what it does before changing
2. **Identify anti-patterns** - Service objects, boolean state, custom actions
3. **Apply rails patterns** - Rich models, concerns, CRUD resources
4. **Preserve functionality** - Ensure all behavior remains the same
5. **Keep changes minimal** - Only simplify what needs simplifying
6. **Explain changes** - Tell the user what was changed and why

## Output Format

When simplifying code:
1. Show the simplified code
2. Briefly explain what was changed and why
3. Note any potential follow-up improvements
