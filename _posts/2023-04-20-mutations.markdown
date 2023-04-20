---
layout: post
title:  "Mutations - Part IV"
date:   2023-04-20 09:25:16 +0200
description: 'Yet another case of mutation testing'
permalink: 'mutations_part_4'
author: kkp
---

# Mutation cases:

## Case 1:
### Test:
```ruby
require "rails_helper"

RSpec.describe Products::CloneProduct do
  include Dry::Effects::Handler.Reader(:product_process)
  subject(:result) do
    with_product_process(ProductProcess.new(nil, params)) { described_class.call }
  end

  let_it_be(:aspect_1) { create :aspect, name: "Ościeżnica", position: 5, aspect_type: :frame }
  let_it_be(:aspect_2) { create :aspect, name: "Doświetlenie", position: 6, aspect_type: :lighting }
  let_it_be(:aspect_3) { create :aspect, name: "Naświetlenie", position: 7, aspect_type: :lighting }
  let_it_be(:door_group) { create :door_group, door_type: :interior, name: "Drzwi wewnętrzne", aspects: [aspect_1, aspect_2] }
  let_it_be(:aspect_definition1) { create :aspect_definition, aspectable: aspect_1 }
  let_it_be(:aspect_definition2) { create :aspect_definition, aspectable: aspect_2 }
  let_it_be(:product) { create :product, name: "Nimbus", door_group_id: door_group.id }
  let_it_be(:excluded_aspect1) { create :excluded_aspect, eliminated_aspect_id: aspect_2.id, aspect_id: aspect_1.id, product_id: product.id }
  let_it_be(:excluded_aspect2) { create :excluded_aspect, eliminated_aspect_id: aspect_3.id, aspect_id: aspect_1.id, product_id: product.id }
  let_it_be(:product_definition1) { create :product_definition, aspect_definition_id: aspect_definition1.id, product_id: product.id }
  let_it_be(:product_definition2) { create :product_definition, aspect_definition_id: aspect_definition2.id, product_id: product.id }
  let_it_be(:price1) { create :price, amount_price: 0.34e2, product_definition_id: product_definition1.id }
  let_it_be(:photo) { create :photo, photoable: product_definition1, image: Rack::Test::UploadedFile.new("spec/fixtures/images/ohno.png") }
  let_it_be(:number_of_clones) { 8 }
  let_it_be(:params) { {product_id: product.id, number_of_clones: number_of_clones} }

  # The gem for cloning handles relationships, which insert_all does not, the cost being n+1 queries
  before { Prosopite.pause }
  after { Prosopite.resume }
  context "with provided number of product clones" do
    context "create the same products" do
      it "provided times" do
        expect(subject.result[:product_clones].count).to eq(8)
      end

      it "to change clones status on unpublished" do
        subject.result[:product_clones].map { |clone|
          expect(clone.published).to be false
        }
      end

      it "with copy number in the name" do
        subject.result[:product_clones].each { |clone|
          expect(clone.name).to eq("[Kopia_#{subject.result[:product_clones].index(Product.find(clone.id)) + 1}] #{product.name}")
        }
      end

      it "with the same attributes" do
        expect(subject.result[:product_clones].map { |clone|
          clone.attributes.except("id", "created_at", "updated_at", "name")
        }).to all(contain_exactly(*product.attributes.except("id", "created_at", "updated_at", "name")))
      end

      it "with attached photo" do
        expect(subject.result[:product_clones].map { |clone|
          clone.product_definitions.count
        }.sum).to eq(16)
        expect(subject.result[:product_clones].all? do |clone|
          clone.product_definitions.find_by(aspect_definition_id: aspect_definition1.id).photo.present?
          clone.product_definitions.find_by(aspect_definition_id: aspect_definition2.id).photo.nil?
        end).to be true
      end

      it "with the same relations with aspect_definitions" do
        expect(subject.result[:product_clones].map { |clone|
          clone.aspect_definitions.count
        }.sum).to eq(16)
      end

      it "with product definitions price" do
        subject
        expect(subject.result[:product_clones].map { |clone|
          clone.product_definitions.where(aspect_definition_id: aspect_definition1.id).map do |definition|
            definition.price.amount_price
          end
        }).to all(contain_exactly(*price1.amount_price))
      end

      it "with excluded aspects" do
        expect(subject.result[:product_clones].map { |clone|
          clone.excluded_aspects.count
        }.sum).to eq(16)
      end

      it "do not create new aspect_definitions" do
        subject
        expect(AspectDefinition.pluck(:id)).to contain_exactly(aspect_definition1.id, aspect_definition2.id)
      end
    end
  end
end
```

### Object:
```ruby
module Products
  class CloneProduct
    include Dry::Effects::Reader(:product_process)
    include BaseService

    def call
      clones = []
      product_process.params.number_of_clones.times.with_index(1) do |_x, idx|
        clones << product_process.product.deep_clone(include: [:excluded_aspects, {product_definitions: [:photo, :price]}]) do |original, kopy|
          kopy.assign_attributes(name: "[Kopia_#{idx}] " + original.name) if original.instance_of?(Product)
        end
      end
      clones.each(&:save!)
      add_result(product_clones: clones)
    end
  end
end
```

#### Mutations:
1:

 ```ruby
  - clones << product_process.product.deep_clone(include: [:excluded_aspects, { product_definitions: [:photo, :price] }]) { |original, kopy|
  + clones << product_process.product.deep_clone(include: [:excluded_aspects, { product_definitions: [:price] }]) { |original, kopy|
```
Sensible, a spec was missing that checks if photos are cloned correctly

2:
```ruby
-      if original.is_a?(Product)
+      if original.instance_of?(Product)
```
`instance_of?` is a better fit here indeed

3:
```ruby
-        kopy.assign_attributes(published: false, name: "[Kopia_#{idx}] " + original.name)
+        kopy.assign_attributes(name: "[Kopia_#{idx}] " + original.name)
```
`published:false` is set by default in the schema, along with `null:false`, so it's not needed here

4:
```ruby
-  clones.map(&:save!)
+  clones.each(&:save!)
```
Also makes sense, since the map is not used for anything

##### Changes:
The object changes were:
```ruby
-          kopy.assign_attributes(published: false, name: "[Kopia_#{idx}] " + original.name) if original.is_a?(Product)
+          kopy.assign_attributes(name: "[Kopia_#{idx}] " + original.name) if original.instance_of?(Product)
         end
       end
-      clones.map(&:save!)
+      clones.each(&:save!)
```

And an added check:
```ruby
+        expect(subject.result[:product_clones].all? do |clone|
+          clone.product_definitions.find_by(aspect_definition_id: aspect_definition1.id).photo.present? &&
+          clone.product_definitions.find_by(aspect_definition_id: aspect_definition2.id).photo.nil?
+        end).to be true
```

##### Conclusion:
This time a more sensible, helpful run that spotted a missing check, the rest was kinda more of an rubocop suggestion
