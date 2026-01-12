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

## Database Best Practices

### 13. UUIDs as Primary Keys

**Use UUIDs (preferably UUIDv7) instead of auto-incrementing integers:**

```ruby
# Migration setup
class EnableUuids < ActiveRecord::Migration[7.1]
  def change
    enable_extension 'pgcrypto' unless extension_enabled?('pgcrypto')
  end
end

# BAD - Auto-incrementing IDs (vulnerable to enumeration)
class CreateCards < ActiveRecord::Migration[7.1]
  def change
    create_table :cards do |t|
      t.string :title
      t.timestamps
    end
  end
end

# GOOD - UUID primary keys
class CreateCards < ActiveRecord::Migration[7.1]
  def change
    create_table :cards, id: :uuid do |t|
      t.string :title
      t.timestamps
    end
  end
end

# Configure as default in application.rb
config.generators do |g|
  g.orm :active_record, primary_key_type: :uuid
end
```

**Benefits:**
- Prevents ID enumeration attacks (no guessing `/cards/123`, `/cards/124`)
- Safe for distributed systems
- Clients can generate IDs before insertion
- Facilitates merging across databases

### 14. Account ID Everywhere (Multi-Tenancy)

**Include `account_id` as a foreign key across all tenant-scoped tables:**

```ruby
# Migration
class CreateCards < ActiveRecord::Migration[7.1]
  def change
    create_table :cards, id: :uuid do |t|
      t.references :account, null: false, foreign_key: true, type: :uuid
      t.string :title
      t.timestamps
    end
  end
end

# BAD - No tenant scoping
class Card < ApplicationRecord
  validates :number, uniqueness: true
end

# GOOD - Scoped to account
class Card < ApplicationRecord
  belongs_to :account, default: -> { Current.account }

  validates :number, uniqueness: { scope: :account_id }

  # All queries go through account
  default_scope { where(account: Current.account) if Current.account }
end

# ApplicationRecord can enforce this
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  def self.inherited(subclass)
    super
    # Auto-scope to account if column exists
    subclass.class_eval do
      default_scope { where(account_id: Current.account.id) if Current.account && column_names.include?('account_id') }
    end
  end
end
```

### 15. Database-Backed Infrastructure (No Redis)

**Use Solid Queue, Solid Cache, and Solid Cable instead of Redis:**

```ruby
# Gemfile - Database-backed infrastructure
gem "solid_queue"   # Background jobs
gem "solid_cache"   # Caching
gem "solid_cable"   # WebSockets

# config/environments/production.rb
config.active_job.queue_adapter = :solid_queue
config.cache_store = :solid_cache_store
config.action_cable.adapter = :solid_cable

# BAD - Redis dependencies
gem "sidekiq"
gem "redis"
config.active_job.queue_adapter = :sidekiq
config.cache_store = :redis_cache_store

# GOOD - Database-backed (simpler ops, unified backup/restore)
gem "solid_queue"
config.active_job.queue_adapter = :solid_queue
```

**Benefits:**
- Reduces operational dependencies
- Unified backup and restore procedures
- Simpler ops for small-to-medium scale
- SQLite support in development

### 16. Counter Caches

**Denormalize counts on parent records:**

```ruby
# Migration
class AddCardsCountToBoards < ActiveRecord::Migration[7.1]
  def change
    add_column :boards, :cards_count, :integer, default: 0, null: false

    # Backfill existing counts
    Board.find_each do |board|
      Board.reset_counters(board.id, :cards)
    end
  end
end

# BAD - Expensive COUNT query on every page load
class Board < ApplicationRecord
  has_many :cards

  def cards_count
    cards.count  # N+1 or expensive query
  end
end

# GOOD - Denormalized count
class Board < ApplicationRecord
  has_many :cards
  # cards_count column is automatically maintained
end

class Card < ApplicationRecord
  belongs_to :board, counter_cache: true
end

# Usage - no query needed
@board.cards_count  # Reads from column
```

### 17. Minimal Foreign Key Constraints

**Use foreign keys strategically, not everywhere:**

