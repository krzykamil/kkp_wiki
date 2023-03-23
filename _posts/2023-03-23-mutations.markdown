---
layout: post
title:  "Mutations - Part I"
date:   2023-03-23 15:55:36 +0200
description: 'First two simple cases of mutation testing'
permalink: 'mutations_part_1'
author: kkp
---

# Mutantowe przypadki:

## Przypadek 1:

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
### Obiekt

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
### Result mutacji
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

#### Mutacje:
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
##### Zmiana:
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

##### Wnioski:
Generalnie podpowiedź z mutacji dotyczała nieprecyzyjnego testu, który badał, że mail wyśle się jak status będzie "correction". Zmiana kontrolki w ifie oczywiście prowadziłaby do niepoprawnego zachowania gdzie mail wysyła się zawsze, ale kod testu się tym nie przejmował, bo wysyłał się tez na correction.
Nic szczególnego, prosty

## Przypadek 2:
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
### Obiekt

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
### Result mutacji
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
### Mutacje
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
##### Zmiana:
Tutaj musiał się zmienić też kod obiektu, bo wyszło na to, że robi za mało. Do obiektu dodano tylko:
```ruby
user.update(company: target.company) if target.instance_of?(CompanyBranch)
```
Usunięta została też linijka
```ruby
target.save!
```
Jako, że ta akcja wykonuje zapis:
```ruby
target.users << user
```
A test skończył tak:
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

##### Wnioski:
Obie mutacje tak naprawdę wskazuja na to samo: jakbyśmy nie zrobili `save!` na obiekcie przekazywanym do targetu, to efekt testu będzie taki sam. Tzn. że test nie sprawdza poprawnie czy użytkownik został dopisany do firmy.
No i jest to prawdą, przyglądając się testowi widać, że sprawdzany jest obiekt Company pod względem obecności userów, brak jednak temu precyzji. Jest założenie, że Company w seedach nie ma usera (i nie ma) i na tym założeniu opiera się test. Więc w razie rozszerzenia setupu firmy, gdzie miałaby usera, test byłby już w ogóle jeszcze bardziej nieprecyzyjny.
Generalnie ten przypadek, pokazuje gdzie można doprecyzować test, dzięki czemu będzie też mniej podatny na jakieś zewnętrzne zmiany z setupem testów, danych.
Dodatkowo dzięki doprecyzowaniu tego testu udało się odkryć brak w kodzie
