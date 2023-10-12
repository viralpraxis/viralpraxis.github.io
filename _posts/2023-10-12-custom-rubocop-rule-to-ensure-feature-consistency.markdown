### Problem

Consider the following model:

```ruby
class User < ApplicationRecord
   # some definitions

   validates :address, presence: true

   scope :with_address, ->(address) { where(address: normalize_address(address)) }

   def has_address?(address)
     address == normalize_address(address)
   end

   # some definitions

   private

     def normalize_address(address)
       # do some normalization routine
     end
end
```

We have a model `User` which has an associated attribute `address`. We want the user's address to be normalized before persisting in the database.

Furthermore, we would probably want the following:

- be able to query all `users` with provided *non-normalized* address (via `User.with_address` scope)
- be able to ask a single `User` record if it has provided a *non-normalized* address (via `User#has_address?` instance method).

The suggested implementation works well; both methods are logically correct and implement required features.
What happens if we're asked to remove the "fuzzy match" feature and return exact address matches both in scope and instance method?

Obviously, we have to update both method implementations, and we **Must** do it simultaneously; i.e., changing one without changing another would lead to inconsistent business logic implementation (i.e., a bug).

`ActiveRecord` models tend to be big, it's not that hard to forget there's another piece of code that implements the same feature. Even if we use some sort of per-model separated `ActiveSupport` concern that stores all address-search-related functionality on one file, nothing guarantees all the changes will be applied at once.

In other words, there is a single **feature** "fuzzy user address search", which is split across two methods; to ensure consistency, both methods should be modified at once (i.e., have the same **revision**).

### Solution

One solution is to use a custom `Rubocop` cop that will somehow track related methods and their revisions.

We'll have to mark each of these grouped methods with some magic comment, for instance:

```ruby
# ...

# [feature-revision] id: user-address-search, revision: 1
scope :with_address, ->(address) { where(address: normalize_address(address)) }

# [feature-revision] id: user-address-search, revision: 1
def has_address?(address)
  address == normalize_address(address)
end

# ...
```

The process of implementing custom cops is covered in the [official rubocop documentation](https://docs.rubocop.org/rubocop/development.html). There's also a [great article by Evil Martians](https://evilmartians.com/chronicles/custom-cops-for-rubocop-an-emergency-service-for-your-codebase).

Let's start with defining a new instance of `RuboCop::Cop::Base` instance:

```ruby
# cop/lint/feature_revisions_consistency.rb

# frozen_string_literal: true

module RuboCop
  module Cop
    module Lint
      class FeatureRevisionsConsistency < Base
        MSG = "Unmatched feature revision"

        def on_new_investigation
          return unless processed_source.ast

          investigate
        end

        private

          def investigate
            # do some actual investigation here
          end
      end
    end
  end
end
```

We override `on_new_investigation` entry point method, which will run the actual investigation if source code AST exists.

Next, we'll add a method that takes an AST and extracts all the comments whose contents match our regular expression:

```ruby
class FeatureRevisionsConsistency < Base
  # ...
  DEFAULT_OPTIONS = {
    magic_comment_regexp: /^\s*#\s*\[feature-revision\]\s*id:\s*(?<id>\S+),\s*revision:\s*(?<revision>\S+)\s*$/
  }.freeze

  # We'll store suspicious comments in custom `TargetComment` structure.
  # Consider using new `Data` ruby object-value :)
  OffendingComment = Struct.new(:node, :data, keyword_init: true) do
    extend Forwardable

    def_delegators :data, :id, :revision

    def initialize(node:, data:)
      super node: node, data: OpenStruct.new(data.named_captures)
    end
  end

  private

    def source_offending_comments(source)
      source.comments.filter_map do |comment|
        if (match_data = magic_comment_regexp.match(comment.text))
          OffendingComment.new(node: comment, data: match_data)
        end
      end
    end

    # We might want to override the default regular expression via the configuration option `magic_comment_regexp`.
    def magic_comment_regexp
      cop_config.fetch("MagicCommentRegExp", DEFAULT_OPTIONS.fetch(:magic_comment_regexp))
    end
  # ...
end

```
The source file investigation will work as follows:

- initialize some *persistent* container that will store target comments during file investigation.
- on each new comment, if there's a comment in container with the same `id` with another `revision` -- consider it a `failed` comment group.

There's one problem here. Naturally, RuboCop doesn't provide a native way to persist some data between single file investigations. All checks are designed to be file-locale, i.e., there's no natural way to implement a custom rule that will work over a set of files. See this [issue](https://github.com/rubocop/rubocop/issues/7968) for details.

To resolve this issue, we'll add a new class variable `@@__data`:

```ruby
@@__data = Hash.new { Set.new }
```

Now we can store target comments in `@@__data`:

```ruby
class FeatureRevisionsConsistency < Base
  def investigate
    source_offending_comments(processed_source).each do |comment|
      if has_offending_comment_in_same_group?(comment)
        add_offense comment.node, severity: :error
      end

      store_offending_comment(comment)
    end
  end

  private

    def has_offending_comment_in_same_group?(comment)
      !@@__data[comment.id].empty? && !@@__data[comment.id].include?(comment.revision)
    end

    def store_offending_comment(comment)
      @@__data[comment.id] = @@__data[comment.id].add(comment.revision)
    end
end
```

Note that we simply call `add_offense` RuboCop API in order to report feature revision mismatches.

Let's wrap it up:

```ruby
module RuboCop
  module Cop
    module Lint
      class FeatureRevisionsConsistency < Base
        OffendingComment = Struct.new(:node, :data, keyword_init: true) do
          extend Forwardable

          def_delegators :data, :id, :revision

          def initialize(node:, data:)
            super node: node, data: OpenStruct.new(data.named_captures)
          end
        end

        DEFAULT_OPTIONS = {
          magic_comment_regexp: /^\s*#\s*\[feature-revision\]\s*id:\s*(?<id>\S+),\s*revision:\s*(?<revision>\S+)\s*$/
        }.freeze

        MSG = "Unmatched feature revision"

        include RangeHelp

        @@__data = Hash.new { Set.new }

        def on_new_investigation
          return unless processed_source.ast

          investigate
        end

        private

          def investigate
            source_offending_comments(processed_source).each do |comment|
              if has_offending_comment_in_same_group?(comment)
                add_offense comment.node, severity: :error
              end

              store_offending_comment(comment)
            end
          end

          def source_offending_comments(source)
            source.comments.filter_map do |comment|
              if (match_data = magic_comment_regexp.match(comment.text))
                OffendingComment.new(node: comment, data: match_data)
              end
            end
          end

          def has_offending_comment_in_same_group?(comment)
            @@__mutex.synchronize do
              !@@__data[comment.id].empty? && !@@__data[comment.id].include?(comment.revision) # rubocop:disable Rails/NegateInclude
            end
          end

          def store_offending_comment(comment)
            @@__mutex.synchronize do
              @@__data[comment.id] = @@__data[comment.id].add(comment.revision)
            end
          end

          def magic_comment_regexp
            cop_config.fetch("MagicCommentRegExp", DEFAULT_OPTIONS.fetch(:magic_comment_regexp))
          end
      end
    end
  end
end
```

Let's test it: (you'll have to add  `- ./cop/lint/feature_revisions_consistency` to `require` rubocop configuration directive)

