---
layout: post
title:  "Mutations - Part II"
date:   2023-03-28 10:55:36 +0200
description: 'Another two cases of mutation testing'
permalink: 'mutations_part_2'
author: kkp
---

# Mutation cases:

## Case 1:
### Test
```ruby
require "rails_helper"

RSpec.describe Aspects::CreateDefinition do
  subject(:result) { described_class.call(params: params) }
  let(:params) {
    {
      name: name,
      aspectable_id: aspect_1&.id,
      aspectable_type: aspect_1&.class&.name,
      photo: {
        image: photo
      }
    }
  }

  context "when given valid params" do
    let(:name) { "nazwa" }
    let(:photo) { Rack::Test::UploadedFile.new("spec/fixtures/images/genius.png") }
    let(:door_group) { create :door_group }
    let(:aspect_1) { create :aspect }

    it "create new definition" do
      expect { subject }.to change(AspectDefinition, :count).by 1
      definition = AspectDefinition.find_by(name: name)
      expect(definition).to be_present
      expect(definition.photo).to be_present
    end

    context "when definition with that name exist under a different aspect" do
      let(:aspect_2) { create :aspect }
      let!(:existing_definition) { create :aspect_definition, name: name, aspectable: aspect_2 }

      it "create new definition anyway" do
        expect { subject }.to change { AspectDefinition.count }.by 1
        expect(AspectDefinition.find_by(name: name)).to be_present
      end
    end
  end

  context "when given invalid params" do
    context "when empty params" do
      let(:name) { "" }
      let(:photo) { nil }
      let(:aspect_1) { nil }

      it "throw error" do
        expect { subject }.to change { AspectDefinition.count }.by 0
        expect(subject.errors).to include("name must be filled")
        expect(subject.errors).to include("image must be a hash")
        expect(subject.errors).to include("aspectable_id must be filled")
        expect(subject.errors).to include("aspectable_type must be filled")
      end
    end

    context "when values are busy" do
      let(:name) { "nazwa" }
      let(:photo) { Rack::Test::UploadedFile.new("spec/fixtures/images/genius.png") }
      let(:aspect_1) { create :aspect }
      let!(:existing_definition) { create :aspect_definition, name: name, aspectable: aspect_1 }
      it "throw error" do
        expect { subject }.to_not change { AspectDefinition.count }
        expect(subject.errors).to include("name already exists in this aspect")
      end
    end
  end
end

```
### Object
```ruby
module Aspects
  class CreateDefinition
    include BaseService
    attr_reader :params

    def initialize(params:)
      @params = params
    end

    def call
      parsed_params = ParsePhotoParams.call(params)
      validation = ::AspectDefinitionContract.new.call(parsed_params)
      if validation.success?
        parsed_params[:photo] = parsed_params[:original_photo]
        parsed_params.delete(:original_photo)
        definition = AspectDefinition.new(parsed_params)
        definition.save!
        add_result(definition: definition)
      else
        full_default_error_messages(validation)
      end
    end
  end
end
```
### Results of mutations
```shell
Mutations:       100
Results:         100
Kills:           95
Alive:           5
Timeouts:        0
Runtime:         51.60s
Killtime:        191.50s
Overhead:        -73.05%
Mutations/s:     1.94
Coverage:        95.00%
```

#### Mutations:
All mutations are from the same method, so posting the full one once, then posting just the differences to save space
1:
```ruby
 def call
   parsed_params = ParsePhotoParams.call(params)
-  validation = ::AspectDefinitionContract.new.call(parsed_params)
+  validation = AspectDefinitionContract.new.call(parsed_params)
   if validation.success?
     parsed_params[:photo] = parsed_params[:original_photo]
     parsed_params.delete(:original_photo)
     definition = AspectDefinition.new(parsed_params)
     definition.save!
     add_result(definition: definition)
   else
     full_default_error_messages(validation)
   end
 end
```

2:
```ruby
-    parsed_params[:photo] = parsed_params[:original_photo]
+    parsed_params[:photo] = parsed_params.fetch(:original_photo)
```

3:
```ruby
-    add_result(definition: definition)
+    nil
```

4:
```ruby
-    add_result(definition: definition)
+    add_result(definition__mutant__: definition)
```

