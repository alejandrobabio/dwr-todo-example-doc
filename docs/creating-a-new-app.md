# Creating a new app

I'll reproduce step by step the entire process of creating an app. Here I'll write the working code, but if you want to see the tests they are in the [GitHub](https://github.com/alejandrobabio/drw-todo-example) project with each commit.

I'll create a ToDo app that allows users sign up, log in, log out, add tasks, see owned tasks and mark as completed. Besides these main actions, It needs some support of other functionalities as authentication, authorization, and showing to each user the data that she/he is allowed to see. Last version of the app is currently deployed [here](https://dwr-todo-example.herokuapp.com).

## Install gems and create the app

```
$ gem install bundler
$ gem install dry-web-roda
$ dry-web-roda new todo
$ cd todo
$ bundle install
```

## Database

Create databases `todo_development` & `todo_test`

```
$ rake db:create
$ RACK_ENV=test rake db:create
```

Try the server for the first time
```
$ shotgun -p 3000 -o 0.0.0.0 config.ru
```
And when you visit http://localhost:3000 in your browser, you should see **"Welcome to dry-web-roda!"**

