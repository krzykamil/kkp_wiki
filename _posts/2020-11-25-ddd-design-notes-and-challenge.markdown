---
layout: post
title:  "DDD course notes for design and a challenge!"
date:   2020-11-25 15:00:00 +0200
description: 'Taken at 36% of the course. First bigger challenge, but the next one if even more detailed and complicated.'
permalink: 'ddd_design_notes_with_challenge'
author: kkp
---

# Design Notes

### Vocabulary and reminders
    AR - Aggregate Root
    BC - Bounded Context
    Authorization - what a person can do
    Authentication - who a person is

### Some General Rules
- Bounded context can have multiple Aggregates, you do not have to have 1 AR per BC.
Rule of thumb is that 3-10 is okey, and 1 is standard for simpler/smaller BC
- Rails is definitely not the most important thing in DDD rails app, as weird as it sounds. There is no need
to try and follow Rails directory structure when using DDD in rails, rails can just be used for its routing, views, controllers
etc, while DDD follows its own rules of segregating read models, BCs. Do not force rails architecture into DDD, do it the other way around.
- Authentication and Authorization: 
  - can though of as BC (to me that seems too complicated, and makes it easier to mistake it into putting auth into domain level whcih is not fine as said below)
  - current_user makes sense at the application level, but not at domain level. Passing id of user into command is fine, but not the whole objects.
  standard solution is app controller filter that uses current user. deal with auth and then creates commands, passes them further, or grabs read model etc.
  Domain elements should not know about auth, application should, since auth is part of the application not the domain.
 
### Example Design challenge to implement:
#### Workshops:
##### Scenario
- organizer holds a new on-site workshop edition
- edition is financially viable if at least 20 participants register, otherwise its called off
- edition ca be held if organizer manages to book a venue
- above conditions need to be made 2 weeks before workshop date
##### Happy path: edition is confirmed
- 20 people register
- venue is booked
- still 2 weeks or more until workshop conducts
##### Unhappy path: edition is cancelled
- less than 20 people with 2 weeks left
- no venue with 2 weeks left

### My solution to [design challenge](#example-design-challenge-to-implement):

My thought process went as follow (along with the code):
At first I thought that the easiest thing to identify is all the events and commands to trigger the events.
Commands I came up with are obviously *scheduling the edition*, *registering a participant for it*, *booking a venue*, and happy and
unhappy paths described, so *confirming* it or *cancellation*.
This gives me 5 commands (put into 1 module file for clarity, they are small since its just design, not a real implementation):

```ruby
module Workshops
  module Commands
    class ScheduleEdition < Command
      attribute :edition_id, Types::UUID
      attribute :date, Types::DateTime
    end

    class RegisterForEdition < Command
      attribute :edition_id, Types::UUID
      attribute :user_id, Types::UUID # assuming user is the participant here
    end

    class BookVenue < Command
      attribute :edition_id, Types::UUID
      attribute :venue_id, Types::UUID
    end

    class ConfirmEdition < Command
      attribute :edition_id, Types::UUID
    end

    class CancelEdition < Command
      attribute :edition_id, Types::UUID
    end
  end
end
```

So each and every of those commands would have their CommandHandler hooked into CommandBus. Command gets called, CommandHandler
gets fired to do what needs to be done so uses AggregateRoot to publish an event.
Here are those events:

```ruby
module Workshops
  module Events

    # Assuming existence of just some base event class
    class ScheduledEdition < Event
      attribute :edition_id, Types::UUID
      attribute :date, Types::DateTime
    end

    class UserRegisteredForEdition < Event
      attribute :edition_id, Types::UUID
      attribute :user_id, Types::UUID
    end

    class BookedVenue < Event
      attribute :edition_id, Types::UUID
      attribute :venue_id, Types::UUID
    end

    class ConfirmedEdition < Event
      attribute :edition_id, Types::UUID
    end

    class CanceledEdition < Event
      attribute :edition_id, Types::UUID
    end
  end
end
``` 

What really sticks out so far for me in DDD (or maybe just this approach) is how similar events and commands structure is. I know this probably comes down
to examples being really simple, but events like this are identical to commands, and with little differences in naming, it can be confusing at time.
But this might come from the simplicity of the examples. It probably is not a problem later on, with a good files structure, careful naming
and general better understanding of the subject, implementation and domain.

Anyway, next step has to be aggregate root that will get used by command handlers to publish events.