```ruby
# BAD - Foreign keys on everything (inflexible)
class CreateCards < ActiveRecord::Migration[7.1]
  def change
    create_table :cards, id: :uuid do |t|
      t.references :account, foreign_key: true
      t.references :board, foreign_key: true
      t.references :creator, foreign_key: { to_table: :users }
      t.references :assignee, foreign_key: { to_table: :users }
    end
  end
end

# GOOD - Foreign keys only for critical integrity (account)
class CreateCards < ActiveRecord::Migration[7.1]
  def change
    create_table :cards, id: :uuid do |t|
      t.references :account, null: false, foreign_key: true, type: :uuid
      t.references :board, null: false, type: :uuid  # No FK constraint
      t.references :creator, type: :uuid             # No FK constraint
      t.references :assignee, type: :uuid            # No FK constraint
    end
  end
end
```

**Rationale:** Application-level handling for most relationships, database integrity where it matters most (tenant isolation).

### 18. Index Strategy

**Index foreign keys and query columns, avoid over-indexing:**

```ruby
# Migration with proper indexing
class CreateCards < ActiveRecord::Migration[7.1]
  def change
    create_table :cards, id: :uuid do |t|
      t.references :account, null: false, foreign_key: true, type: :uuid, index: true
      t.references :board, null: false, type: :uuid, index: true
      t.string :status
      t.datetime :due_at
      t.timestamps
    end

    # Composite index for common query patterns
    add_index :cards, [:account_id, :status]
    add_index :cards, [:board_id, :created_at]

    # Index columns used in WHERE/ORDER BY
    add_index :cards, :due_at
  end
end

# Index rules:
# - Always index foreign keys
# - Index columns used in WHERE clauses
# - Index columns used in ORDER BY
# - Create composite indexes for common query patterns
# - Don't over-index (each index has write-performance cost)
```

### 19. Database Constraints Over Validations

**Enforce data integrity at the database level:**

```ruby
# BAD - Ruby-only validation (race condition vulnerable)
class Card < ApplicationRecord
  validates :title, presence: true
  validates :number, uniqueness: { scope: :account_id }
end

# GOOD - Database constraints + validations for error messages
class CreateCards < ActiveRecord::Migration[7.1]
  def change
    create_table :cards, id: :uuid do |t|
      t.string :title, null: false  # NOT NULL constraint
      t.integer :number, null: false
      t.references :account, null: false
    end

    # Unique constraint at database level
    add_index :cards, [:account_id, :number], unique: true
  end
end

class Card < ApplicationRecord
  validates :title, presence: true  # For nice error messages
  validates :number, uniqueness: { scope: :account_id }  # For nice error messages
end
```

## Routing Best Practices

### 20. Everything is CRUD - Verbs Become Nouns

**Transform action verbs into noun-based resources:**

```ruby
# BAD - Custom actions on resources
resources :cards do
  post :close
  post :reopen
  post :archive
  post :pin
  post :publish
end

# GOOD - Noun-based resources with standard CRUD
resources :cards do
  scope module: :cards do
    resource :closure      # POST to close, DELETE to reopen
    resource :pin          # POST to pin, DELETE to unpin
    resource :publication  # POST to publish, DELETE to unpublish
    resource :archive      # POST to archive, DELETE to unarchive
  end
end
```

**Verb to Noun Conversion Table:**

| Verb Action | Resource Name |
|-------------|---------------|
| Close a card | `card.closure` |
| Watch a board | `board.watching` |
| Pin an item | `item.pin` |
| Publish a board | `board.publication` |
| Assign a user | `card.assignment` |
| Mark as golden | `card.goldness` |
| Postpone | `card.not_now` |

### 21. Singular vs Plural Resources

**Use `resource` (singular) for one-per-parent relationships:**

```ruby
# BAD - Plural for single-instance relationships
resources :cards do
  resources :closures  # Only one closure per card
  resources :pins      # Only one pin per card
end

# GOOD - Singular for one-per-parent
resources :cards do
  resource :closure    # A card has exactly one closure state
  resource :watching   # A user's watch status per card
  resource :goldness   # A card is either golden or not
  resource :pin        # A card is pinned or not

  resources :comments  # A card has many comments (plural)
  resources :taggings  # A card has many taggings (plural)
end
```

