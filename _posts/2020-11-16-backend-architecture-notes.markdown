#BackendArchitectureNotes

### Some articles to read before/after this:

- https://martinfowler.com/articles/201701-event-driven.html - nice read about the topic from Martin Fowler
- https://blog.arkency.com/2016/09/command-bus-in-a-rails-application/ - about command bus gem, but also as a pattern of commands in CQRS overall
- https://blog.arkency.com/process-managers-revisited/ - about Process Managers pattern
- https://blog.arkency.com/2017/06/dogfooding-process-manager/ - also about Process Managers pattern
##CQRS READ MODELS

 **In CQS a method must either be a command that performs an action or a query that returns data. Never both**

Most apps are 90% read and 10% write. So separating read from write can have a huge performance boost. Also separating write
is rather important, since it protects, your domain and business logic better.

#####Pros of CQRS in short:
- Improve read operations performance
    - Data store of read models is optimized for reads
    - Scales independently
 - Simplicity and ease of changes in read models
    - thor away and rebuild
- Decluttering the domain
    - not exposing public attrs for reads
    - easier to test, since you can only test the write side
Cons are obviously a requirement of changing the mindset. It is a big infrastructure too, larger codebase.
Write operations might rake longer, but do no need to.

Read Model
--------
  A good read model will be/have:
  - Read Optimized
    - Denormalized data (duplicating is ok, when it helps you to quickly display data).
    - Tailor Made - do not force using the same read model for every place/page/view etc. Prepare exactly what you need, it makes read models simpler.
  - Re-Creatable 
    - it is build completely from domain events.
    - you can throw it away and rebuild it historic from domain data, when you achieve the above point
  
  RM can be build without Domain Event ofc. but its not decoupled, it cannot be easily deactivated/refactored.
    
#####Read Models and caching:
  - RM is the solo consumer of changing events, we always know when data will change
  - Caching is time based or key-attribute based

**CQRS does not have to be a top level architecture. It can be applied as part of the system, whenever
the benefits outweigh the costs.**

_Read Models should limit the logic as much as possible, its best if you manage to keep them as places where you 'dump' data._

Solution to it, is trying to handle as much logic as possible on the write side ofc. Write side should be more complicated.

Event Sourcing
--------
**Storing all changes in form of domain events**

#####Pros:
  - no OR mapping, means avoidance of impedance mismatch
  - append only store, not losing any data, like ever
  - audit logs come with
  - rebuilding state is easier so, so much   
  - replay events to reprocess old data
  - temporal queries are easy
  - different read models possible
  - easy to scale, start small go big with the same architecture

#####Cons:
  - no rails c and change some data on prod
  - no data migration
  - huge mindset change
  - need separate models for reporting
  - lack of tooling

How it even works:
 1. Read old models from DB
 2. Apply historical events to change state
 3. Call a method/command
       - Verify variants
       - Apply new events to change state
 4. Save new events to database and publish to MQ (message que)

**Only safe way to change is to apply new events.**

_Applying the event cannot have side effects outside the object in question._ 

Remember that bringing too much data into events is an anti pattern, and means that there is not enough decoupling in your system.
 You probably need more divided events etc. in those situations.

Process Managers and Sagas
--------

#####Example system
1. PostalAddedToOrder     - #event
2. PostalAddressFilledOut - #event
3. PdfGenerated           - #event
4. PaymentPaid            - #event

5. => SendPdfViaPostal #command

Events do not need to happen in that exact order though.

###Saga
A saga is a "long-lived business transaction or process". The problem with aggregates is that they only care about their little part of the universe. Sagas, on the other hand, are coordinating objects.  They listen for what's happening and they tell other objects to take appropriate action.
Sagas manage process.  They contain business behavior, but only in the form of process.  This is a critical point.  Sagas, in their purest form, don't contain business logic.

Sagas work on commands mostly, since command are simpler concepts, they produce a clear result.
- **Domain events work in publish-subscribe way. When an event happens, there can be multiple subscribers. In commands there is 1-to-1 mapping. 1 command 1 command handler**