```ruby
#lets go with arkency aggregate root formula (or rather my understanding of it)
require 'aggregate_root'

module Workshop
  module Edition
    include AggregateRoot

    def initialize(id)
      @id = id
      @status = workshop.status #we start each edition as a draft, and then change the status based on events
      @users_registered_size ||= users_count
      @start_date = workshop.start_date
    end

    def schedule_edition(date:)
      apply ScheduledEdition.new(data: {edition_id: @id, date: date})
    end

    on ScheduledEdition do |event|
      @status = :scheduled
      @start_date = event.data[:date]
    end

    def register_for_edition(user_id:)
      apply UserRegisteredForEdition.new(data: {edition_id: @id, user_id: user_id})
    end

    on UserRegisteredForEdition do |event|
      @users_registered_size += 1
    end

    def book_venue(venue_id:)
      apply BookedVenue.new(data: {edition_id: @id, venue_id: venue_id})
    end

    on BookedVenue do |event|
      @status = :booked
    end

    def confirm_edition
      apply ConfirmedEdition.new(data: {edition_id: @id}) if can_confirm?
    end

    on ConfirmedEdition do |event|
      @status = :confirmed
    end

    def cancel_edition
      apply CanceledEdition.new(data: {edition_id: @id}) if can_cancel?
    end

    on CanceledEdition do |event|
      @status = :cancelled
    end

    private

    def users_count
      workshop.users.size
    end

    def workshop
      @workshop ||= WorkshopEdition.find(@id)
    end

    def can_confirm?
      @users_registered_size >= 20 && @status = :booked &&
        @start_date.beginning_of_day >= (Time.now + 14.days).beginning_of_day
    end


    def can_cancel?
      @users_registered_size < 20 && @status >= :booked &&
        @start_date.beginning_of_day == (Time.now + 14.days).beginning_od_day
    end
  end
end
```

In the aggregate root what i am most uncertain of is if setting the state here is okey, or should it be determined by event - 
so commands would set the state, and events register them? It was hard to determine this, since I had no idea how useful any of those options
would be.

With base AR somewhat ready I could add command handlers. That handle issuing events to AR. Those are simple handlers, they
are even initialized as empty, with just command passed to their calls.

```ruby
# We assumme each handler has access to aggregate root through `with_aggregate` just like in arkency RES gem implementation
module Workshops
  module Handlers
     class OnEditionSchedule < CommandHandler
       def call(command)
         with_aggregate(Edition, command.aggregate_id) do |edition|
           edition.schedule_edition(date: command.date)
         end
       end
     end

     class OnEditionRegistration < CommandHandler
       def call(command)
         with_aggregate(Edition, command.aggregate_id) do |edition|
           edition.register_for_edition(user_id: command.user_id)
         end
       end
     end

     class OnVenueBooking < CommandHandler
       def call(command)
         with_aggregate(Edition, command.aggregate_id) do |edition|
           edition.book_venue(venue_id: command.venue_id)
         end
       end
     end

     class OnEditionConfirm < CommandHandler
       def call(command)
         with_aggregate(Edition, command.aggregate_id) do |edition|
           edition.confirm_edition
         end
       end
     end

     class OnEditionCancel < CommandHandler
       def call(command)
         with_aggregate(Edition, command.aggregate_id) do |edition|
           edition.cancel_edition
         end
       end
     end
  end
end

# CONFIG
Rails.configuration.command_bus.tap do |bus|
  bus.register(Workshops::Commands::ScheduleEdition, Workshops::Handlers::OnEditionSchedule.new)
  bus.register(Workshops::Commands::RegisterForEdition, Workshops::Handlers::OnEditionRegistration.new)
  bus.register(Workshops::Commands::BookVenue, Workshops::Handlers::OnVenueBooking.new)
  bus.register(Workshops::Commands::ConfirmEdition, Workshops::Handlers::OnEditionConfirm.new)
  bus.register(Workshops::Commands::CancelEdition, Workshops::Handlers::OnEditionCancel.new)
end
```

