# The dry-web-roda framework

[Dry-web-roda](https://github.com/dry-rb/dry-web-roda) is a super slim ruby framework for web apps. It is supported by the [Roda](http://roda.jeremyevans.net) routing tree and its plugins, the [dry-rb](http://dry-rb.org) gems and the [ROM](http://rom-rb.org) persistence library. It puts all these things together in a clean architecture and adds few methods and plugins that make the task of building apps a joy. 


## Goals

* It's a flexible framework and you can organize your components to fit your own taste.
* It brings a set of gems that are built with an eye on quality and safety. You will find deliberate restriction to mutations in its objects, that are frozen and do not have `attr_writer` or `attr_accessor` calls in its classes. So, you can expect that the instantiated objects remain unchanged along all its life cycle. 
* The dependency inversion principle is the other goal of its gems, and in this framework, it is provided by default. You will build an app with components highly isolated following few and simple rules.
* It is really fast, it uses a dozen gems incredible lightweight. And you will see that running your tests. 
* You can take advantage of `dry-transaction` to define your business rules or use cases (transactions here) and it can be smoothly integrated into your application.

## Architecture

The framework provides two structures to generate your app: **umbrella** and **flat** projects. Here, we'll explore the first one with `dry-web-roda` version `0.9.0`.

The file structure for an umbrella application called `todo` is:
```
.
|-- apps
|   `-- main
|       |-- lib
|       |   `-- todo
|       |       `-- main
|       |           |-- view
|       |           |   |-- context.rb
|       |           |   `-- controller.rb
|       |           `-- views
|       |               `-- welcome.rb
|       |-- system
|       |   |-- boot.rb
|       |   `-- todo
|       |       `-- main
|       |           |-- container.rb
|       |           |-- import.rb
|       |           `-- web.rb
|       `-- web
|           |-- routes
|           |   `-- example.rb
|           `-- templates
|               |-- layouts
|               |   `-- application.html.slim
|               `-- welcome.html.slim
|-- bin
|   |-- console
|   `-- setup
|-- config.ru
|-- db
|   |-- sample_data.rb
|   `-- seed.rb
|-- Gemfile
|-- lib
|   |-- persistence
|   |   |-- commands
|   |   `-- relations
|   |-- todo
|   |   |-- operation.rb
|   |   |-- repository.rb
|   |   `-- view
|   |       |-- context.rb
|   |       `-- controller.rb
|   `-- types.rb
|-- log
|-- Rakefile
|-- README.md
|-- spec
|   |-- db_spec_helper.rb
|   |-- factories
|   |   `-- example.rb
|   |-- spec_helper.rb
|   |-- support
|   |   |-- db
|   |   |   |-- factory.rb
|   |   |   `-- helpers.rb
|   |   `-- web
|   |       `-- helpers.rb
|   `-- web_spec_helper.rb
`-- system
    |-- boot
    |   |-- monitor.rb
    |   |-- persistence.rb
    |   `-- settings.rb
    |-- boot.rb
    `-- todo
        |-- container.rb
        |-- import.rb
        `-- web.rb
```
As it has an architecture of [monolith first](https://martinfowler.com/bliki/MonolithFirst.html), it has a directory `apps` where you will isolate several apps that could be moved in the future without rewriting the entry the app. Usually, you could have an app for the web app (in `apps/main`) another for admin area, one for an API, and you can add the ones that you need.

Within each sub-app, you will find folders `web`, `system` and `lib`.

The `web` folder has the user interface, the output is in `templates`, and in `routes` lives the processing of the user request, which delegates actions to isolated classes that are in `lib` folder.

The `system` folder has the booting stuff of the app (or sub-app), the Web and Container classes are the main actors here. 

About the `views` folder, is for view template preparation.