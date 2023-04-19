---
layout: post
title:  "Mutations - Part III"
date:   2023-04-19 15:55:36 +0200
description: 'Another case of mutation testing'
permalink: 'mutations_part_3'
author: kkp
---

# Mutation cases:

## Case 1:
### Test:
```ruby
require "rails_helper"

RSpec.describe CompanyQuery do
  subject(:result) { described_class.new(params: params).call }
  let(:params) {
    {
      nip: nip,
      phone: phone,
      company_id: company_id,
      companies_status: status,
      page: page,
      per_page: per_page,
      sort_column: column,
      sort_direction: direction
    }
  }
  let(:nip) { "" }
  let(:phone) { "" }
  let(:company_id) { "" }
  let(:name) { "" }
  let(:status) { "" }
  let(:page) { 1 }
  let(:per_page) { 25 }
  let(:column) { "" }
  let(:direction) { "" }

  context "with one param" do
    let(:status) { "active" }
    let!(:company_1) { create :company, nip: nip, name: name, registration_status: status, phone: phone }
    let!(:company_2) { create :company, nip: "0000000111", name: "taki sie nie wygeneruje", phone: "666 666 666" }

    context "when given nip exists" do
      let(:nip) { "0000000001" }

      it "returns correct company" do
        expect(result.pagination.count).to eq 1
        expect(result.records.first).to eq company_1
      end
    end

    context "when given company_id exists" do
      let(:company_id) { company_1.id }

      it "returns correct company" do
        expect(result.pagination.count).to eq 1
        expect(result.records.first).to eq company_1
      end

      context "when param phone is not precise " do
        let!(:company_1) { create :company, nip: nip, name: name, registration_status: status, phone: "123 123 123" }
        let(:phone) { "123 " }
        it "return correct company" do
          expect(result.pagination.count).to eq 1
          expect(result.records.first).to eq company_1
        end
      end
    end

    context "when given phone exists" do
      let(:phone) { "999 999 999" }

      it "returns correct company" do
        expect(result.pagination.count).to eq 1
        expect(result.records.first).to eq company_1
      end
    end

    context "when given status exists" do
      let(:status) { "active" }

      it "returns correct company" do
        expect(result.pagination.count).to eq 1
        expect(result.records.first).to eq company_1
      end
    end
  end

  context "with combined params" do
    let(:status) { "active" }

    let!(:company_1) { create :company, nip: nip, name: name, registration_status: status, phone: phone }
    let!(:company_2) { create :company, nip: nip, name: name, phone: "666 666 666", registration_status: status }

    context "when both nip and phone exists" do
      let(:nip) { "0000000001" }
      let(:phone) { "999 999 999" }

      it "returns companies that match any of the params" do
        expect(result.pagination.count).to eq 2
        expect(result.records).to match array_including(company_1, company_2)
      end
    end

    context "when none of the params are in database" do
      let(:nip) { "0000000001" }
      let(:phone) { "999 999 999" }
      let!(:company_1) { create :company, nip: "111111111", name: "notinaquery", registration_status: status, phone: "123 123 123" }
      let!(:company_2) { create :company, nip: "222222222", name: "nope", phone: "666 666 666", registration_status: status }

      it "returns nothing" do
        expect(result.pagination.count).to eq 0
        expect(result.records.first).to eq nil
      end
    end

    context "when by id exists but the status is different" do
      let(:company_id) { company_1.id }
      let!(:company_1) { create :company, name: name, registration_status: "pending" }
      let!(:company_2) { create :company, name: name, registration_status: status }

      it "does not return anything" do
        expect(result.pagination.count).to eq 0
      end
    end
  end

  context "when pagination spans multiple pages" do
    let(:per_page) { 1 }
    let(:page) { 2 }
    let(:name) { "name" }
    let!(:company_1) { create :company, name: name, registration_status: status }
    let!(:company_2) { create :company, name: name, registration_status: status }
    it "return the page from params" do
      expect(result.pagination.page).to eq 2
    end
  end

  context "with sorting params" do
    let(:column) { "name" }
    let(:direction) { "asc" }
    let!(:company_1) { create :company, name: "ccc" }
    let!(:company_2) { create :company, name: "bbb" }

    it "return sorted asc params by name" do
      expect(result.records.first).to eq company_2
      expect(result.records.last).to eq company_1
    end

    context "with sorting params desc" do
      let(:direction) { "desc" }

      it "return sorted desc params by name" do
        expect(result.records.first).to eq company_1
        expect(result.records.last).to eq company_2
      end
    end
  end
end

```
### Object:
```ruby
class CompanyQuery
  include Pagy::Backend
  DEFAULT_ORDER = :asc
  DEFAULT_COLUMN = :created_at
  BASE_FIELDS = %i[nip phone]

  Result = Struct.new(:pagination, :records)

  attr_reader :params

  def initialize(params:)
    @params = params
  end

  def call
    companies = if params[:company_id].present?
      Company.where(id: params[:company_id])
    else
      base = base_fields_query_and_params
      base.empty? ? Company : Company.where(base[0], base[1])
    end
    companies = companies.where(registration_status: params[:companies_status]) if params[:companies_status].present?
    companies = companies.order(sort_column => sort_direction)

    @pagy, companies = pagy(companies, page: page, items: per_page)

    Result.new(@pagy, companies)
  end

  private

  # Does this look like we need ransack gem? Too much magic?
  def base_fields_query_and_params
    base_query = [].tap do |query|
      BASE_FIELDS.each { |f| query << "#{f} ILIKE :#{f}" if params[f].present? }
    end
    return [] if base_query.empty?

    query = (base_query.size > 1) ? base_query.join(" OR ") : base_query[0]
    query_params = {}.tap { |sql_params| BASE_FIELDS.each { |f| sql_params[f] = "%#{params[f]}%" } }
    [query, query_params]
  end

  def sort_column
    params[:sort_column].present? ? params[:sort_column].to_sym : DEFAULT_COLUMN
  end

  def sort_direction
    params[:sort_direction].present? ? params[:sort_direction].to_sym : DEFAULT_ORDER
  end

  def page
    params[:page] || 1
  end

  def per_page
    params[:per_page] || 25
  end
end

```

