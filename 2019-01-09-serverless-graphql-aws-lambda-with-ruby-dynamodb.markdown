---
layout: guide
title: "GraphQL on AWS Lambda with Ruby and DynamoDB"
subtitle: How to run GraphQL Ruby in AWS Lambda.
updated: 2019-01-19 02:32:00 +0530
comments: true
header-img: 'assets/img/guides/hooks.jpg'
permalink: /guides/:title
categories:
  - graphql
  - ruby
  - aws
  - serverless
---

<span class='display-5'>Serverless</span> and AWS Lambda [created a buzz](https://twitter.com/nitzanshapira/status/1068797180331274240) more than any other technology in AWS reInvent 2018.


Google Cloud has its own serverless offering - Cloud Functions.


IBM offers Cloud functions based on Apache OpenWhisk.


Microsoft has Azure functions. And there are many other serverless providers.

While the world is getting hot on serverless, lets deep dive into serverless and run a GraphQL server on it!

## What is serverless technology? ##

Serverless lets you architect your application without thinking about servers. Traditionally, a web server keeps running listening to incoming requests in a particular port. So there is a process thats running in the CPU somewhere in the cloud.

But when running on a serverless provider like AWS Lambda, a whole "container" gets created, only after it receives an incoming request or a trigger. So there is nothing that is running, which waits for a request. The code gets executed after the container is created. The internal implementation of how this happen, differs from provider to provider. Until the occurance of a trigger(in this case an incoming request from the user), there is no process that is listening to requests. A trigger can be anything, like an updation of a table, or an incoming request.

![FaaS providers](/assets/img/guides/faas-providers.jpg "FaaS providers")
<div class='small italic text-center'>FaaS Provider Landscape</div>

When there are more triggers, more "containers" gets created and this is scaled automatically by the provider.
There are some things to note here - one, there are no "servers" that is running; two, since there are no servers thats waiting for the requests, no resources are used and so no cost is incurred if there are no incoming requests.

[AWS Lambda](https://aws.amazon.com/lambda/) is one such serverless provider. Read about the AWS Lambda container lifecycle [here](https://aws.amazon.com/blogs/compute/container-reuse-in-lambda/). Amazon recently announced first class support for Ruby language in AWS Lambda.

In this guide, we will create and deploy a Ruby GraphQL end point in AWS Lambda, that can add and list books of a bookshelf. We are using the [serverless framework](https://serverless.com/framework/), a toolkit that makes building serverless applications easier.

![GraphQL Ruby in AWS Lambda](/assets/img/guides/lambda-ruby-graphql.jpg "GraphQL Ruby in AWS Lambda")

<div class='divider'>…</div>
## STEP 1: Setting things up

Since serverless framework is being used, you need to install the npm package for serverless.

```bash
npm install serverless -g
```

Create a directory named bookshelf and run the following command
```
cd bookshelf

serverless create --template aws-ruby
```
This will create a `serverless.yml` file and a `handler.rb` file.

Lets organize http handlers in a handlers directory; models in the models directory. Create these folders and move the `handler.rb` file to the handlers directory, renaming it to `test.rb`.

```
mkdir models handlers
mv handler.rb handlers/test.rb
```

### Getting AWS credentials

Login to AWS console, and from IAM service create a new user. For example, name the user - `sls-user01`. Enable programatic access. In the permissions page, select **Attach existing policies directly** and click on **Create policy**. Select the JSON tab, add the following JSON file you'll find in [this gist](https://gist.github.com/ServerlessBot/7618156b8671840a539f405dea2704c8).

Copy the Access Key ID and Secret access key from the final step of user creation.

Now export the AWS credentials to be used for deployment.

```
export AWS_ACCESS_KEY_ID=<your-key-here>

export AWS_SECRET_ACCESS_KEY=<your-secret-key-here>
```

Open the `serverless.yml` file created and add an http endpoint, to the handler, by changing the functions section(note the intendation).


```
# File: serverless.yml

#...
functions:
  hello:
    handler: handlers/test.hello
    events:
      - http:
          path: hello
          method: get
#...

```

Deploy time. Run `serverless deploy` from the bookshelf directory.


This will display a set of messages and finally show the service information, that got deployed successfully. From the service information, copy the endpoint and send a GET request to it from a http client like [Postman](https://www.getpostman.com/). You should get a successful message.

<div class='divider'>…</div>
## STEP 2: DynamoDB model

DynamoDB is a no-sql key-value and document database which is fully managed by AWS. Its highly performant and scalable and as per the AWS website -

*DynamoDB can handle more than 10 trillion requests per day and support peaks of more than 20 million requests per second.*

Lets store books data in DynamoDB as key-value pairs. It will have these attributes:

### Book Model
- title
- author
- ISBN

[`aws-record`](https://github.com/aws/aws-sdk-ruby-record) gem adds Ruby support for DynamoDB. Create a Gemfile, and bundle install, after adding the aws-record gem. Make sure that you are on Ruby 2.5 before doing bundle install, as AWS lambda currently supports only Ruby 2.5 runtime.

The book attributes are defined by constructs like `string_attr`, `integer_attr`. Create a `book.rb` file with these fields, in this case all strings. We are assuming that `isbn` will be the unique key, and we make it the hash key in the table.

```rb
# File: models/book.rb

require 'aws-record'

class Book
  include Aws::Record

  set_table_name ENV['DYNAMODB_TABLE']

  string_attr :isbn, hash_key: true
  string_attr :title
  string_attr :author
end

```
<div class='alert alert-info'>
<b>Please Note:</b> If you dont use Ruby version 2.5.x then AWS lambda will fail to recognize the gems and you will get an <i>"Internal server error"</i>.
</div>

### Defining the DynamoDB table for books

The config in serverless.yml file, needs to change for setting `DYNAMODB_TABLE` which is passed as an environment variable. Also the permissions to access the table. These needs to be added in the provider section.

```
# Changes to serverless.yml

provider:
  name: aws
  runtime: ruby2.5
  region: us-east-1
  environment:
    DYNAMODB_TABLE: books-${opt:stage, self:provider.stage}
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource: "arn:aws:dynamodb:${opt:region, self:provider.region}:*:table/${self:provider.environment.DYNAMODB_TABLE}"

```

### Telling AWS about the DynamoDB table

The table definition needs to be added in the resources section in the same `serverless.yml` file, indicating the hash key `isbn`.

```
# Changes to serverless.yml

resources:
  Resources:
    Users:
      Type: 'AWS::DynamoDB::Table'
      DeletionPolicy: Retain
      Properties:
        TableName: ${self:provider.environment.DYNAMODB_TABLE}
        AttributeDefinitions:
          -
            AttributeName: isbn
            AttributeType: S
        KeySchema:
          -
            AttributeName: isbn
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1

```
You can view the full serverless file changes [here]().

To check if everything works fine, change the test handler to load some books and retrieve a book by ISBN.

```rb
require 'json'
require_relative '../models/book'

def hello(event:, context:)
  Book.new(
    title: 'Educated',
    author: 'Tara Westover',
    isbn: '0525589983'
  ).save!

  Book.new(
    title: 'Sapiens',
    author: 'Yuval Noah Harari',
    isbn: '0062316110'
  ).save!

  book = Book.find(isbn: '0525589983')

  {
    statusCode: 200,
    body: JSON.generate(book.to_h)
  }
end

```
Run `bundle install --deployment` so that bundler saves the gems in the vendor folder, which needs to be uploaded to AWS Lambda for the function to work. Now, deploy the function with `sls deploy` and the function should be live. Send a request to the endpoint and you should get a successful response.

<div class='divider'>…</div>
## STEP 3: Adding GraphQL boilerplate files

With the Book model working fine, its time to add GraphQL related files.

Add the `graphql` gem (specify version 1.8.0 or above for the [class based api]()) and do a `bundle install --deployment`.

Create a the `graphql/types` folder and create a Book type in `book.rb` file. In the `graphql` folder, create a schema (`schema.rb`), root query (`query.rb`). The files look like these:

```rb
# File graphql/types/book.rb

require 'graphql'

modules Types
  class Book < GraphQL::Schema::Object
    description 'Resembles a Book'

    field :isbn, String, null: false
    field :title, String, null: false
    field :author, String, null: false
  end
end

```

In the root query, add the `books` field for the books query, that will return `Book.scan`. For aws-record, `Book.scan` will scan the DynamoDB table and return all records.

<div class='alert alert-info'>
<b>Note:</b> `scan` is an expensive DynamoDB operation and its recommended to not use it in production. For simplicity sake, this guide uses scan, instead of using other secondary indices.
</div>

```rb
# File graphql/query_type.rb

require 'graphql'
require_relative 'types/book'
require_relative '../models/book'

class QueryType < GraphQL::Schema::Object
  description "The query root of this schema"

  field :books, [Types::Book], null: false do
    description 'Get all books'
  end

  def books
    Book.scan
  end
end

```

The schema file will be like below, with the QueryType added to schema.

```rb
# File graphql/bookshelf_schema.rb

require 'graphql'
require_relative 'query_type'

class BookshelfSchema < GraphQL::Schema
  query QueryType
end
```

The schema definition is complete. Now you need to add a graphql handler that will be used to query this schema.

Create a new handler - `graphql.rb` in the handlers folder with the following contents.

```rb
# File handlers/graphql.rb

require 'json'

require_relative '../graphql/schema'

def execute(event:, context:)
  data = JSON.parse(event['body'])

  result = BookshelfSchema.execute(
    data['query'],
    variables: data['variables'],
    context: { current_user: nil },
  )

  {
    statusCode: 200,
    body: JSON.generate(result)
  }
end

```

In this handler, the GraphQL query and variables that are being sent by the user (obtained by parsing the `event['body']`) is passed the `execute()` function of the schema, which retuns a ruby hash of the results.

### Add the GraphQL endpoint to serverless.yml

Add the new function in `serverless.yml`, which maps to the POST path `/graphql`.

```
functions:
  hello:
    ...

  graphql:
    handler: handlers/graphql.execute
    events:
      - http:
          path: graphql
          method: post
```

View the full changes [here]().

Deploy with `serverless deploy` and we are ready to go. Note the endpoint created.

<div class='divider'>…</div>
## STEP 4: Querying Books

To the endpoint created send the following query in a [graphql client like GraphiQL]().


```
{
  books{
    isbn
    title
  }
}
```

You should be able to see the two books we loaded in the `test.hello` function.
![Book Scan GraphQL](/assets/img/guides/book-scan.png "List all books in GraphQL")

Lets modify the QueryType to allow querying a book by isbn.

```rb
# Changes to graphql/query_type.rb

require 'graphql'
require_relative 'types/book'
require_relative '../models/book'

class QueryType < GraphQL::Schema::Object
  # ...
  # ...

  field :book, Types::Book, null: true do
    description 'Get a book by isbn'
    argument :isbn, String, required: true
  end

  def book(isbn:)
    Book.find(isbn: isbn)
  end

  # ...
```

If you query by isbn like below, you will get the correct book.

![Find Book GraphQL](/assets/img/guides/book-by-isbn.png "Get book by ISBN")

<div class='divider'>…</div>
## STEP 5: Mutation to create a book

To create a book, you need to add `create_book` mutation in GraphQL. In the `graphql/mutations` folder add a new file `create_book.rb`, which will take all the three fields as arguments. The resolve function will get the arguments and save the book to DynamoDB. Returns the json fields `success` and `errors`.

```rb
# File graphql/mutations/create_book.rb

require 'graphql'
require_relative '../../models/book'

module Mutations
  class CreateBook < GraphQL::Schema::Mutation
    description 'Creates a Book'

    argument :title, String, required: true
    argument :isbn, String, required: true
    argument :author, String, required: true

    field :success, Boolean, null: false
    field :errors, [String], null: false

    def resolve(name:, bio:, twitter_handle:, talk_title:)

      book = Book.new(
        title: title,
        author: author,
        isbn: isbn
      )

      if book.save!
        {
          success: true,
          errors: []
        }
      else
        {
          success: false,
          errors: ['Cannot save the book']
        }
      end
    end
  end
end

```
Add a root mutation.

```rb
require 'graphql'
require_relative 'mutations/create_book'

class MutationType < GraphQL::Schema::Object
  description "The mutation root of this schema"

  field :createBook, mutation: Mutations::CreateBook
end

```

And then add it to the schema.

```rb
require 'graphql'
require_relative 'query_type'
require_relative 'mutation_type'

class BookshelfSchema < GraphQL::Schema
  query QueryType
  mutation MutationType
end

```

For more on [GraphQL mutations read this](https://medium.com/hash32/graphql-with-sinatra-ruby-part-2-mutations-d6903699af3e).

Deploy to AWS Lambda, and you can run the following mutation.

![GraphQL Mutation](/assets/img/guides/create-book-mutation.png "Create Book")


This will save the book in the table, and show a successful message.

That's it. You now have a fully functional GraphQL endpoint, running on serverless AWS Lambda!

<!-- The source code for this post is available at this repo - [https://github.com/hash32/serverless-graphql-example](https://github.com/HASH32/serverless-graphql-example). -->


## Credits & References

- Read the [GraphQL Ruby documentation]().
- Read my [previous post]() on creating a GraphQL server in Sinatra.
- [AWS-Record]() - an abstraction for Amazon DynamoDB.
- *Header Photo by [Ravi Sharma on Unsplash](https://unsplash.com/photos/GyhakN_lgec)*.


