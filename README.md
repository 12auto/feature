[![Gem Version](https://badge.fury.io/rb/feature.svg)](https://rubygems.org/gems/feature)
[![Travis-CI Build Status](https://travis-ci.org/mgsnova/feature.svg)](https://travis-ci.org/mgsnova/feature)
[![Coverage Status](http://img.shields.io/coveralls/mgsnova/feature/master.svg)](https://coveralls.io/r/mgsnova/feature)
[![Code Climate](https://codeclimate.com/github/mgsnova/feature.svg)](https://codeclimate.com/github/mgsnova/feature)
[![Inline docs](http://inch-ci.org/github/mgsnova/feature.svg)](http://inch-ci.org/github/mgsnova/feature)
[![Dependency Status](https://gemnasium.com/mgsnova/feature.svg)](https://gemnasium.com/mgsnova/feature)

# Feature

Feature is a battle-tested [feature toggle](http://martinfowler.com/bliki/FeatureToggle.html) library for ruby.

The feature toggle functionality has to be configured by feature repositories. A feature repository simply provides lists of active features (symbols!). Unknown features are assumed inactive.

With this approach Feature is highly configurable and not bound to a specific kind of configuration.

**NOTE:** Ruby 1.9 is supported explicitly only until version 1.2.0. Later version may require 2+.

**NOTE:** Ruby 1.8 is only supported until version 0.7.0. Later Versions require Ruby 1.9+.

**NOTE:** Using feature with ActiveRecord and Rails 3 MAY work. Version 1.1.0 supports Rails 3.

## Installation

        gem install feature

## How to use

* Setup Feature
    * Create a repository (for more infos about configuration backends, see section below)

            require 'feature'
            repo = Feature::Repository::SimpleRepository.new

    * Set repository to Feature

            Feature.set_repository(repo)

* Use Feature in your production code

        Feature.active?(:feature_name) # => true/false

        Feature.inactive?(:feature_name) # => true/false

        Feature.with(:feature_name) do
          # code
        end

        Feature.without(:feature_name) do
          # code
        end

        # this returns value_true if :feature_name is active, otherwise value_false
        Feature.switch(:feature_name, value_true, value_false) 

        # switch may also take Procs that will be evaluated and it's result returned.
        Feature.switch(:feature_name, -> { code... }, -> { code... })

* Use Feature in your test code (for reliable testing of feature depending code)

        require 'feature/testing'

        Feature.run_with_activated(:feature) do
          # your test code
        end

        # you also can give a list of features
        Feature.run_with_deactivated(:feature, :another_feature) do
          # your test code
        end

* Feature-toggle caching

    * By default, Feature will lazy-load the active features from the
      underlying repository the first time you try to check whether a
      feature is set or not. 

    * Subsequent calls to Feature will access the cached in-memory
      representation of the list of features. So changes to toggles in the 
      underlying repository would not be reflected in the application
      until you restart the application or manally call 

            Feature.refresh!

    * You can optionally pass in true as a second argument on
      set_repository, to force Feature to auto-refresh the feature list
      on every feature-toggle check you make.

            Feature.set_repository(your_repository, true) 

## How to setup different backends

### SimpleRepository (in-memory)

        # File: Gemfile
        gem 'feature'

        # setup code
        require 'feature'

        repo = Feature::Repository::SimpleRepository.new
        repo.add_active_feature :be_nice

        Feature.set_repository repo

### RedisRepository (features configured in redis server)

        # See here to learn how to configure redis: https://github.com/redis/redis-rb

        # File: Gemfile
        gem 'feature'
        gem 'redis'

        # setup code (or Rails initializer: config/initializers/feature.rb)
        require 'feature'

        # "feature_toggles" will be the key name in redis
        repo = Feature::Repository::RedisRepository.new("feature_toggles")
        Feature.set_repository repo

        # add/toggle features in Redis
        Redis.current.hset("feature_toggles", "ActiveFeature", true)
        Redis.current.hset("feature_toggles", "InActiveFeature", false)

### YamlRepository (features configured in static yml file)

        # File: Gemfile
        gem 'feature'

        # File: config/feature.yml
        features:
            an_active_feature: true
            an_inactive_feature: false

        # setup code (or Rails initializer: config/initializers/feature.rb)
        repo = Feature::Repository::YamlRepository.new("#{Rails.root}/config/feature.yml")
        Feature.set_repository repo

You may also specify a Rails environment to use a new feature in development and test, but not production:

        # File: config/feature.yml
        test:
            features:
                a_new_feature: true
        production:
            features:
                a_new_feature: false

        # File: config/initializers/feature.rb
        repo = Feature::Repository::YamlRepository.new("#{Rails.root}/config/feature.yml", Rails.env)
        Feature.set_repository repo

### ActiveRecordRepository (features configured in a database) using Rails

        # File: Gemfile
        gem 'feature'

        # Run generator and migrations
        $ rails g feature:install
        $ rake db:migrate

        # Add Features to table FeaturesToggle for example in
        # File: db/schema.rb
        FeatureToggle.create!(name: "ActiveFeature", active: true)
        FeatureToggle.create!(name: "InActiveFeature", active: false)

        # or in initializer
        # File: config/initializers/feature.rb
        repo.add_active_feature(:active_feature)
