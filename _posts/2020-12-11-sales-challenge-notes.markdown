---
layout: post
title:  "DDD course notes for sales challenge"
date:   2020-12-12 10:00:00 +0200
description: 'Taken at 40% of the course. Most code so far in the course for me'
permalink: 'ddd_sales_challenge'
author: kkp
---

### Some articles to read before/after this:
- https://railseventstore.org/docs/v1/app/

# Design Challenge
Given a certain business description, provide a design in form of code, drawing, tests or whatever.

## Example Design challenge to implement:
### Sales Processing:
#### Actors
- Sales person
- Company owning products
- Commission platform
#### Requirements
- Sales person sells a product. They get a commission based on agreed conditions, from the company. The commission platform receives an agreed (with the company) N% from each commission too.
- Sales people can sell products from different companies
- The company can see, each month, how much they need to pay to each sales person and why
- The platform admins can see how much they will charge the companies
- The sales person knows at any moment how much of their sales they made - how much commission was made
#### What ifs
- we get a sale information, but the product was not provided yet?
- we get a sale info, but we don't have the commission agreement with the sales man?
- what if the commission agreement can change, also back in time?
### My design of [design challenge](#sales-processing):

Base Files structure for just sale made looks like that:
![Domain Structure](https://i.imgur.com/KaPKtL9.jpg "Files")

And the base domain design for viewing at this [link](https://docs.google.com/drawings/d/1191HnGXYs8LeWVcfA4796CnqJbUJwL_jf0ud2Ty3bE8/edit?usp=sharing)

Here is my explanation for this structure step by step:

### Objects
- *Sales Person*: main player here that makes the sales, and puts most thing into motion. His presence is rather obvious, he has
  to be registered with every sale as the one who made it.
- *Company*: when a sale is made, product that was sold is allocated to certain company. That Company has an agreement with a sales person
  that we need.
- *Platform*: A passive object that takes % from each commission of certain company that it is associated with.
- *Product*: Registered on company, needs to have a price field, and can come in quantities. Can be depleted from stock.
- *Sale*: Although it sounds more like an event, it also needs to be saved in database, registered or something similiar, since
  it holds a lot of important information, and its state can change a lot (cancel, confirm, return etc).
- *Agreement*: Signed between sales person and a company, contains a percentage and a status (:draft, :rejected, :accepted)

### Events

- *SaleMade* main thing to happen in this event are
    1. Product is taken out of stock in quantity given - separate command needs to be issued with its handler
    2. Commissions get distributed (to platform and sales person) (they are calculated too before that maybe, the last event can happen on its own, but should it happen here too? To be certain that commissions are up to date?)
- *AgreementCreated* - When a sales persons signs in with a company, they have to agree on certain %. When they do it for the 1st time this event gets called to save agreement details into DB.
- *AgreementEdited* - Agreement details can change. This accounts for that. This can retroactively change the past commissions received from sales made on old terms, making up for the third `what if`
- *ProductProvided* - Just a stock replenish event, but can also help with the first `what if`
- *CommissionsApplied* - This event will cause commissions for both the sales person and the platform to get calculated. Fired after sale is made, or on demand when altering agreements

This is written as I go so some questions are still here, along with uncertainties.
On to the code. This app's code is available at ![GH](https://github.com/krzykamil/ddd_app/commit/ab94aa40c7fab2a6118383abbc8eecfe6701b866) this time it is fully flesh out mini app that works (with some mocks and assumptions, like 1 product for 1 sale but in many quantities, no discounts, simple relationships, but still).

### Note
This uses nice helpers by Arkency as defined [here](https://github.com/RailsEventStore/cqrs-es-sample-with-res/tree/0de944cf7fa31ba26ee1572b3ccb4d3215b14aed/lib)
I use almost the same (minus the types that I will not use, slight modification in Event) in my lib/ since they are nice and simple and obviously made to work with RES. I recommend checking the source code of them it is some clever stuff!
I will skip showing snippets of migrations, application.rb config etc. cause that is not what is important in this lesson. This note is mostly about
contracting the idea and design.
I have included basic seeds to make the base example work.

```ruby 
#command

module Sales
  class MakeSale < Command
    attribute :sale_id, Types::ID
    attribute :product_id, Types::ID
    attribute :quantity, Types::Coercible::Integer
    alias :aggregate_id :sale_id
  end
end

#command handler

module Sales
  class OnSaleMake
    include CommandHandler

    def call(command)
      with_aggregate(SaleRoot, command.aggregate_id) do |sale|
        sale.make_sale(command.sale_id, command.quantity, command.product_id)
      end
    end
  end
end

#event

module Sales
  class SaleMade < Event
    attribute :sale_id, Types::ID
    attribute :product_id, Types::ID
    attribute :quantity, Types::Integer
  end
end

#Root

require 'aggregate_root'

  require 'aggregate_root'

module Sales
  class SaleRoot
    include AggregateRoot

    NotInStock = Class.new(StandardError)
    NoAgreementWithCompany = Class.new(StandardError)


    def initialize(id)
      @id = id
      @state = "in_progress"
    end

    def make_sale(sale_id, quantity, product_id)
      raise NotInStock if product_quantity_not_enough?(product_id,quantity)
      raise NoAgreementWithCompany if no_agreement?(product_id, sale_id)
      apply SaleMade.new(data: { sale_id: sale_id, quantity: quantity, product_id: product_id })
    end

    on SaleMade do |event|
      @state = "sold"
    end

    private

    def no_agreement?(product_id, sale_id)
      # bad i know, thats not the point here though
      Agreement.find_by(
        owner: SalesPerson.find(Sales::Sale.find(sale_id)),
        company_id: Products::Product.find(product_id).company.id
      )
    end

    def product_quantity_not_enough?(product_id, quantity)
      Products::Product.find(product_id).quantity - quantity < 0
    end
  end
end


#readmodels

module Sales
  class OnSaleMadeEvent
    def call(event)
      find_sale(event.data[:sale_id])
      find_product(event.data[:product_id])
      @sale.update(total: event.data[:quantity] * @product.price)
    end

    private

    def find_product(id)
      @product ||= Products::Product.find(id)
    end

    def find_sale(id)
      @sale ||= Sales::Sale.find(id)
    end
  end
end

module Products
  class OnSaleMadeEvent
    def call(event)
      product = Products::Product.find(event.data[:product_id])
      product.update(quantity: product.quantity - event.data[:quantity])
    end
  end
end

#config

# Subscribe event handlers below
  Rails.configuration.event_store.tap do |store|
    #I added Event part cause it was too similair to command handler (made -> make) at this stage I still need it to make it easier to different classes
    store.subscribe(Sales::OnSaleMadeEvent, to: [Sales::SaleMade])
    store.subscribe(Products::OnSaleMadeEvent, to: [Sales::SaleMade])
    #Note that two read models subscribe to the same event.s
    store.subscribe_to_all_events(->(event) { Rails.logger.info(event.event_type) })
  end

  Rails.configuration.command_bus.tap do |bus|
    # ADD COMMAND HANDLERs TO THE COMMANDs
    bus.register(Sales::MakeSale, Sales::OnSaleMake.new)
  end
end
```

This is what the first version looks like, just a base scenario of making a sale by itself.

```ruby
irb(main):004:0> Products::Product.find(7).quantity
  Products::Product Load (0.6ms)  SELECT "products".* FROM "products" WHERE "products"."id" = ? LIMIT ?  [["id", 7], ["LIMIT", 1]]
=> 11
```

We have some Product7 in stock, lets go and register a sale
![Form](https://i.imgur.com/SA5Tet6.jpg "form")
```ruby
> Products::Product.find(7).quantity
Products::Product Load (0.6ms)  SELECT "products".* FROM "products" WHERE "products"."id" = ? LIMIT ?  [["id", 7], ["LIMIT", 1]]
=> 4
> Sales::Sale.last.total.to_f
Sales::Sale Load (0.6ms)  SELECT "sales".* FROM "sales" ORDER BY "sales"."id" DESC LIMIT ?  [["LIMIT", 1]]
=> 49.0

```

Stuff that we wanted to happened did happen. Awesome. To check up on the events

```ruby

ap RailsEventStoreActiveRecord::Event.last
  RailsEventStoreActiveRecord::Event Load (0.3ms)  SELECT "event_store_events".* FROM "event_store_events" ORDER BY "event_store_events"."id" DESC LIMIT ?  [["LIMIT", 1]]
#<RailsEventStoreActiveRecord::Event:0x000055f99a8d6118> {
            :id => "e880e3bf-0df0-4cbc-bf98-f89dc217ec22",
    :event_type => "Sales::SaleMade",
      :metadata => "---\n:timestamp: 2020-12-09 12:01:24.355745249 Z\n:remote_ip: \"::1\"\n:request_id: c2a10488-3abc-4b50-a540-ad7630b2c040\n",
          :data => "---\n:sale_id: 2\n:product_id: 7\n:quantity: 1\n",
    :created_at => Wed, 09 Dec 2020 12:01:24 UTC +00:00
}
```
We have event data saved.
Now we need to deal with commissions logic, and sales person getting correct commission from the company that owns the product he sold.
We will also account for the edge cases (what ifs)
### Saga Note
I think this process could also work as a saga, with read models for counting total money made from sale, subtracting
products from inventory and calculating commissions. But for the sake of getting a better grasp on what has been mostly covered
so far I continued making it that way.

I decided to create a new Bounded Context for commissions:
1. They need to react to certain events, like change of agreement, sale made
2. Those events come from different BC's. Retracing back to older notes that I made, this point makes a lot more sense now
   ```
   Communication between BC happens through Domain Events, so when something happens in the code of BC, that can concern
   other BC, an event is being published. Other contexts receive this event, and does something with it.
   ```

So commissions code looks somewhat like that (below). Im skipping here displaying the commissions since that rather straightforward.
All that we need to do is collect correct events. In case of agreement change that modifies the commissions in past sales
that is also super easy thanks to event sourcing. When we change agreements, we fire up calculation for all sales of given sales
person or platform depending on who changes the agreement. When we got those new calculations, the display would just need to take
events for each sale that are the newest (specific events about calculating the commissions). Then display the correct value field
from the event. I did not write that code since its rather easy and straightforward and the code below for base commissions already
help me a lot in grasping the whole idea and design of the rest is enough for me here. Pretty happy with how it all is coming together.

![Commissions](https://i.imgur.com/PKJv1r0.jpg "Commission FIles")

```ruby

#read models

module Commissions
   class OnCommissionsApplied
      def call(event)
         # here we could continue with notyfing about commissionsn etc. but in general now displayt matters more but
         # that is outside the scope of this example app
      end
   end
end

# frozen_string_literal: true

module Commissions
   class OnSaleMadeEvent
      def call(event)
         Rails.configuration.command_bus.(Commissions::DistributeSaleCommissions.new(sale_id: event.data[:sale_id]))
      end
   end
end


#AR

require 'aggregate_root'

module Commissions
   class CommissionRoot
      include AggregateRoot

      def initialize(id)
         @id = id
      end

      def distribute_commissions(sale_id)
         total = calculate_sale_total(sale_id)
         apply CommissionsApplied.new(
                 data: {
                         sale_id: sale_id,
                         commission_sales_person: calculate_commission(sales_person_agreement.percentage, total),
                         commission_platform: calculate_commission(platform_agreement.percentage, total)
                 }
         )
      end

      on CommissionsApplied do |event|
         puts "handle"
      end

      private

      def calculate_commission(percentage, total)
         total.to_f * percentage / 100.to_f
      end

      def sales_person_agreement
         Agreement.find_by(owner: @sale.sales_person, company: company)
      end

      def platform_agreement
         Agreement.find_by(owner: platform, company: company)
      end

      def company
         @company ||= Company.find(@sale.company_id)
      end

      def platform
         #doing more for platform in terms of model and architecture seemed like an overkill for this lesson
         @platform ||= company.platform
      end

      def calculate_sale_total(sale_id)
         # Not getting total from model in case of race issue between events. This is why saga might have been a good idea.
         # A different solution is passing more data (not sale ID but sale info, so price of product and quantity, to avoid
         # the query). But this is the simplest in terms of implementation and fastest, the least code, and this challenge
         # is mostly about design and getting a better grip on the ideas themselves.
         sale(sale_id).quantity * Products::Product.find(@sale.product_id).price.to_f
      end

      def sale(sale_id)
         @sale ||= Sales::Sale.find(sale_id)
      end
   end
end

#event
module Commissions
   class CommissionsApplied < Event
      attribute :sale_id, Types::ID
      attribute :commission_sales_person, Types::Float
      attribute :commission_platform, Types::Float
   end
end

#command

module Commissions
   class DistributeSaleCommissions < Command
      attribute :sale_id, Types::ID
      alias :aggregate_id :sale_id
   end
end

#CH

module Commissions
   class OnDistributeSaleCommissions
      include CommandHandler
      def call(command)
         with_aggregate(CommissionRoot, command.aggregate_id) do |commission|
            commission.distribute_commissions(command.sale_id)
         end
      end
   end
end


#updated config

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
      # store.subscribe(Movies::OnMovieAddedToRepertoire, to: [Movies::MovieAddedToRepertoire]) #READ MODEL SUBSCRIBES TO EVENT
      #I added Event part cause it was too similair to command handler (made -> make) at this stage I still need it to make it easier to different classes
      store.subscribe(Sales::OnSaleMadeEvent, to: [Sales::SaleMade])
      store.subscribe(Products::OnSaleMadeEvent, to: [Sales::SaleMade])
      store.subscribe(Commissions::OnSaleMadeEvent, to: [Sales::SaleMade])
      store.subscribe(Commissions::OnCommissionsApplied, to: [Commissions::CommissionsApplied])
      #Note that more than 1 read models subscribe to the same event
      store.subscribe_to_all_events(->(event) { Rails.logger.info(event.event_type) })
   end

   Rails.configuration.command_bus.tap do |bus|
      # ADD COMMAND HANDLERs TO THE COMMANDs
      # bus.register(Movies::AddMovieToRepertoire, Movies::OnMovieAddToRepertoire.new(imdb_adapter: OpenStruct.new(fetch_number: "Mocked")))
      bus.register(Sales::MakeSale, Sales::OnSaleMake.new)
      bus.register(Commissions::DistributeSaleCommissions, Commissions::OnDistributeSaleCommissions.new)
   end
end

```
I have simplified errors to the cases where there is no agreement or products in stock.
There are obviously other, better more complex solutions to it, but writing all this in code here would be overkill
for a design challenge. So a better solution would be something like:
- Allow register a sale in one of the problematic cases, do not calculate commissions, and use UI to display info about problems
- Display the UI info both during sale process and in the panels for sales person and company to let them know, there is a registered sale
  without agreement between them. Events are ok for it.
- Process Manager could also come in handy in this scenarios. Maybe send out agreements to send and request to restock.
  Or maybe schedule PM to keep tracks of agreements and products in stock, and when everythiong checks out, it processes the commissions flow. I like this one better
  it requires a lot more code, cause PM needs to monitor other events and then issue other commands, but its more on the READ side instead
  of write side, which already does a lot of work in this process
  An interesting idea would not to keep commissions calculation on the write side, after sale is made,
  just read it everytime it is being accessed.
