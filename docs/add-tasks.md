# Add tasks

## Add tasks table

create tasks migrations (<a href="http://sequel.jeremyevans.net/rdoc/files/doc/schema_modification_rdoc.html" target="_blank">Migrations guide</a>)
```
$ rake db:create_migration[create_tasks]
```

Let's write the migration with `description`, `completed`, and foreign key `user_id` fields.

```ruby
ROM::SQL.migration do
  change do
    create_table(:tasks) do
      primary_key :id
      String :description
      Boolean :completed
      foreign_key :user_id, :users
      DateTime :created_at, null: false, default: Sequel::CURRENT_TIMESTAMP
      DateTime :updated_at, null: false, default: Sequel::CURRENT_TIMESTAMP
    end
  end
end
```
Run the migration for development and test environments
```
$ rake db:migrate
$ RACK_ENV=test rake db:migrate
```

## Add tasks relation and repository

`lib/persistence/relations/tasks.rb`
```ruby
# frozen_string_literal: true

module Persistence
  module Relations
    class Tasks < ROM::Relation[:sql]
      schema(:tasks, infer: true) do
        associations do
          belongs_to :user
        end
      end
    end
  end
end
```

`lib/todo/repositories/tasks_repo.rb`
```ruby
# frozen_string_literal: true

require 'todo/repository'

module Todo
  module Repositories
    class TasksRepo < Todo::Repository[:tasks]
      commands :create, update: :by_pk, delete: :by_pk
    end
  end
end
```