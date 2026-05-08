# Ruby on Rails Migration Guide for DSQL

How to run Rails applications against Aurora DSQL.

Sources:
- [Rails Sample](https://github.com/aws-samples/aurora-dsql-samples/tree/main/ruby/rails)
- [Ruby pg Driver Sample](https://github.com/aws-samples/aurora-dsql-samples/tree/main/ruby/ruby-pg)
- [Aurora DSQL Connectivity Tools](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/aws-sdks.html)
- [Rails with IAM Auth](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/SECTION_program-with-ruby-rails.html)

---

## 1. Dependencies

```ruby
# Gemfile
gem 'pg'                    # PostgreSQL adapter
gem 'aws-sdk-dsql'          # DSQL SDK for token generation
gem 'aws_rds_iam'           # IAM auth adapter for ActiveRecord (if available)
```

---

## 2. Database Configuration

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: 10
  host: <%= ENV['DSQL_ENDPOINT'] %>  # <cluster-id>.<region>.dsql.amazonaws.com
  port: 5432
  database: postgres                  # Always 'postgres'
  sslmode: require
  # No username/password here — use IAM token

development:
  <<: *default

production:
  <<: *default
  pool: 25
```

### IAM Token Generation

Create an initializer to generate tokens:

```ruby
# config/initializers/dsql_auth.rb
require 'aws-sdk-dsql'

module DsqlAuth
  def self.generate_token
    client = Aws::DSQL::Client.new(region: ENV['AWS_REGION'] || 'us-east-1')
    client.generate_db_connect_admin_auth_token(
      hostname: ENV['DSQL_ENDPOINT']
    )
  end
end

# Override ActiveRecord connection to inject IAM token
ActiveSupport.on_load(:active_record) do
  ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.class_eval do
    private

    alias_method :original_connect, :connect
    def connect
      @connection_parameters[:password] = DsqlAuth.generate_token
      @connection_parameters[:user] = 'admin'
      original_connect
    end
  end
end
```

**Alternative:** Use the `aws_rds_iam` gem if available for your Rails version, which handles this automatically.

---

## 3. Model Changes

### Use UUID Primary Keys

```ruby
# config/initializers/generators.rb
Rails.application.config.generators do |g|
  g.orm :active_record, primary_key_type: :uuid
end
```

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  # All models inherit UUID primary key
end
```

Migration for UUID support:

```ruby
# db/migrate/001_enable_uuid.rb
class EnableUuid < ActiveRecord::Migration[7.1]
  def change
    # gen_random_uuid() is built-in to DSQL — no extension needed
    # Just ensure migrations use uuid type
  end
end
```

### Remove Foreign Key Declarations

```ruby
# BAD: Rails will try to create FK constraint
class CreateTickets < ActiveRecord::Migration[7.1]
  def change
    create_table :tickets, id: :uuid do |t|
      t.references :organization, foreign_key: true  # ❌ FK will fail
      t.references :reporter, foreign_key: { to_table: :users }  # ❌
    end
  end
end

# GOOD: Plain columns without FK constraints
class CreateTickets < ActiveRecord::Migration[7.1]
  def change
    create_table :tickets, id: :uuid do |t|
      t.bigint :org_id, null: false
      t.uuid :reporter_id, null: false
      t.uuid :assignee_id
      t.string :title, limit: 500, null: false
      t.string :priority, limit: 20, default: 'medium'
      t.json :metadata
      t.boolean :is_resolved, default: false
      t.timestamps
    end
  end
end
```

### Model Associations Without FK Constraints

```ruby
# app/models/ticket.rb
class Ticket < ApplicationRecord
  # Use belongs_to WITHOUT database FK — Rails handles in-memory association
  belongs_to :organization, class_name: 'Organization', foreign_key: 'org_id',
             optional: false, inverse_of: :tickets

  belongs_to :reporter, class_name: 'User', foreign_key: 'reporter_id',
             optional: false

  belongs_to :assignee, class_name: 'User', foreign_key: 'assignee_id',
             optional: true

  # Application-layer FK validation
  validate :validate_foreign_keys

  private

  def validate_foreign_keys
    unless Organization.exists?(org_id)
      errors.add(:org_id, 'organization does not exist')
    end
    unless User.exists?(reporter_id)
      errors.add(:reporter_id, 'reporter does not exist')
    end
    if assignee_id.present? && !User.exists?(assignee_id)
      errors.add(:assignee_id, 'assignee does not exist')
    end
  end
end
```

**Key insight:** Rails' `belongs_to` works fine without a database FK constraint. It just does a SELECT to load the associated record. The FK constraint is only for database-level enforcement.

### ENUM Handling

```ruby
# PostgreSQL ENUM → string column with validation
class Ticket < ApplicationRecord
  PRIORITIES = %w[low medium high critical].freeze

  validates :priority, inclusion: { in: PRIORITIES }
end
```

Migration:

```ruby
# Use string column + CHECK constraint (not PostgreSQL ENUM type)
class CreateTickets < ActiveRecord::Migration[7.1]
  def change
    create_table :tickets, id: :uuid do |t|
      t.string :priority, limit: 20, default: 'medium'
    end

    # Add CHECK constraint (must be at CREATE TABLE time for DSQL)
    # Or add via raw SQL in same migration:
    execute "ALTER TABLE tickets ADD CONSTRAINT chk_priority CHECK (priority IN ('low','medium','high','critical'))"
  end
end
```

---

## 4. Migrations

### One DDL Per Migration

DSQL requires one DDL per transaction. Split complex migrations:

```ruby
# BAD: Multiple DDL in one migration
class CreateUsersAndIndexes < ActiveRecord::Migration[7.1]
  def change
    create_table :users, id: :uuid do |t|
      t.string :email, null: false
      t.timestamps
    end
    add_index :users, :email, unique: true  # Second DDL — will fail
  end
end

# GOOD: Separate migrations
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users, id: :uuid do |t|
      t.string :email, null: false
      t.timestamps
    end
  end
end

class AddUsersEmailIndex < ActiveRecord::Migration[7.1]
  def change
    # Use raw SQL for ASYNC index
    execute "CREATE UNIQUE INDEX ASYNC idx_users_email ON users (email)"
  end
end
```

### Disable Foreign Key Helpers

```ruby
# config/environments/production.rb (or all environments)
config.active_record.schema_format = :sql  # Use structure.sql, not schema.rb
```

In `config/application.rb`:

```ruby
# Prevent Rails from generating FK constraints
config.active_record.belongs_to_required_by_default = true  # Validation only, no FK
```

---

## 5. OCC Retry

### Concern for Retry Logic

```ruby
# app/models/concerns/occ_retryable.rb
module OccRetryable
  extend ActiveSupport::Concern

  class_methods do
    def with_occ_retry(max_retries: 5, &block)
      attempt = 0
      begin
        ActiveRecord::Base.transaction(&block)
      rescue ActiveRecord::SerializationFailure => e
        attempt += 1
        if attempt < max_retries
          delay = [0.05 * (2 ** attempt) + rand(0.0..0.05), 5.0].min
          sleep(delay)
          retry
        else
          raise
        end
      end
    end
  end
end
```

Usage:

```ruby
class TicketService
  include OccRetryable

  def self.create_ticket(params)
    with_occ_retry do
      ticket = Ticket.new(params)
      ticket.save!
      ticket
    end
  end
end
```

### Global Retry (Rack Middleware)

```ruby
# lib/middleware/dsql_retry_middleware.rb
class DsqlRetryMiddleware
  MAX_RETRIES = 3

  def initialize(app)
    @app = app
  end

  def call(env)
    attempt = 0
    begin
      @app.call(env)
    rescue ActiveRecord::SerializationFailure => e
      attempt += 1
      if attempt < MAX_RETRIES
        sleep([0.05 * (2 ** attempt), 2.0].min)
        retry
      else
        raise
      end
    end
  end
end

# config/application.rb
config.middleware.insert_before ActionDispatch::ShowExceptions, DsqlRetryMiddleware
```

---

## 6. Connection Pool Settings

```yaml
# config/database.yml
production:
  <<: *default
  pool: 25
  checkout_timeout: 5
  reaping_frequency: 30
  idle_timeout: 600        # Close idle connections after 10 min
  # DSQL connections timeout at 60 min — Rails pool handles reconnection
```

### Reconnection on Token Expiry

```ruby
# config/initializers/dsql_reconnect.rb
ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.class_eval do
  # Reconnect on authentication failure (expired token)
  def reconnect_with_fresh_token
    disconnect!
    connect  # Will call our overridden connect with fresh token
  end
end

# In application code, handle expired token:
ActiveSupport.on_load(:active_record) do
  ActiveRecord::Base.connection_pool.reaper.frequency = 30
end
```

---

## 7. Indexes

Use raw SQL for DSQL's async indexes:

```ruby
class AddTicketIndexes < ActiveRecord::Migration[7.1]
  def up
    execute "CREATE INDEX ASYNC idx_tickets_org ON tickets (org_id)"
    execute "CREATE INDEX ASYNC idx_tickets_reporter ON tickets (reporter_id)"
    execute "CREATE INDEX ASYNC idx_tickets_created ON tickets (created_at DESC)"
  end

  def down
    execute "DROP INDEX IF EXISTS idx_tickets_org"
    execute "DROP INDEX IF EXISTS idx_tickets_reporter"
    execute "DROP INDEX IF EXISTS idx_tickets_created"
  end
end
```

---

## 8. JSON Columns

Rails' `json` column type works directly with DSQL:

```ruby
# Migration
create_table :users, id: :uuid do |t|
  t.json :preferences, default: {}
end

# Model — query JSON fields
User.where("preferences->>'theme' = ?", 'dark')
User.where("preferences::jsonb @> ?", { theme: 'dark' }.to_json)
```

---

## 9. Things to Avoid

| Rails Feature | DSQL Issue | Alternative |
|---|---|---|
| `foreign_key: true` in migrations | FK constraints not supported | Remove; use model validations |
| `add_foreign_key` | Not supported | Skip |
| `t.references ... foreign_key: true` | Not supported | Use `t.bigint` / `t.uuid` |
| `dependent: :destroy` (with FK) | No cascade at DB level | Implement in `before_destroy` callback |
| `ActiveRecord::Enum` with PG enum | CREATE TYPE not supported | Use string + validates inclusion |
| `add_index` (standard) | Needs ASYNC | Use `execute "CREATE INDEX ASYNC..."` |
| `change_column` | ALTER COLUMN type not supported | Recreate table |
| `remove_column` | DROP COLUMN not supported | Recreate table |
| `TRUNCATE` (via `Model.delete_all`) | Not supported | Use `Model.delete_all` (generates DELETE FROM) |

---

## 10. Cascade Deletes (Without FK)

```ruby
# app/models/organization.rb
class Organization < ApplicationRecord
  has_many :tickets, foreign_key: 'org_id', dependent: :destroy
  has_many :users, foreign_key: 'org_id', dependent: :destroy

  # Before destroying, cascade manually
  before_destroy :cascade_resolve_tickets

  private

  def cascade_resolve_tickets
    # Mark tickets as resolved before org deletion
    Ticket.where(org_id: id, is_resolved: false).update_all(is_resolved: true)
  end
end
```

---

## 11. Checklist

- [ ] Add `aws-sdk-dsql` gem
- [ ] Configure IAM token generation in initializer
- [ ] Set `database: postgres` in database.yml
- [ ] Add `sslmode: require`
- [ ] Use `id: :uuid` for all tables
- [ ] Remove all `foreign_key: true` from migrations
- [ ] Add model-level FK validation (`validate` callbacks)
- [ ] Split migrations to one DDL per file
- [ ] Use `execute "CREATE INDEX ASYNC..."` for indexes
- [ ] Add OCC retry (concern or middleware)
- [ ] Set `schema_format = :sql` (use structure.sql)
- [ ] Replace `change_column` / `remove_column` with table recreation
- [ ] Test `dependent: :destroy` works via callbacks (not FK cascade)
- [ ] Set connection pool idle timeout < 60 min