Both commands and events are messages basically, that produce the output based on input. But Events operate in past tense, while commands represent an intent.
Saga can take many events as an input, as presented in the [example system](#example-system). All those events can be presented as an input to the Saga that will then invoke the correct command.
Saga can for example inherit from ApplicationJob, use sidekiq etc.
Saga can also for example handle cases like the same event happening several times. For example if you have an event about order being purchased, and you get a discount
after 5  purchased orders, Saga will await receiving 5 of those event and then send a command that will generate discount.

###Process Managers
Also a pattern that can be helpful in modeling long running business processes. They usually consist of multiple steps, they do not have to happen in any specific order etc. or one after another.
This pattern is helpful when there can be some kind of 'race' problems, when events can happen in different order everytime they happen, when the whole process in initialized.
Like in sagas, PM operates on multiple domain events and produces a command. It is kind of a big event handler, listening to many events at once.

- Process Manager is at its basis a function. It takes some events as an argument and then issues a command. Command triggers processes in the rest of the system.
- Its important to distinct PM from aggregate. PM automates a certain process, orchestrating flow between several bounded contexts (usually)
- PM has a state, that can be done using AR, Event Sourcing state or whatever you want.
- PM fetches the current state, applies new domain event based on the state and decides if the system needs to perform some action if yes it invokes a correct command  to issue that.

Example process manager, for processing the payment, using standard credit card flow with authorize, capture and release.
```ruby
class PaymentProcess
  def initialize(store: Rails.configuration.event_store,
                 bus: Rails.configuration.command_bus)
    @store = store
    @bus = bus
  end

  def call(event)
    state = build_state(event)
    if state.release?
      bus.call(Payments::ReleasePayment.new(
        order_id: state.order_id,
        transaction_id: state.transaction_id))
    end
  end

  private
  attr_reader :store, :bus

  def build_state(event)
    stream_name = "PaymentProcess$#{event.data.fetch(:order_id)}"
    past = store.read.stream(stream_name).to_a
    last_stored = past.size - 1
    store.link(event.event_id, stream_name: stream_name, expected_version: last_stored)
    ProcessState.new.tap do |state|
      past.each{|ev| state.call(ev)}
      state.call(event)
    end
  rescue RubyEventStore::WrongExpectedEventVersion
    retry
  end

  class ProcessState
    def initialize
      @order = :draft
      @payment = :none
    end
    attr_reader :transaction_id, :order_id

    def call(event)
      case event
      when Payments::PaymentAuthorized
        @payment = :authorized
        @transaction_id = event.data.fetch(:transaction_id)
      when Payments::PaymentReleased
        @payment = :released
      when Ordering::OrderSubmitted
        @order = :submitted
        @order_id = event.data.fetch(:order_id)
      when Ordering::OrderExpired
        @order = :expired
      when Ordering::OrderPaid
        @order = :paid
      end
    end

    def release?
      @payment == :authorized && @order == :expired
    end
  end
end
```
And a test for it
```ruby
require 'test_helper'

class PaymentProcessTest < ActiveSupport::TestCase
  test 'happy path' do
    fake = FakeCommandBus.new
    process = PaymentProcess.new(bus: fake)
    given([
      order_submitted,
      payment_authorized,
      order_paid
    ]).each do |event|
      process.call(event)
    end
    assert_nil(fake.received)
  end

  test 'order expired without payment' do
    fake = FakeCommandBus.new
    process = PaymentProcess.new(bus: fake)
    given([
      order_submitted,
      order_expired,
    ]).each do |event|
      process.call(event)
    end
    assert_nil(fake.received)
  end

  test 'order expired after payment authorization' do
    fake = FakeCommandBus.new
    process = PaymentProcess.new(bus: fake)
    given([
      order_submitted,
      payment_authorized,
      order_expired,
    ]).each do |event|
      process.call(event)
    end
    assert_equal(fake.received,
      Payments::ReleasePayment.new(transaction_id: transaction_id)
    )
  end

  test 'order expired after payment released' do
    fake = FakeCommandBus.new
    process = PaymentProcess.new(bus: fake)
    given([
      order_submitted,
      payment_authorized,
      payment_released,
      order_expired,
    ]).each do |event|
      process.call(event)
    end
    assert_nil(fake.received)
  end

  private

  class FakeCommandBus
    attr_reader :received
    def call(command)
      @received = command
    end
  end

  def transaction_id
    @transaction_id ||= SecureRandom.hex(16)
  end

  def order_id
    @order_id ||= SecureRandom.uuid
  end

  def order_number
    '2018/12/16'
  end

  def customer_id
    123
  end

  def given(events, store: Rails.configuration.event_store)
    events.each{|ev| store.append(ev)}
    events
  end

  def order_submitted
    Ordering::OrderSubmitted.new(data: {order_id: order_id, order_number: order_number, customer_id: customer_id})
  end

  def order_expired
    Ordering::OrderExpired.new(data: {order_id: order_id})
  end

  def order_paid
    Ordering::OrderPaid.new(data: {order_id: order_id, transaction_id: transaction_id})
  end

  def payment_authorized
    Payments::PaymentAuthorized.new(data: {
      transaction_id: transaction_id,
      order_id: order_id
    })
  end

  def payment_released
    Payments::PaymentReleased.new(data: {
      transaction_id: transaction_id,
      order_id: order_id
    })
  end
end
```

And here is how you can subscribe the events to the process in the config of event store
```ruby
    store.subscribe(PaymentProcess, to: [
      Ordering::OrderSubmitted,
      Ordering::OrderExpired,
      Ordering::OrderPaid,
      Payments::PaymentAuthorized,
      Payments::PaymentReleased,
    ])
```


###Differences between Sagas and Process Managers
Sagas are more about being able to compensate for failure in a process, for example backtracking in a process that has a predictable failure, or just allows this kind of action to be taken.
PM in this case, are more useful to handle processes that are complicated, or large, but do not allow failures.
For example: you book a hotel, book a car, and then try to book a flight, but something goes wrong there/you change your mind. You go back and cancel the car and hotel. Saga is the pattern to be used
in this kind of scenario, not Process Manager.
