# Add users

## Add users table

Add Users migration, in the console: `rake db:create_migration[create_users]` and add the `create_table` method call:

```ruby
ROM::SQL.migration do
  change do
    create_table(:users) do
      primary_key :id
      String :email
      String :password_digest
      DateTime :created_at, null: false, default: Sequel::CURRENT_TIMESTAMP
      DateTime :updated_at, null: false, default: Sequel::CURRENT_TIMESTAMP
    end
  end
end
```

Run the migrations for development and test environments: 
```
rake db:migrate
RACK_ENV=test rake db:migrate
```

## Add users repository

A basic repository for users should be added: `lib/todo/repositories/users_repo.rb`

```ruby
# frozen_string_literal: true

require 'todo/repository'

module Todo
  module Repositories
    class UsersRepo < Todo::Repository[:users]
      commands :create, update: :by_pk, delete: :by_pk
    end
  end
end
```


## Add users relation

With [ROM](http://rom-rb.org) version 4, a repository expects its relation to be defined. Then in `lib/persistence/relations/users.rb` we put:

```ruby
# frozen_string_literal: true

module Persistence
  module Relations
    class Users < ROM::Relation[:sql]
      schema(:users, infer: true)
    end
  end
end
```