Singular resources generate routes without `:id` parameter since only one instance exists per parent.

### 22. Module Scoping for Controller Organization

**Use `scope module:` to organize controllers without changing URLs:**

```ruby
# Using scope module (preserves URL structure)
resources :cards do
  scope module: :cards do
    resource :closure  # Cards::ClosuresController → /cards/:id/closure
    resource :pin      # Cards::PinsController → /cards/:id/pin
    resources :comments do
      scope module: :comments do
        resources :reactions  # Cards::Comments::ReactionsController
      end
    end
  end
end

# Directory structure:
# app/controllers/
# ├── cards_controller.rb
# ├── cards/
# │   ├── closures_controller.rb
# │   ├── pins_controller.rb
# │   └── comments/
# │       └── reactions_controller.rb
```

### 23. Shallow Nesting

**Prevent deeply nested URLs using `shallow: true`:**

```ruby
# BAD - Deeply nested URLs
resources :boards do
  resources :cards do
    resources :comments  # /boards/1/cards/2/comments/3
  end
end

# GOOD - Shallow nesting
resources :boards, shallow: true do
  resources :cards do
    resources :comments, shallow: true
  end
end

# Generated routes:
# /boards/:board_id/cards       (index, new, create)
# /cards/:id                    (show, edit, update, destroy)
# /cards/:card_id/comments      (index, new, create)
# /comments/:id                 (show, edit, update, destroy)
```

### 24. Use `resolve` for Polymorphic URLs

**Enable polymorphic URLs for nested resources:**

```ruby
# config/routes.rb
resolve "Comment" do |comment, options|
  options[:anchor] = ActionView::RecordIdentifier.dom_id(comment)
  route_for :card, comment.card, options
end

resolve "Notification" do |notification, options|
  polymorphic_url(notification.notifiable_target, options)
end

# Now you can use:
url_for(@comment)  # => /cards/123#comment_456
redirect_to @notification  # => Goes to the notifiable target
```

### 25. Path-Based Multi-Tenancy

**Include account context in URL path:**

```ruby
# config/routes.rb
scope "/:account_id" do
  resources :boards
  resources :cards
end

# Middleware extracts account_id and sets Current.account
# app/middleware/account_middleware.rb
class AccountMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    request = ActionDispatch::Request.new(env)
    if account_id = request.path_parameters[:account_id]
      Current.account = Account.find(account_id)
    end
    @app.call(env)
  end
end
```

### 26. Same Controller, Different Format (No Separate API)

**Use `respond_to` blocks instead of separate API controllers:**

```ruby
# BAD - Separate API controllers
# app/controllers/api/v1/cards/closures_controller.rb
module Api::V1
  class Cards::ClosuresController < ApiController
    def create
      @card.close!
      render json: @card, status: :created
    end
  end
end

# GOOD - Single controller, multiple formats
class Cards::ClosuresController < ApplicationController
  before_action :set_card

  def create
    @card.close!

    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.html { redirect_to @card }
      format.json { head :created, location: card_path(@card) }
    end
  end

  def destroy
    @card.reopen!

    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.html { redirect_to @card }
      format.json { head :no_content }
    end
  end

  private

  def set_card
    @card = Current.account.cards.find(params[:card_id])
  end

  def render_card_replacement
    render turbo_stream: turbo_stream.replace(@card)
  end
end
```

**Consistent Response Codes:**

| Operation | Code | Details |
|-----------|------|---------|
| Create | 201 Created | Include `Location` header |
| Update | 204 No Content | Empty response body |
| Delete | 204 No Content | Empty response body |

### 27. Real-World Routes Structure Example