5:
```ruby
-    add_result(definition: definition)
+    add_result(definition: nil)
```
##### Changes:
##### Conclusion:
1. Case 1 is just redundant `::` operator for zeitwerk.
2. Mutant is making this change a lot, I am not sure what is the reasoning for it. It wants to enforce using `fetch`. `{}[non_existent_key]` returns nil, but `{}.fetch(non_existent_key)` raises a `KeyError`. I was thinking a long time about why it always checks it. Does it want to enforce using code that throws an error is non-existing key is being accessed from hash? I checked for more detailed behaviour of fetch vs regular hash extraction:

```ruby
hash = {
  'a' => :some_value,
  'b' => nil,
  'c' => false
}

hash.fetch('a', :default_value) #=> :some_value
hash.fetch('b', :default_value) #=> nil
hash.fetch('c', :default_value) #=> false
hash.fetch('d', :default_value) #=> :default_value

(hash['a'] || :default_value) #=> :some_value
(hash['b'] || :default_value) #=> :default_value
(hash['c'] || :default_value) #=> :default_value
(hash['d'] || :default_value) #=> :default_value
```

So basically fetch is a shortcut to: `hash.key?(key) ? hash[key] : default` but with also an option to raise an error. Mutant however does not say anything about adding default value is his mutation.
The reasoning I see it could have behind it is something like: if key is missing from the hash, nil will get passed down the code stack, this should probably break the code but it does not, fetch would.
Having fetch and missing a key, should either break the spec/code or be handled (by default value or catch, rescue etc). I think this is a bit redundant mutation.
Anyway I added the `fetch` here. It is SOMEWHAT pointless in my opinion, I still don't really get that mutation and the point it is trying to prove, but `fetch` here is as good as regular `[key]` extraction due to the hash being validated before anyway.
3. Result was assigned and used in the place that used the service, but not in code, which it should have
4. As point 3, the same thing, result did not matter in the spec
5. As point 3, the same thing, result did not matter in the spec

## Przypadek 2:
### Test

```ruby
require "rails_helper"

RSpec.describe ParsePhotoParams do
  subject(:result) { described_class.call(params) }
  context "when given rails controller params" do
    let(:photo) { Rack::Test::UploadedFile.new("spec/fixtures/images/genius.png") }
    let_it_be(:uuid) { SecureRandom.uuid }
    let(:image_params) { ActionController::Parameters.new(image: photo).permit! }
    let(:params) {
      ActionController::Parameters.new(
        name: FFaker::Name.name,
        photo: image_params,
        aspectable_id: uuid,
        aspectable_type: "Aspect"
      ).permit!
    }

    it "parses to deep symbolized hash" do
      expect(subject[:original_photo]).to be_an_instance_of(Photo)
    end

    it "parses image params into a hash" do
      expect(subject).to eq({
        name: subject[:name],
        photo: {
          image: {
            object_class: image_params[:image].class,
            file_type: image_params[:image].content_type,
            file_size: image_params[:image].tempfile.size
          }
        },
        aspectable_id: uuid,
        aspectable_type: params[:aspectable_type],
        original_photo: subject[:original_photo]
      })
    end
  end
end
```

### Object

```ruby
class ParsePhotoParams
  class << self
    def call(params)
      photo = params.to_h.deep_symbolize_keys.dig(:photo, :image)
      photo.tempfile.open if photo.present? && photo.tempfile.closed?
      params[:original_photo] = Photo.new(image: photo) if photo
      params[:photo] = parsed_photo(photo) if photo
      params.to_h.deep_symbolize_keys
    end

    private

    def parsed_photo(photo)
      {
        image:
         {
           object_class: photo.class,
           file_type: photo.content_type,
           file_size: photo.tempfile.size
         }
      }
    end
  end
end
```
### Results of mutations
```shell
Mutations:       139
Results:         139
Kills:           104
Alive:           35
Timeouts:        0
Runtime:         67.80s
Killtime:        244.00s
Overhead:        -72.21%
Mutations/s:     2.05
Coverage:        74.82%
```