```ruby
# frozen_string_literal: true

# user.rb

class User < ApplicationRecord
  validates :address, presence: true

  # [feature-revision] id: user-address-search, revision: 2
  scope :with_address, ->(address) { where(address: normalize_address(address)) }

  # [feature-revision] id: user-address-search, revision: 1
  def has_address?(address)
    address == normalize_address(address)
  end

  private

    def normalize_address(address)
      address.strip
    end
end
```

```bash
bundle exec rubocop user.rb --only Lint/FeaturesRevisionConsistency
Inspecting 1 file
E

Offenses:

user.rb:9:3: E: Lint/FeaturesRevisionConsistency: Unmatched feature revision
  # [feature-revision] id: user-address-search, revision: 1
  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1 file inspected, 1 offense detected
```

### Limitations

There are some limitations and problems with the provided solution.

1. If there's a group of two comments with revision mismatches, only one of them will be reported. That's not a big issue, because that's definitely enough to figure out what's wrong and fix the issue. Once again, that's due to a lack of native context-sharing cops.
2. RuboCop usually runs in parallel (using `nproc` threads — count of cores on your machine). It means that we have to protect
class variable `@@__data` with mutex/RWmutex.
3. RuboCop uses aggressive caching, and it kind of breaks when using context-sharing cops. For simplicity, we can simply force RuboCop not to use any cache at all:

```ruby
class FeatureRevisionsConsistency < Base
  # ...

  def external_dependency_checksum
    @external_dependency_checksum ||= SecureRandom.hex : ""
  end

  # ...
end
```

Here we override `external_depdency_checksum` which is used by RuboCop to determine if it should invalidate the cache due to some external files changing. Note that it will significantly slow down rubocop'ing, so I'd recommend running this cop on CI only (or, manually, "on demand" before committing). To do so, we can use an environment variable:

```ruby
# frozen_string_literal: true

class FeatureRevisionsConsistency < Base
  # ...

  ENV_KEY_COP_ENABLED = "RUBOCOP_RUN_CACHELESS_COPS"

  def external_dependency_checksum
    @external_dependency_checksum ||= ENV.key?(ENV_KEY_COP_ENABLED) ? SecureRandom.hex : ""
  end

  # ...
end
```

And enable the cop only if `RUBOCOP_RUN_CACHELESS_COPS` is present:

```yaml
# .rubocop.yml

Lint/FeaturesRevisionConsistency:
  Enabled: <%= ENV.key?("RUBOCOP_RUN_CACHELESS_COPS") %>
```

Now you can run all the cops (including cacheless) via `RUBOCOP_RUN_CACHELESS_COPS=true bundle exec rubocop`.

There are few more step's you'd probably want to do:

1. Add custom cop documentation, as recommended by [documentation](https://docs.rubocop.org/rubocop/development.html#documentation).
2. Add some [tests](https://docs.rubocop.org/rubocop/development.html#documentation).

That's all! ☺️

Full source code can be found [here](https://github.com/viralpraxis/rubocop-feature-revisions-consistency)
