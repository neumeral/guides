---
layout: guide
title: "GraphQL with Sinatra"
subtitle: How to create a GraphQL server using the Sinatra framework in Ruby.
updated: 2019-01-09 02:32:00 +0530
comments: true
header-img: 'assets/img/guides/nodes-graph.jpg'
permalink: /guides/:title
categories:
  - graphql
  - ruby
---

<span class='display-5'>GraphQL</span> has redefined the way we deal with APIs and endpoints. With the client being able to specify the fields it needs, development in the client could in one way be independent of the server changes, if a schema is predefined.

<!--more-->

![GraphQL + Sinatra](/assets/img/guides/sinatra-graphql.jpg)

We’ll see how to create a GraphQL server using the Sinatra framework in Ruby. In this example we will be creating the schema for a conference app, where you can add speakers and list them.

<div class='divider'>…</div>
## STEP 1: Create a Sinatra application

I wish there was a template to create a sinatra application using a cli. But there isn’t a lot of boilerplate files to create, so lets add it one-by-one.

We’ll be using `puma` as the server. Create an `app.rb` file which defines a basic sinatra app with a `/` route. Also a `Gemfile` is added and bundle install is run.

<script src="https://gist.github.com/hash32bot/3097ed9f44f1b88be5c1968be3949a8d.js"></script>

Next we need a rackup file config.ru so that puma will pickup the app as a rack application.

<script src="https://gist.github.com/hash32bot/1ccf48bf29df129f986bd14b3e1ac7e4.js"></script>