**Complete nested resource example:**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  scope "/:account_id" do
    resources :boards do
      scope module: :boards do
        resource :publication
        resource :entropy
        resource :involvement

        namespace :columns do
          resource :not_now
          resource :stream
          resource :closed
        end
      end
    end

    resources :cards do
      scope module: :cards do
        resource :board        # Move card to different board
        resource :closure      # Close/reopen card
        resource :column       # Assign to workflow column
        resource :goldness     # Mark as important
        resource :image        # Manage header image
        resource :not_now      # Postpone card
        resource :pin          # Pin to sidebar
        resource :publish      # Publish draft
        resource :reading      # Mark as read
        resource :watch        # Subscribe to updates

        resources :assignments
        resources :steps
        resources :taggings
        resources :comments do
          scope module: :comments do
            resources :reactions
          end
        end
      end
    end
  end
end
```

## Filtering Best Practices

### 28. Filter Object Pattern (POROs, Not Controllers)

**Extract filtering logic from controllers into dedicated filter objects:**

```ruby
# BAD - Filter logic in controller
class CardsController < ApplicationController
  def index
    @cards = Current.account.cards
    @cards = @cards.where(status: params[:status]) if params[:status].present?
    @cards = @cards.where(board_id: params[:board_ids]) if params[:board_ids].present?
    @cards = @cards.tagged_with(params[:tag_ids]) if params[:tag_ids].present?
    @cards = @cards.assigned_to(params[:assignee_ids]) if params[:assignee_ids].present?
    @cards = @cards.order(params[:sort_by] || :created_at)
  end
end

# GOOD - Dedicated filter object
class CardsController < ApplicationController
  before_action :set_filter

  def index
    @cards = @filter.cards
  end

  private

  def set_filter
    @filter = Current.account.card_filter_from(filter_params)
  end

  def filter_params
    params.permit(:status, :sort_by, board_ids: [], tag_ids: [], assignee_ids: [])
  end
end

# app/models/account/card_filter.rb
class Account::CardFilter
  def initialize(account, params = {})
    @account = account
    @status = params[:status]
    @sort_by = params[:sort_by]
    @board_ids = params[:board_ids]
    @tag_ids = params[:tag_ids]
    @assignee_ids = params[:assignee_ids]
  end

  def cards
    @cards ||= begin
      result = account.cards
      result = result.where(status: status) if status.present?
      result = result.where(board_id: board_ids) if board_ids.present?
      result = result.tagged_with(tags) if tags.present?
      result = result.assigned_to(assignees) if assignees.present?
      result = result.order(sort_by || :created_at)
      result
    end
  end

  def tags
    @tags ||= account.tags.where(id: tag_ids) if tag_ids.present?
  end

  def assignees
    @assignees ||= account.users.where(id: assignee_ids) if assignee_ids.present?
  end

  private

  attr_reader :account, :status, :sort_by, :board_ids, :tag_ids, :assignee_ids
end

# Add factory method to Account
class Account < ApplicationRecord
  def card_filter_from(params)
    Account::CardFilter.new(self, params)
  end
end
```

**Benefits:**
- Testable in isolation without HTTP overhead
- Reusable across controllers, views, and background jobs
- Clear separation of concerns

### 29. Query Composition with Lazy Evaluation

**Build queries incrementally using memoization:**

```ruby
class Filter
  def cards
    @cards ||= begin
      result = creator.accessible_cards.preloaded
      result = result.where(status: status) if status.present?
      result = result.where(board_id: boards.ids) if boards.present?
      result = result.assigned_to(assignees.ids) if assignees.present?
      result = result.where(creator_id: creators.ids) if creators.present?
      result = result.tagged_with(tags.ids) if tags.present?
      result = result.where(created_at: creation_window) if creation_window
      result = result.open unless include_closed?

      # Use reduce for multiple similar filters (e.g., search terms)
      result = terms.reduce(result) do |r, term|
        r.mentioning(term, user: creator)
      end

      result.distinct  # Handle duplicates from joins
    end
  end