And then of course ReadModels and their config
```ruby

module Workshops
  class ReadModels
    class ScheduledEdition
      def call(event)
        WorkshopEdition.find(event.data[:edition_id]).update(date: event.data[:date])
      end
    end

    class OnUserRegistered
      def call(event)
        workshop = WorkshopEdition.find(event.data[:edition_id])
        workshop.users << User.find(event.data[:user_id])
      end
    end

    class OnBookedVenue
      def call(event)
        WorkshopEdition.find(event.data[:edition_id]).update(venue_id: event.data[:venue_id])
      end
    end

    class OnConfirmedEdition
      def call(event)
        WorkshopEdition.find(event.data[:edition_id]).confirm
      end
    end

    class OnCanceledEdition
      def call(event)
        WorkshopEdition.find(event.data[:edition_id]).cancel
      end
    end
  end
end
Rails.configuration.event_store.tap do |store|
  store.subscribe(Workshops::ReadModels::OnEditionSchedule, to: [Workshops::Events::ScheduledEdition])
  store.subscribe(Workshops::ReadModels::OnUserRegistered, to: [Workshops::Events::UserRegisteredForEdition])
  store.subscribe(Workshops::ReadModels::OnBookedVenue, to: [Workshops::Events::BookedVenue])
  store.subscribe(Workshops::ReadModels::OnConfirmedEdition, to: [Workshops::Events::ConfirmedEdition])
  store.subscribe(Workshops::ReadModels::OnCanceledEdition, to: [Workshops::Events::CanceledEdition])
end
```

I am not happy about some of the stuff here but I was not sure about many of them, so I left it as it is to compare with
the actual design solution to look for better solutions for those problems.
My notes of my problems along with solutions and how it should be done after reviewing the solution below along with pointing
up the stuff that was actually good it seems.

### Notes after reviewing the walkthrough to: [design challenge](#example-design-challenge-to-implement):
- **Thought:** Probably should have started with aggregate first, end defining events there, and then adding commands to it.
This order seems more logical, since you start with a more connected structure right away, the way I did it, make it a bit harder
to build a sensible AR.
- **Mistake:** confirming an event should be checked after every registration of a new user, it be automatic. The way I did it
it required a separate command to be issued. Probably a mistake that came from only trying to do a design code instead of fully working code.
I think I misunderstood the task in that second issue, but that not the point of this mistakes list.
On book venue we should also try to confirm. I kinda assumed those things would be checked and called periodically. 
- **Mistake:** Booking a venue should not actually be a bard of that AR. That process could be complicated or full of
edge cases and paths. There is a need for another BC, dealing with venues.
- **Mistakes:** A separate `Policy` object should be added for cancellation or confirming, called with adequate
events. Those Policy objects would check conditions for their actions (cancel/confirm) and issue command to command bus, based on 
conditions. Something like this:

```ruby

module Workshops
  module CancellationPolicy
    def initlize(command_bus)
      @command_bus = command_bus
    end

    def call(event)
      edition_id = event.data.fetch(:edition_id)
      @events[edition_id] << event
      return confirm_edition if can_be_confirmed?(edition)
      cancel_edition if can_be_cancelled?(edition)
    end

    private

    def can_be_confirmed?(edition_id)
      ...
    end

    def can_be_cancelled?(edition_id)
      ...
    end

    def enough_people_registered?(edition_id)
      ...
    end

    def confirm(edition_id)
      @command_bus.(ConfirmEdition.new(edition_id))
    end

    def cancel(edition_id)
      @command_bus.(CancelEdition.new(edition_id))
    end
  end
end

```

-**Thought:** there was a nice way of scheduling the event into the future presented, which I liked. So instead
of classical cron job we could do something like this:

```ruby
module Calendar
  class TwoWeeksBeforeEditionReached < RailsEventStore::Event
  end

  class Deadline
    include Sidekiq::Worker
    def perform(edition__id)
      Rails.configuration.event_store.publish(TwoWeeksBeforeEditionReached.new(data: {
        edition_id: edition_id
      }))
    end
  end
end

deadline = -> (edition_scheduled) do
  Calendar::Deadline.perform_at(
    edition_scheduled.data.fetch(:start_date) - 2.weeks,
    edition_scheduled.data.fetch(:edition_id)
  )
end

Rails.configuration.event_store.subscrive(deadline, to: [EditionScheduled])
```

- **Good that I did:** I think a lot of the connections between classes clicked for me here finally. Event to command
to command handler to aggregate. It is hard without working on such a complex concept without a real project
so even like a small design challenge like this helped a lot to put stuff together and I think I did okey in it.
My structure was not that much different than what was showed, even if I failed to identify a separate BC.
- **Mistake:** Creating a working implementation rather than a design skeleton would allow me to write tests, to put all
of this together even more nicely.

I will try to make the next challenge more as an small app based on this experience and I think it will be better.