Running the puma server will serve your application at [http://localhost:9292](http://localhost:9292), and you should see the message ‘It Works!’.

```
bundle exec puma
```
Yay! The app is up.

<div class='divider'>…</div>
## STEP 2: Add JSON responses
For the server to respond to JSON, we add the [sinatra-contrib](https://github.com/sinatra/sinatra/tree/master/sinatra-contrib) gem, which adds a JSON helper. Change the `app.rb` file to respond to json.

<script src="https://gist.github.com/hash32bot/401ff194d8c4b8078f7a8a865dc1f0ba.js"></script>

Now our app contains just these files:

```
conference_app
  |
  ├── Gemfile
  ├── Gemfile.lock
  ├── app.rb
  └── config.ru

```

<div class='divider'>…</div>
## STEP 3: Add database connections and models with ActiveRecord

For talking to the database, we’ll use `activerecord` gem.

#### Add database configuration to connect to sqlite3

Also add a configuration file `database.yml` with the connection details and the `sqlite3` gem for connecting to the sqlite database. `app.rb` needs changes to update this configuration.

<script src="https://gist.github.com/hash32bot/c8e0e97c1503050790f0fcdcc23aa7cf.js"></script>

#### Add Rakefile

Add the `rake` gem along with the `Rakefile`. This gives handy rake tasks for creating the table (migrations) and managing them.
<script src="https://gist.github.com/hash32bot/4a0d347e74eef66cc7cf20cacf8b3dde.js"></script>

`bundle exec rake -T` will display the added rake tasks.

Create the sqlite database, by running `bundle exec rake db:create`.

#### Add a migration and model for Speaker object

Create a migration with the following rake command:

```
bundle exec rake db:create_migration NAME=create_speakers
```

Change the created migration file in `db/migrate` folder, to add the required database fields.

<script src="https://gist.github.com/hash32bot/9269780de679f9f2396216efca2c78f3.js"></script>

Run migrations with the rake task `bundle exec rake db:migrate`

Create a model file for **Speaker**, to access this table.

<script src="https://gist.github.com/hash32bot/c2ca8762028badd66df22effbda590a9.js"></script>

I’ve added a basic validation for the model. Read more on activerecord basics in the [official basics introduction](http://guides.rubyonrails.org/active_record_basics.html).


Add the `pry` gem for debugging and execute the following two statements in the pry console, for adding rows to the `speakers` table.

![Seed Speakers](/assets/img/guides/seed-speakers.png)

```rb
require './app'

Speaker.create(name: 'John', twitter_handle: 'johnruby',
  bio: 'This is John\'s bio', talk_title: 'How to bootstrap a sinatra application')

Speaker.create(name: 'Jacob', twitter_handle: 'jacob-ruby',
  bio: 'This is Jacob\'s bio', talk_title: 'Introduction to graphql')
```

#### Add a `/speakers` endpoint

Create a new endpoint to show the list of speakers, as JSON.

<script src="https://gist.github.com/hash32bot/4cc2ae8ae27213115a43fe741ec0304d.js"></script>

<div class='divider'>…</div>
## STEP 4: Add graphql and define a query to list speakers

Now we have a sinatra app that connects to the database and shows a list of speakers as a JSON response. Now let’s add graphql and define a schema for speakers.

Add the `graphql` gem. [https://github.com/rmosolgo/graphql-ruby](https://github.com/rmosolgo/graphql-ruby).

Also the `rack-contrib` gem needs to be added so that the sinatra app can accept raw JSON payloads.

#### Add type, query and schema for graphql

Now we need to add a type for **Speaker**, also a query and a schema for GraphQL.

<script src="https://gist.github.com/hash32bot/6bb06aba0f56fa74a47f8cbf5249c329.js"></script>

We need to then add a root query.

<script src="https://gist.github.com/hash32bot/88b8bb5a79481dd55f677e5fce3d892e.js"></script>

Define a schema for GraphQL.

<script src="https://gist.github.com/hash32bot/ef62dba77b26a29fd937ea86156949d9.js"></script>

#### The `/graphql` endpoint

We now need to have a POST endpoint for GraphQL.

GraphQL schema can be executed to give a `GraphQL::Query::Result` which can then be converted to JSON. `app.rb` needs change to include this endpoint.

<script src="https://gist.github.com/hash32bot/c0debd10ebdd695f806ec9455b477374.js"></script>

#### Querying the endpoint

You can use the [GraphiQL](https://github.com/skevy/graphiql-app/releases) app or the [Postman app](https://www.getpostman.com/) to query the endpoint. Make sure that you have puma running and the server is up.

![Postman Query Graphql](/assets/img/guides/gql-postman-success.png)

A JSON response like the below will be obtained.

<script src="https://gist.github.com/hash32bot/f8ca57738dbe3a7233f2f60001f5a42d.js"></script>

You have a GraphQL server up and running on sinatra, and you can query the endpoint to get a list of speakers with the fields defined in the GraphQL query.

Now let us add mutations.

## Mutations

A mutation is something that "mutates" or changes the data in the server. In DB terms, if we need to change the data in a table using graphql we need mutations — be it an INSERT, UPDATE or DELETE. Only SELECTs are covered with a `Query`.

So to add a new speaker to the database we need a mutation.

![Graphql Mutations](/assets/img/guides/gql-mutations.jpg)

In the GraphQL language, a mutation is of the form

```
mutation AddSpeaker($name:String, $talkTitle:String) {
  createSpeaker(name: $name, talkTitle:$talkTitle) {
    success
    errors
  }
}
```
A set of “query” variables needs to be supplied to the GraphQL endpoint.
Say for example,

```
{
  "name": "John Doe",
  "talkTitle": "Introduction to GraphQL in Ruby"
}

```

Read more about GraphQL mutations and its syntax in the specifications — [https://graphql.org/learn/queries/#mutations](https://graphql.org/learn/queries/#mutations).

For our little server to accept mutations, we need to make some changes and add more files for defining mutations. Lets see how, step-by-step.

<div class='divider'>…</div>
## STEP 5: Adding a Mutation root type

A mutation root `MutationType` has to be created and it should then be added to our Schema, like the `QueryType` that was added in the last post.

<script src="https://gist.github.com/hash32bot/b7c6278cffd24908205e4c23a61d06f5.js"></script>

<div class='divider'>…</div>
## STEP 6: Define a Mutation for speaker creation

Next, we need to tell GraphQL about the parameters that needs to be accepted for creating a new speaker.

Let’s split this into a separate file, that handles this mutation — `create_speaker.rb`. Instead of inheriting from `GraphQL::Schema::Mutation` we create a mutation base class `Mutations::BaseMutation`. Also group all the mutations in `mutations` folder.

<script src="https://gist.github.com/hash32bot/c32c19a31ce53527db8fb35941a03956.js"></script>

It needs to accept all the fields for a speaker, which we created as strings in the DB. In GraphQL ruby, strings are represented with the `String` type, as defined in the gem.

<script src="https://gist.github.com/hash32bot/86bba2aed6fb10c0abd0a40c6a232318.js"></script>

Next we need to take these fields and then call `speaker.save` with the defined input fields in the `resolve` function.
<script src="https://gist.github.com/hash32bot/bbd8f6a5de3c5bbed163882c0242811b.js"></script>

This returns a hash with success and errors. We need to tell GraphQL about it as well. **Note**: `errors` is an array of Strings. We define these as "fields" in the mutation.


<script src="https://gist.github.com/hash32bot/c6e1b059827368c59b302f17c3e9e6ff.js"></script>
Now the `CreateSpeaker` mutation is complete. It needs to be added to the root mutation — `MutationType` so that it gets included in the schema.

<script src="https://gist.github.com/hash32bot/1b9818e4a18a80f34c7aa139a3d14545.js"></script>

Restart the server with `bundle exec puma`.

If you use a client like [GraphiQL](https://github.com/skevy/graphiql-app/releases), you should be able to see the docs in the right sidebar changes and now has a `Mutation`.

Add a new speaker, with the mutation and the query variables in the client.

<script src="https://gist.github.com/hash32bot/4aee7ac41f952a29af2544770a5b1d7c.js"></script>

Execute the mutation in the GraphiQL client. You should be able to see the response data json, with something like below:

![GraphiQL Mutation Execution](/assets/img/guides/graphiql-mutation-exec.png)

The `CreateSpeaker` mutation has all the fields optional, but the Speaker model validates the presence of the name field. If you try to create a speaker without giving a `name` field, it will show up in `errors` return field.

We can avoid this validation at the schema level, by making the required `argument` mandatory. You just need to change `null: false` to the respective arguments.

<script src="https://gist.github.com/hash32bot/126adb71ed7c25cd5171dc13148ecd8c.js"></script>

Now the `name` and `talk_title` fields are non-nullable fields, and you’ll have to always give these fields when executing the mutation. Read about the type system in the [official documentation](https://graphql.org/learn/schema/#type-system).


## Show me the code
You can see all the code for this post in the [**sinatra-graphql**](https://github.com/awinabi/sinatra-graphql) Github repository.
