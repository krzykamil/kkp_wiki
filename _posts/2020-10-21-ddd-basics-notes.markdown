---
layout: post
title:  "DDD course basic notes!"
date:   2020-10-21 18:38:36 +0200
description: 'Taken at 25% of the course'
permalink: 'ddd_basic_notes_1'
author: kkp
---

# ReadModels and CQRS:

Base info
---------------------
  - (read models) Also called materialized views

# Event Sourcing

Base info
---------------------
  - persisting all the details about data, instead of just the newest version od data
  - it not only stores the data, you also store all the changes
  - ReadModels and WriteModels separate the ways you change, and read data
  - Main advantage is avoiding object rational mismatch (??? I could not understand what he was saying https://arkency.teachable.com/courses/537520/lectures/9769617)
  - In case of mistakes, you compensate for them, instead of fixing the data, using correct events.
  

# DDD

Base Info
---------------------
  - When presenting a problem, you should not immediately think about the solution cause you get caught in the solution domain.
  Pretty much never the first problem domain is a final one, and there is always more than one solution to a problem.
  - Tactical patterns: patterns used in DDD that build the Domain.
    * [Value Object](#value-object)
    * [Entity](#entity)
    * [Aggregate](#aggregate)
    * Domain Event
    * Repository Pattern, Factory Pattern etc.
  - Strategic patterns: those patterns in DDD are used to clear out and set concepts, communication and they define ubiquitous language
    * [Bounded Context](#bounded-context)
    * Context maps
    * Core and sub-domains
    
Tactical Pattern
---------------------
#### Value Object
  Its not a string, date, number, etc. its not a primitive type of data. They have no identity, they are identical to another VO
  if they have the same values. They are also immutable. It is usually a class that compounds attributes.

#### Entity
  Domain object with unique identity. It is usually mutable. Something like 'Product', 'Invoice' etc. ActiveRecord::Base is close to it somewhat.
  They can be described by Value Objects.

#### Aggregate
  Put simply it is just few entities together.
  - Order(entity) has multiple OrderLines(entity), so they create an Aggregate when put together.
  - Those ^ create an objects hierarchy on top of which we have aggregate root. It is usually provided with a conceptual 
  name of an aggregate.
  - When something changes in one part of the hierarchy, we get the whole aggregate, modify it, and save it in one DB transaction.
  - Cluster of things that we operate on. We operate on the root, while nested stuff are implementation details, with some constrains.
  
  Aggregates will often publish events.
  Role of aggregates is to protect certain business stuff, that need to be kept safe, when two objects state depend on each other, 
  when one is not valid, when the other one has a certain state.


Strategic Pattern
---------------------
#### Bounded Context
  - Rails makes it hard to refactor nicely cause it has too many ways to access data through AR and it makes it impossible to know all the places affected by certain code changes. Alternative is DDD.
  - Whenever something interesting happens in a certain domain we publish that information. Certain parts of code can usbscribe to it and react to it in their own way, use it in their own data.
  - So instead of one code using data controller by another code, they use an event published by that code based on its data.
  - Makes it easier to detect what you are changing, what can you break when you refactor.
  - They will have two things always
    * Data (database, elastic, whatever)
    * Code
  - BC need to be isolated first and foremost, that is their goal, and their usage, to isolate contexts of codebases and databases, 
  that are bounded together, from different contexts, that might need it, but need to be independent.
  - Communication between BC happens through Domain Events, so when something happens in the code of BC, that can concern
   other BC, an event is being published. Other contexts receive this event, and does something with it.


CQRS (command query responsibility segregation)
---------------------
  - Based on cqs principal that states, that object should have no method that modifies and returns the data in its caller
  - ReadModel and WriteModels are both optimized only to deal with their responsibilities to read/write instead of one object
  that has to handle both functionalities. This is also easier to change, when you have two models tailored for specific things.
  ReadModels are usually really simple, which makes them easy to redo from scratch, refactor, change, enhance, build upon, update.
  - One model flow, makes writes slower, cause write is coupled with read in this approach. So CQRS makes for a nice speed bump.


#### Commands and Events
 Command is a user intent to do something — to perform some action.
 This action may or may not be possible to perform, it is the business rules that decides about in particular case. 
 Those rules live in a model that responds to the invoked command.
 Event is something that describes what happened (in the past form, as this already happened and cannot be rejected like a Command).
 it is a result produced by a model from a Command invoked on it. Aggregate is the model, containing rules


### Example of all this working together:

```ruby
require 'rails_event_store'
require 'aggregate_root'
require 'arkency/command_bus'

Rails.configuration.to_prepare do
  Rails.configuration.event_store = RailsEventStore::Client.new
  Rails.configuration.command_bus = Arkency::CommandBus.new

  AggregateRoot.configure do |config|
    config.default_event_store = Rails.configuration.event_store
  end

  # Subscribe event handlers below
  Rails.configuration.event_store.tap do |store|
    store.subscribe(Movies::OnMovieAddedToRepertoire, to: [Movies::MovieAddedToRepertoire]) #READ MODEL SUBSCRIBES TO EVENT
    store.subscribe_to_all_events(->(event) { Rails.logger.info(event.type) })
  end

  Rails.configuration.command_bus.tap do |bus|
    bus.register(Movies::AddMovieToRepertoire, Movies::OnMovieAddedToRepertoire.new(
      imdb_adapter: OpenStruct.new(fetch_number: "Mocked"))
    )  # ADD COMMAND HANDLER TO THE COMMAND
  end
end
```

```ruby
# Bounded Context for movies
module Movies
end
require_dependency 'movies/movie' #aggregate root
require_dependency 'movies/movie_added_to_repertoire' #event
require_dependency 'movies/on_movie_add_to_repertoire' #command handler
require_dependency 'movies/add_movie_to_repertoire' #command itself
```

```ruby
require 'aggregate_root'

module Movies
  class Movie
    include AggregateRoot

    AlreadyInRepertoire = Class.new(StandardError)
    MovieOutdated = Class.new(StandardError)
    MissingPublisher = Class.new(StandardError)

    def initialize(id)
      @id = id
      @state = :draft
      @tickets = []
    end

    def add_to_repertoire(imdb_id, publisher)
      raise AlreadyInRepertoire if @state == :in_repertoire
      raise MovieOutdated if @state == :outdated
      raise MissingPublisher unless publisher
      apply MovieAddedToRepertoire.new(data: {movie_id: @id, imdb_id: imdb_id, publisher: publisher})
    end

    on MovieAddedToRepertoire do |event|
      @publisher_id = event.data[:publisher_id]
      @imdb_id = event.data[:imdb_id]
      @state = :in_repertoire
    end

    private
  end
end
```

Command is the thing that gets called for example in controller, and then the whole chain follow of command handler, and Read Model
accessing the applied event data.
Command and command handler:
```ruby
# frozen_string_literal: true

module Movie
  class AddMovieToRepertoire < Command
    attribute :movie_id, Types::UUID
    attribute :publisher, Types::Publisher

    alias :aggregate_id :movie
  end
end


# frozen_string_literal: true

module Movies
  class OnMovieAddToRepertoire
    include CommandHandler

    def initialize(imdb_adapter:)
      @imdb_adapter = imdb_adapter
    end

    def call(command)
      with_aggregate(Movie, command.aggregate_id) do |order|
        imdb_number = imdb_adapter.fetch_number
        movie.add_to_repertoire(imdb_number, command.customer_id)
      end
    end

    private

    attr_accessor :imdb_adapter
  end
end
```

Read Model (this those not sit in bounded context in this approach, it is located in app/read_models/movies)
```ruby
# frozen_string_literal: true

module Movies
  class OnMovieAddedToRepertoire
    def call(event)
      movie = Movie.find_by(uid: event.data[:movie_id])
      movie.number = event.data[:movie_number]
      movie.customer = Publisher.find(event.data[:publisher]).name
      movie.state = "in_repertoir"
      movie.save!
    end
  end
end
```

And finally, the event:
```ruby
#frozen_string_literal: true
module Movies
  class MovieAddedToRepertoire < Event
    attribute :movie_id, Types::UUID
    attribute :imdb_number, Types::ImdbNumber
    attribute :publisher, Types::Publisher
  end
end
```

So now, the whole chain of events, commands etc can be started like this.
```ruby
Movies::AddMovieToRepertoire.new(movie_id: params[:movie_id], publisher: publisher)
```