#### Mutations:
1:
```ruby
-      if params[f].present?
+      if true
```
and some other, similar mutations with this condition, all resulted in similar results, in general the if was redundant, since the query was sanitized later on anyway)

2:
```ruby
-  if base_query.empty?
-    return []
-  end
+  nil
```
and some other, similar mutations with this condition, showed two more useless conditional, which I removed and the spec still passed, so the object still did it job

3:
```ruby
-  query = if (base_query.size > 1)
-    base_query.join(" OR ")
-  else
-    base_query[0]
-  end
+  query = base_query.join(" OR ")
```
and some other, similar mutations with this condition, showed that the condition was redundant, since the join would not really do anything if there was one element only or zero

4:
```ruby
- Company.where(id: params[:company_id])
+ Company.where(id: params.fetch[:company_id])
```

I honestly just do not understand the point of this mutation. The where is not executed if params do not have the key or it is empty or it is nil. So in the cases when the param is present its fetch vs direct hash extraction.

I think the point of this mutation is to enforce using `fetch` instead of `#[]`. There is no performance gain, if anything according to benchmark fetch is slower (barely), fetch is more feature rich, yes, but this has no usage in this case. I think this mutation should be ignored honestly.

BUT since ignoring in mutation in mutant config seems to be broken I just have to comply... if it is not broken it would be nice if it would be documented, it seems to be, but the documentation for it is broken (syntax errors and just not working configs)

5:
```ruby
-    Company.where(base[0], base[1])
+    Company.where(base[0], base.fetch(1))
```

Same as above, if I wanted fetch to be enforced I would just configure rubocop to do it I do not see the point of this mutation, but I do comply to it, cause ignore config is not working...

6:

```ruby
 def page
-  params[:page] || 1
+  nil
end

def per_page
  -  params[:per_page] || 25
  +  params[:per_page]
end
```

Those mutations showed that it is useless to do default like that, since Pagy has the same default as we set here, so we can pass param value, cause if it is nil or empty, Pagy default will kick in and do the same


The rest of the mutations was all fetch business...

##### Changes:

Added two new specs:
```ruby
context 'with omitted fields' do
  let(:params) { {} }
  let!(:company_1) { create :company, nip: nip, name: name, registration_status: status, phone: phone }
  let!(:company_2) { create :company, nip: "0000000111", name: "taki sie nie wygeneruje", phone: "666 666 666" }
    
  it 'returns all companies, sorted by defaults' do
    expect(result.pagination.count).to eq 2
    expect(result.records.first).to eq company_1
  end
end

context 'with default pagination' do
  let(:per_page) { nil }
  let(:page) { nil }

  it 'returns results using default pagination' do
    expect(result.pagination.items).to eq 25
    expect(result.pagination.page).to eq 1
  end
end
```

And this is what the object ended up like:
```ruby
class CompanyQuery
  include Pagy::Backend
  DEFAULT_ORDER = :asc
  DEFAULT_COLUMN = :created_at
  BASE_FIELDS = %i[nip phone]

  Result = Struct.new(:pagination, :records)

  attr_reader :params

  def initialize(params:)
    @params = params
  end

  def call
    companies = if params[:company_id].present?
      Company.where(id: params.fetch(:company_id))
    else
      base = base_fields_query_and_params
      Company.where(base.fetch(0), base.fetch(1))
    end
    companies = companies.where(registration_status: params.fetch(:companies_status)) if params[:companies_status].present?
    companies = companies.order(sort_column => sort_direction)

    @pagy, companies = pagy(companies, page: params[:page], items: params[:per_page])

    Result.new(@pagy, companies)
  end

  private

  # Does this look like we need ransack gem? Too much magic?
  def base_fields_query_and_params
    base_query = [].tap do |query|
      BASE_FIELDS.each { |field| query << "#{field} ILIKE :#{field}" }
    end

    query = base_query.join(" OR ")
    query_params = {}.tap { |sql_params| BASE_FIELDS.each { |f| sql_params[f] = "%#{params[f]}%" } }
    [query, query_params]
  end

  def sort_column
    params[:sort_column].present? ? params.fetch(:sort_column).to_sym : DEFAULT_COLUMN
  end

  def sort_direction
    params[:sort_direction].present? ? params.fetch(:sort_direction).to_sym : DEFAULT_ORDER
  end
end
```
##### Conclusion:
It was nice to see some useless control statements, if that were redundant due to later sanitization etc.
It was rather annoying to deal with all the fetches with no actual benefits of them