#### Mutations:
**There were 35 alive mutations, some of them were touching on the same problem, so I condensed them into smaller number of common mutations for the sake of brevity.**
1:
```ruby
-    params.to_h.deep_symbolize_keys
+    params.to_hash.deep_symbolize_keys
```
I'll be honest I thought `to_h` is an alias of `to_hash`. The difference is that `to_h` will return `ActiveSupport::HashWithIndifferentAccess` while the second on will give pure ruby hash. Both still remove the unpermitted params.
That being said I have zero understanding on why this should affect the specs. I tried adding this pattern to ignore in config file, but failed. Documentation shows some bad yml syntax and I was left confused about how to actually add a pattern to it
Worst thing though is that according to the mutation if I do this change, the test will not fail, but it does fail when I change it...
If the mutation is about some different problem than "with this change the test does not fail, it should have" then it is very unclear. I have spent quite some time on it, and it has only left me confused.
2:
Mutant got hooked on this line
```photo.tempfile.open```
This line exists because optionally a file can get closed in the process of getting to the piece of code in question. Most of the alive mutations were about it.
I closed the file in the spec to make it more viable in the test run of the code.
This has not done anything to the mutation (the file was closed now, coming in to the class from the spec).
Tried doing the same live.
The code crashed without the line that checks for closed and open the file (the file is later read when creating a Photo from it).
Don't understand this either. The error does not get raised in the mutation runs for some reason?
3:
```ruby
-    { image: { object_class: photo.class, file_type: photo.content_type, file_size: photo.tempfile.size } }
+    { image: { object_class: photo.class, file_type: photo.content_type, file_size: photo.size } }
```
`photo.size` is the same as `photo.tempfile.size` so we can skip the tempfile referencing
4:
```ruby
-      params[:original_photo] = Photo.new(image: photo)
+      params[:original_photo] = Photo.new(image: nil)
```
```ruby
-      params[:original_photo] = Photo.new(image: photo)
+      params[:original_photo] = Photo.new
```
`expect(subject[:original_photo]).to be_an_instance_of(Photo)` this was not sufficient enough. I added the check for image params

##### Changes:
The end spec is:
```ruby
require "rails_helper"

RSpec.describe ParsePhotoParams do
  subject(:result) { described_class.call(params) }
  let_it_be(:uuid) { SecureRandom.uuid }

  context "when given rails controller params" do
    let(:photo) { Rack::Test::UploadedFile.new("spec/fixtures/images/genius.png") }
    let(:image_params) { ActionController::Parameters.new(image: photo).permit! }
    let(:params) {
      ActionController::Parameters.new(
        name: FFaker::Name.name,
        photo: image_params,
        aspectable_id: uuid,
        aspectable_type: "Aspect"
      ).permit!
    }

    before do
      photo.tempfile.close
      photo.close
    end

    it "parses to deep symbolized hash" do
      expect(subject[:original_photo]).to be_an_instance_of(Photo)
      expect(
        JSON.parse(subject[:original_photo].image_data)["metadata"]["filename"]
      ).to eq image_params["image"].original_filename
    end

    it "parses image params into a hash" do
      expect(subject).to eq({
        name: subject[:name],
        photo: {
          image: {
            object_class: image_params[:image].class,
            file_type: image_params[:image].content_type,
            file_size: image_params[:image].tempfile.size
          }
        },
        aspectable_id: uuid,
        aspectable_type: params[:aspectable_type],
        original_photo: subject[:original_photo]
      })
    end
  end

  context "when not given photo param" do
    let(:photo) { Rack::Test::UploadedFile.new("spec/fixtures/images/genius.png") }
    let(:params) {
      ActionController::Parameters.new(name: FFaker::Name.name, aspectable_id: uuid, aspectable_type: "Aspect").permit!
    }

    it "does not have photo param" do
      expect(subject[:original_photo]).to eq nil
    end

    it "preserves other params" do
      expect(subject).to eq({
        name: subject[:name],
        aspectable_id: uuid,
        aspectable_type: params[:aspectable_type]
      })
    end
  end
end

```

And the end object is:
```ruby
class ParsePhotoParams
  class << self
    def call(params)
      photo = params.to_hash.deep_symbolize_keys.dig(:photo, :image)
      photo.tempfile.open if photo&.tempfile&.closed?
      params[:original_photo] = Photo.new(image: photo) if photo
      params[:photo] = parsed_photo(photo) if photo
      params.to_hash.deep_symbolize_keys
    end

    private

    def parsed_photo(photo)
      {
        image:
         {
           object_class: photo.class,
           file_type: photo.content_type,
           file_size: photo.size
         }
      }
    end
  end
end

```
##### Conclusion:
This was a weak run, the mutant brought a lot of confusion, but has also noticed the lackluster test of the resulting Photo object, and I learned difference between to_h and to_hash called on rails params object.