end
```

**Key techniques:**
- Memoization (`@cards ||=`) executes query only once
- Conditional application - add filters only when data exists
- `distinct` at the end handles join duplicates
- `reduce` for applying multiple similar filters

### 30. URL-Based Filter State (Stateless)

**Store all filter state in URL parameters for shareability:**

```ruby
# app/models/filter/params.rb
module Filter::Params
  extend ActiveSupport::Concern

  PERMITTED_PARAMS = [
    :status,
    :sort_by,
    :creation,
    board_ids: [],
    tag_ids: [],
    assignee_ids: [],
    creator_ids: [],
    terms: []
  ].freeze

  # Convert filter to URL params
  def as_params
    @as_params ||= {
      status: status,
      sort_by: sort_by,
      creation: creation,
      terms: terms,
      tag_ids: tags&.ids,
      board_ids: boards&.ids,
      assignee_ids: assignees&.ids,
      creator_ids: creators&.ids
    }.compact_blank.reject { |k, v| default_value?(k, v) }
  end

  # Remove a specific filter value from params
  def as_params_without(key, value)
    as_params.dup.tap do |params|
      if params[key].is_a?(Array)
        params[key] = params[key] - [value]
        params.delete(key) if params[key].empty?
      elsif params[key] == value
        params.delete(key)
      end
    end
  end
end

# Controller concern
module FilterScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_filter
  end

  private

  def set_filter
    @filter = Current.user.filters.from_params(filter_params)
  end

  def filter_params
    params.reverse_merge(Filter.default_values).permit(*Filter::PERMITTED_PARAMS)
  end
end
```

**Benefits:**
- Shareable filtered views via URL
- Bookmarkable filter combinations
- Back button works naturally
- No server-side session state needed

### 31. Filter Chips as Links (Not Forms)

**Render active filters as removable link chips:**

```ruby
# BAD - Form-based chips requiring JavaScript
def filter_chip_tag(text, name:, value:)
  tag.button class: "chip chip--removable",
    data: { action: "filter-form#removeFilter form#submit" } do
    concat hidden_field_tag(name, value, id: nil)
    concat tag.span(text)
    concat icon("x")
  end
end

# GOOD - Pure link chips, no JavaScript needed
def filter_chip_tag(text, params)
  link_to cards_path(params), class: "chip chip--removable" do
    concat tag.span(text)
    concat icon("x")
  end
end

# View usage
<div class="filter-chips">
  <% @filter.tags.each do |tag| %>
    <%= filter_chip_tag tag.name, @filter.as_params_without(:tag_ids, tag.id) %>
  <% end %>

  <% @filter.assignees.each do |assignee| %>
    <%= filter_chip_tag "Assigned: #{assignee.name}",
        @filter.as_params_without(:assignee_ids, assignee.id) %>
  <% end %>

  <% if @filter.status.present? %>
    <%= filter_chip_tag "Status: #{@filter.status}",
        @filter.as_params_without(:status, @filter.status) %>
  <% end %>
</div>
```

**Benefits:**
- No JavaScript needed for removal
- Works with screen readers and keyboard navigation
- Progressive enhancement - works if JS fails
- Turbo Drive handles navigation automatically

### 32. Stimulus Controllers for Rich Filter UX

**Use separate controllers for filtering and keyboard navigation:**

```javascript
// app/javascript/controllers/filter_controller.js
import { Controller } from "@hotwired/stimulus"
import { debounce } from "helpers/timing_helpers"

export default class extends Controller {
  static targets = ["input", "item"]

  initialize() {
    this.filter = debounce(this.filter.bind(this), 100)
  }

  filter() {
    const query = this.inputTarget.value.toLowerCase()

    this.itemTargets.forEach(item => {
      const matches = item.textContent.toLowerCase().includes(query)
      item.toggleAttribute("hidden", !matches)
    })

    this.dispatch("changed")  // Notify other controllers
  }
}

// app/javascript/controllers/navigable_list_controller.js
export default class extends Controller {
  static targets = ["item"]
  static values = { actionableItems: { type: Boolean, default: false } }

  navigate(event) {
    switch(event.key) {
      case "ArrowDown": this.selectNext(event); break
      case "ArrowUp": this.selectPrevious(event); break
      case "Enter": this.activateCurrentItem(event); break
    }
  }

  selectNext(event) {
    const visible = this.visibleItems
    const index = visible.indexOf(this.currentItem)
    if (index < visible.length - 1) {
      this.setCurrentItem(visible[index + 1])
      event.preventDefault()
    }
  }

