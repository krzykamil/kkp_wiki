---
layout: post
title:  "Mutations - Part I"
date:   2023-03-23 15:55:36 +0200
description: 'First two simple cases of mutation testing'
permalink: 'mutations_part_1'
author: kkp
---

# Mutant cases:

## Case 1:

### Test
```ruby
RSpec.describe OffersAndOrders::TransitStatus do
  extend Support::WithEveryCombination
  subject { described_class.call(params: params, order: order) }

  let(:params) { {status: new_status, comment: comment} }
  let(:company) { create :company }
  let(:user) { create :user, role: :seller }
  let(:offer) { create :offer, company: company, user: user }
  let(:order) { create :order, offer: offer, user: user, company: company, status: Order.statuses.keys.sample, comment: "upsi" }
  let_it_be(:comment) { "hehehe" }

  context "when status is valid" do
    with_every_combination(status: Order.statuses.keys) do
      let(:new_status) { status }
      it "succeeds" do
        expect(subject).to be_success
        expect(subject.result[:status]).to eq status
        expect(order.reload.comment).to eq comment
      end
    end
  end

  context "when status is invalid" do
    let(:new_status) { "JAJCO" }
    it "fails" do
      expect(subject).to be_failure
      expect(subject.errors[0]).to eq "status Podana wartość nie jest dozwoloną wartością"
      expect(order.reload.status).to eq order.status
      expect(order.reload.comment).to eq "upsi"
    end
  end

  context "with correction" do
    let(:new_status) { "correction" }
    it "it sends an email" do
      expect { subject }.to have_enqueued_mail(SellerMailer, :send_correction_info).with(order)
    end
  end
end

```
### Object

```ruby
module OffersAndOrders
  class TransitStatus
    include BaseService

    def initialize(params:, order:)
      @params = params
      @status = params.fetch(:status)
      @comment = params.dig(:comment)
      @order = order
    end

    def call
      validation = OrderContract.new.call(**params)
      if validation.success? && order.update(status: status, comment: comment)
        send_seller_email if order.correction?
        add_result(status: status)
      else
        full_default_error_messages(validation)
      end
    end

    private

    attr_reader :comment, :order, :status, :params

    def send_seller_email
      SellerMailer.send_correction_info(order).deliver_later
    end
  end
end

```
### Mutation Result
```shell
Mutations:       113
Results:         113
Kills:           111
Alive:           2
Timeouts:        0
Runtime:         101.93s
Killtime:        392.37s
Overhead:        -74.02%
Mutations/s:     1.11
Coverage:        98.23%
```

#### Mutation:
1.
  ```ruby
  def call
    validation = OrderContract.new.call(**params)
    if validation.success? && order.update(status: status, comment: comment)
    -  if order.correction?
    +  if true
      send_seller_email
    end
      add_result(status: status)
    else
      full_default_error_messages(validation)
    end
  end
  ```

##### Changes:

```ruby
  context "with email" do
    context "with correction" do
      let(:new_status) { "correction" }
      it "it sends an email" do
        expect { subject }.to have_enqueued_mail(SellerMailer, :send_correction_info).with(order)
      end
    end

    context "with non-correction" do
      with_every_combination(status: Order.statuses.except(:correction).keys) do
        let(:new_status) { status }
        it "it does NOT sends an email" do
          expect { subject }.to_not have_enqueued_mail(SellerMailer, :send_correction_info)
        end
      end
    end
  end
```

##### Conclusions:
In general mutant tips were mostly about inprecise code, which checked that the email will be sent out when the status is set to "correction". Changing the if statement in mutant as lead to inproper behaviour where email is always sent out, but the code did not notice that, because it will still send out at correction status. Very sensible case that helps make the test more precise.

## Case 2:
### Test
```ruby
require "rails_helper"

RSpec.describe Companies::AssignUser do
  subject(:result) { described_class.call(user: user, target: company) }
  let(:user) { create :user, role: "supervisor" }
  let!(:company) { create :company, nip: nip }
  let(:nip) { "1234567899" }

  context "assign user to company" do
    it "check relation user to company" do
      subject
      company = Company.find_by(nip: nip)
      expect(company.users).to be_present
    end
  end

  context "assign user to company branch" do
    let!(:existing_company) { create :company, nip: nip }
    let(:company) { create :company_branch, company: existing_company }

    it "check relation user to company branch" do
      subject
      company_branch = Company.find_by(nip: nip).company_branches.first
      expect(company_branch.users).to be_present
    end
  end

  it "has result data" do
    expect(result.result[:target]).to eq Company.find_by(nip: nip)
  end
end

```
### Object

```ruby
module Companies
  class AssignUser
    include BaseService

    def initialize(user:, target:)
      @target = target
      @user = user
    end

    def call
      target.users << user
      target.save!
      add_result(target: target)
    end

    private

    attr_accessor :target, :user
  end
end

```
### Mutation result
```shell
Mutations:       38
Results:         38
Kills:           36
Alive:           2
Timeouts:        0
Runtime:         24.28s
Killtime:        90.78s
Overhead:        -73.25%
Mutations/s:     1.57
Coverage:        94.74%
```
### Mutations
1:
```ruby
 def call
   target.users << user
-  target.save!
+  target
   add_result(target: target)
 end
```

2:
```ruby
 def call
   target.users << user
-  target.save!
+  nil
   add_result(target: target)
 end
```
##### Changes:
The object did not do enough, so this time the changes were not just in specs.
```ruby
user.update(company: target.company) if target.instance_of?(CompanyBranch)
```
Removed this line
```ruby
target.save!
```
Since this performs a save
```ruby
target.users << user
```
End test:
```ruby
require "rails_helper"

RSpec.describe Companies::AssignUser do
  subject(:result) { described_class.call(user: user, target: target) }
  let(:user) { create :user, role: "supervisor" }
  let!(:company) { create :company, nip: nip }
  let(:nip) { "1234567899" }
  let(:target) { company }

  context "assign user to company" do
    it "check relation user to company" do
      subject
      company = Company.find_by(nip: nip)
      expect(company.users).to be_present
      expect(user.company).to eq company
    end
  end

  context "assign user to company branch" do
    let(:company_branch) { create :company_branch, company: company }
    let(:target) { company_branch }

    it "check relation user to company branch" do
      subject
      company_branch = Company.find_by(nip: nip).company_branches.first
      expect(company_branch.users).to be_present
      expect(user.company).to eq company
      expect(user.company_branch).to eq company_branch
    end
  end

  it "has result data" do
    expect(result.result[:target]).to eq Company.find_by(nip: nip)
  end
end

```

##### Conclusion:
Both mutation pointed the same thing out: if we did not `save!` the object that got passed to target, the test result would be the same. Thies meant that the spec did not correctly test if the user has been assigned to the company.
This has turned out to be true, taking a closer look to the spec, you can see that the Company object is checked for users number, but it was not very precise. It had assumed that the seeds/factory for the company, does not assign it a default user (and in fact it did not). The spec was based on this assumption too heavily. If the set up for the company would be change, in factory or not, the spec would become even more inprecise.
In general a good example, once again, of making specs more precise, making it less vulnerable to outside changes, made to factories, databases or set ups of tests.
It also helped to see a bug in actual code.
