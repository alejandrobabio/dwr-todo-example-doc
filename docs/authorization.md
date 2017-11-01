# Authorization

## The big picture:

I want to authorize as soon as possible and redirect if he/she is not authorized to do current action.

Then the authorization class will need to know the user, the resource and the action. The user id is in session and the resource and the action are in the request. 

I will define an authorization class for each resource, that receives `current_user` and `request` and responds with something that can be evaluated as `true` if the user is authorized to do this action, otherwise as `false`. Each authorization class will hold the access policy to its resource.

## Defining authorizations:

If only logged in users can access to tasks. Then  authorization for tasks could be:

`apps/main/lib/todo/main/authorizations/tasks.rb`

```ruby
# frozen_string_literal: true

module Todo
  module Main
    module Authorizations
      class Tasks
        def call(user, _request)
          user
        end
      end
    end
  end
end
```

It will be needed a mechanism that calls the right `Todo::Main::Authorizations::<Class>` depending on the resource required. Then in `apps/main/lib/todo/main/authorization.rb` I put:

```ruby
# frozen_string_literal: true

module Todo
  module Main
    class Authorization
      def call(user, request)
        name = Inflecto.camelize(
          request.path.split('/').reject(&:empty?).first
        )
        Object.const_get("Todo::Main::Authorizations::#{name}").new.(user, request)
      rescue NameError
        false
      end
    end
  end
end
```

## Putting all together

Update `apps/main/system/todo/main/web.rb` with:

* Define some public routes in application's config
* Add an instance method to access config
* Add an `authorize` method
* Redirect to root if authorize returns false

The call to the `authorize` method should be before `r.multi_route` call, the best place should be at the beginning of the `route` block.

```ruby
module Todo
  module Main
    class Web < Dry::Web::Roda::Application
      ...
      setting :public_routes
      configure do |config|
        ...
        config.public_routes = %w[/ /login /logout /signup]
      end
      ...
      route do |r|
        unless authorize
          flash[:alert] = 'Unauthorized!'
          r.redirect '/'
        end
        
        r.multi_route
        ...
      end
      ...
      private
    
      def config
        self.class.config
      end

      def authorize
        return true if config.public_routes.include? request.path
        self.class['authorization'].(current_user, request)
      end
    end
  end
end
```