  selectPrevious(event) {
    const visible = this.visibleItems
    const index = visible.indexOf(this.currentItem)
    if (index > 0) {
      this.setCurrentItem(visible[index - 1])
      event.preventDefault()
    }
  }

  get visibleItems() {
    return this.itemTargets.filter(item => !item.hidden)
  }
}
```

```erb
<%# View integration %>
<dialog data-controller="filter navigable-list"
        data-action="keydown->navigable-list#navigate
                     filter:changed->navigable-list#reset">
  <%= text_field_tag :search, nil,
      placeholder: "Filter...",
      autofocus: true,
      data: { filter_target: "input", action: "input->filter#filter" } %>

  <ul>
    <% @tags.each do |tag| %>
      <li data-filter-target="item" data-navigable-list-target="item">
        <%= link_to tag.name, cards_path(tag_ids: [tag.id]) %>
      </li>
    <% end %>
  </ul>
</dialog>
```

### 33. Unit Test Filters (Not Integration Tests)

**Test filter objects directly for speed and isolation:**

```ruby
class Account::CardFilterTest < ActiveSupport::TestCase
  test "filters by multiple conditions" do
    filter = accounts(:acme).card_filter_from(
      creator_ids: [users(:david).id],
      tag_ids: [tags(:mobile).id]
    )

    assert_equal [cards(:layout)], filter.cards
  end

  test "filters unassigned cards" do
    filter = accounts(:acme).card_filter_from(
      status: "unassigned",
      board_ids: [boards(:product).id]
    )

    assert_equal [cards(:new_card)], filter.cards
  end

  test "respects access permissions" do
    boards(:private).update!(all_access: false)
    boards(:private).accesses.revoke_from(users(:david))

    filter = users(:david).filters.new(board_ids: [boards(:private).id])

    assert_empty filter.cards
  end

  test "converts to params excluding defaults" do
    filter = accounts(:acme).card_filter_from(
      sort_by: "latest",
      tag_ids: "",  # Empty should be excluded
      assignee_ids: [users(:jz).id]
    )

    expected = { assignee_ids: [users(:jz).id] }
    assert_equal expected, filter.as_params
  end

  test "removes single value from params" do
    filter = accounts(:acme).card_filter_from(
      assignee_ids: [users(:jz).id, users(:kevin).id]
    )

    result = filter.as_params_without(:assignee_ids, users(:jz).id)

    assert_equal({ assignee_ids: [users(:kevin).id] }, result)
  end
end
```

**Benefits:**
- 10-100x faster than controller/system tests
- Test filter logic independent of HTTP/routing
- Easy to test edge cases and combinations

### 34. Filter Persistence with Digest

**Save filters by generating a digest of normalized params:**

```ruby
module Filter::Persistence
  extend ActiveSupport::Concern

  class_methods do
    def find_by_params(params)
      find_by(params_digest: digest_params(params))
    end

    def digest_params(params)
      Digest::MD5.hexdigest(normalize_params(params).to_json)
    end

    def normalize_params(params)
      params
        .to_h
        .compact_blank
        .reject { |k, v| default_value?(k, v) }
        .transform_values { |v| v.is_a?(Array) ? v.map(&:to_s).sort : v.to_s }
        .sort.to_h
    end

    def remember(attrs)
      create!(attrs)
    rescue ActiveRecord::RecordNotUnique
      find_by_params(attrs).tap(&:touch)
    end
  end
end

# Usage
class Filter < ApplicationRecord
  include Filter::Persistence

  belongs_to :user

  # Finds or creates filter, updating timestamp if exists
  # filter = Current.user.filters.remember(tag_ids: [1, 2], status: "open")
end
```

**Benefits:**
- Prevents duplicate saved filters
- Order-independent (`tag_ids: [1, 2]` equals `tag_ids: [2, 1]`)
- Type-independent (`tag_ids: [1]` equals `tag_ids: ["1"]`)
- Enables saved searches, recent filters, filter analytics

## Performance Best Practices

### 35. N+1 Prevention with Preloaded Scopes

**Create preloaded scopes and use in-memory checks:**

```ruby
# BAD - N+1 queries
class CardsController < ApplicationController
  def index
    @cards = Current.account.cards  # Then views trigger N+1
  end
end

# GOOD - Preloaded scope
class Card < ApplicationRecord
  scope :preloaded, -> {
    includes(:column, :tags, :creator, :assignees, board: [:entropy, :columns])
  }
end

class CardsController < ApplicationController
  def index
    @cards = Current.account.cards.preloaded
  end
end

# BAD - Extra query when data is already loaded
def assigned_to?(user)
  assignments.exists?(assignee: user)  # Hits database
end

# GOOD - In-memory check
def assigned_to?(user)
  assignments.any? { |a| a.assignee_id == user.id }  # Uses preloaded data
end
```

**Use `prosopite` gem for N+1 detection in development/test.**

### 36. Batch SQL Over N+1 Loops

**Replace `find_each` loops with JOINs for bulk operations:**

```ruby
# BAD - N+1 loop
user.mentions.find_each do |mention|
  if mention.card&.collection_id == collection.id
    mention.destroy
  end
end

# GOOD - Single query with JOINs
user.mentions
  .joins("LEFT JOIN cards ON cards.id = mentions.mentionable_id
          AND mentions.mentionable_type = 'Card'")
  .joins("LEFT JOIN comments ON comments.id = mentions.mentionable_id
          AND mentions.mentionable_type = 'Comment'")
  .where("cards.collection_id = ? OR comments.card_id IN
          (SELECT id FROM cards WHERE collection_id = ?)",
         collection.id, collection.id)
  .destroy_all

# Accept "unmaintainable" SQL when performance requires it
# Speed takes priority over code elegance
```

### 37. Counter Caches for Fast Reads

**Use counter caches to avoid COUNT queries:**

```ruby
# BAD - COUNT query on every page load
def cards_count
  cards.count  # SELECT COUNT(*) FROM cards WHERE board_id = ?
end

# GOOD - Counter cache (instant read from column)
class Card < ApplicationRecord
  belongs_to :board, counter_cache: true
end

# Migration
add_column :boards, :cards_count, :integer, default: 0, null: false

# Now @board.cards_count reads from column, no query

# NOTE: Counter caches bypass callbacks
# If you need side effects, use manual approach:
class Card < ApplicationRecord
  after_create_commit :increment_board_count
  after_destroy_commit :decrement_board_count

  private

  def increment_board_count
    board.increment!(:cards_count)
  end

  def decrement_board_count
    board.decrement!(:cards_count)
  end
end
```

### 38. Lazy Loading with Turbo Frames

**Load expensive content on interaction, not page load:**

```ruby
# BAD - Load everything on page load
<%= render "cards/menu", card: @card %>  # Expensive menu rendered for every card

# GOOD - Lazy load with turbo frame
<%= turbo_frame_tag dom_id(@card, :menu), src: card_menu_path(@card), loading: :lazy do %>
  <%= render "cards/menu_placeholder" %>
<% end %>

# Or load on interaction (even better)
<button data-action="click->turbo-frame#load"
        data-turbo-frame-src-value="<%= card_menu_path(@card) %>">
  Open Menu
</button>

<turbo-frame id="<%= dom_id(@card, :menu) %>">
  <!-- Loaded on click -->
</turbo-frame>
```

**Benefits:**
- Significantly reduces initial render time
- Only loads what user actually needs
- Better perceived performance

### 39. Debouncing for Responsive UX

**Use 100ms debounce on filter/search inputs:**

```javascript
// app/javascript/controllers/filter_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "item"]

  initialize() {
    // 100ms debounce provides responsive feel
    this.filter = this.debounce(this.filter.bind(this), 100)
  }

  filter() {
    const query = this.inputTarget.value.toLowerCase()
    this.itemTargets.forEach(item => {
      item.toggleAttribute("hidden",
        !item.textContent.toLowerCase().includes(query))
    })
  }

  debounce(fn, delay) {
    let timeoutId
    return (...args) => {
      clearTimeout(timeoutId)
      timeoutId = setTimeout(() => fn(...args), delay)
    }
  }
}
```

### 40. Pagination Strategy

**Start with reasonable page sizes and adjust:**

```ruby
# Start with 25-50 items per page
class CardsController < ApplicationController
  def index
    @cards = @filter.cards.page(params[:page]).per(25)
  end
end

# Use "Load more" pattern with Turbo
<%= turbo_frame_tag "cards_page_#{@page}" do %>
  <% @cards.each do |card| %>
    <%= render card %>
  <% end %>

  <% if @cards.next_page %>
    <%= turbo_frame_tag "cards_page_#{@cards.next_page}",
        src: cards_path(page: @cards.next_page),
        loading: :lazy do %>
      <div class="loading">Loading more...</div>
    <% end %>
  <% end %>
<% end %>

# Or use intersection observer for infinite scroll
<div data-controller="infinite-scroll"
     data-infinite-scroll-url-value="<%= cards_path(page: @cards.next_page) %>">
</div>
```

**Guidelines:**
- Reduce page size if initial render is slow (50 → 25)
- Implement separate pagination per column/section
- Prefer "Load more" over traditional pagination for better UX

### 41. Active Storage Optimizations

**Optimize file uploads and serving:**

```ruby
# config/storage.yml - Extend signed URL expiry for slow uploads
amazon:
  service: S3
  # Cloudflare buffering can exceed default 5 minute timeout
  upload:
    expires_in: 48.hours

# Skip previews for large files to avoid timeouts
class Document < ApplicationRecord
  has_one_attached :file

  def previewable?
    file.blob.byte_size < 16.megabytes
  end
end

# Use preprocessed: true for read replicas (lazy generation fails on read-only)
has_one_attached :avatar do |attachable|
  attachable.variant :thumb, resize_to_limit: [100, 100], preprocessed: true
end

# Redirect to blob URL instead of streaming through Rails (faster)
class AvatarsController < ApplicationController
  def show
    redirect_to @user.avatar.url, allow_other_host: true
  end
end
```

### 42. Puma/Ruby Tuning

**Configure Puma for optimal performance:**

```ruby
# config/puma.rb
workers Concurrent.physical_processor_count
threads 1, 1  # Single thread per worker for simpler debugging

before_fork do
  # GC, compact, malloc_trim for Copy-on-Write optimization
  Process.warmup
end

# Use autotuner gem to collect data and suggest tuning
# gem 'autotuner'
```

### 43. CSS Performance

**Avoid complex selectors that cause browser issues:**

```css
/* BAD - Complex :has() selectors freeze Safari */
.container:has(.item:has(.nested:hover)) {
  /* Safari will freeze */
}

/* GOOD - Simpler selectors or JavaScript */
.container.has-hovered-nested {
  /* Add class via JS instead */
}

/* Remove unnecessary view-transition-name that cause navigation jank */
/* Only use when actually needed for view transitions */
```

### 44. Optimistic UI for Drag & Drop

**Insert items immediately, then sync with server:**

```javascript
// app/javascript/controllers/sortable_controller.js
import { Controller } from "@hotwired/stimulus"
import Sortable from "sortablejs"

export default class extends Controller {
  static values = { url: String }

  connect() {
    this.sortable = Sortable.create(this.element, {
      onEnd: this.onEnd.bind(this)
    })
  }

  onEnd(event) {
    // Item already visually moved (optimistic)
    // Now sync with server asynchronously
    const itemId = event.item.dataset.id
    const newPosition = event.newIndex

    fetch(this.urlValue, {
      method: "PATCH",
      headers: {
        "Content-Type": "application/json",
        "X-CSRF-Token": document.querySelector("[name='csrf-token']").content
      },
      body: JSON.stringify({ id: itemId, position: newPosition })
    }).catch(() => {
      // Revert on failure
      this.sortable.sort(this.originalOrder)
    })
  }
}
```

**Benefits:**
- Immediate visual feedback
- Better perceived performance
- Graceful degradation on failure

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
