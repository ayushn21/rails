**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON <https://guides.rubyonrails.org>.**

Configuring Rails Applications
==============================

This guide covers the configuration and initialization features available to Rails applications.

After reading this guide, you will know:

* How to adjust the behavior of your Rails applications.
* How to add additional code to be run at application start time.

--------------------------------------------------------------------------------

The `Configuration` Object
--------------------------

Rails' configuration settings are all held in an instance of
[`Rails::Application::Configuration`](https://api.rubyonrails.org/classes/Rails/Application/Configuration.html).
It's instantiated when Rails boots, and can be accessed anywhere in
the application with `Rails.app.config` or `Rails.configuration`.

### Applying Configuration Settings

Rails offers three standard locations to add or modify values on the configuration object:

1. `config/application.rb`
2. Environment-specific configuration files
3. Initializer files

#### `config/application.rb`

The `config/application.rb` can be thought of as the entry point to your
Rails app. The configuration object is available via `config` in
the application class:

```ruby#14,15
require_relative "boot"

require "rails/all"

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

module MyRailsApp
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 8.1

    config.autoload_lib(ignore: %w[assets tasks])
    config.time_zone = "Central Time (US & Canada)"
  end
end
```

#### Environment-specific configuration files

Each Rails environment has a file for environment-specific settings:

* `config/environments/production.rb`
* `config/environments/development.rb`
* `config/environments/test.rb`

These files are structured in the same way:

```ruby
Rails.application.configure do
  # Settings specified here will take precedence over those in config/application.rb.

  config.eager_load = true
  # ...
end
```

The configuration object can be accessed using `config` inside the
`configure` block. You may also add additional arbitrary code outside
the `configure` block which will be run when Rails boots in a
specific environment.

This file is useful for defining settings that are environment
dependent — such as the logger, SMTP servers, the cache store, and
error handling.

#### Initializers

All Ruby files under `config/initializers` are loaded by Rails when it
boots. Create files in this folder to apply custom settings. The
configuration object isn't automatically available in these files —
use `Rails.app.configure`, or access the object directly
using `Rails.app.config`.

```ruby
# config/initializers/cookies.rb

# Directly accessing the configuration object
Rails.app.config.action_dispatch.signed_cookie_digest = "SHA256"

# Using a block to access the configuration object
Rails.app.configure do
  config.action_dispatch.signed_cookie_salt = "a new salt"

  # ...
end
```

Initializer files are a great place for custom app-specific
intialization and configuration code, as you can logically group
settings in multiple files. Some examples of components you may use
an initializer file to configure are: cookies, sessions, inflections,
and Rack middleware.

Initializer files are sorted and then loaded one-by-one, but the load
order isn't guaranteed. If an initializer has code that relies
on code in another initializer, combine them into a single file. This
makes the dependencies explicit and hence easier to reason about.

WARNING: Manually loading initializers with `require` is not recommended,
as it will cause the file to be loaded twice.

If your application needs to run some code before Rails itself is
loaded, put it above `require "rails/all"` in `config/application.rb`.

You can learn more about the exact load order of the files described
above, and details about the Rails boot process in the
[initialization guide](initialization.html).

### Custom Configuration

You can add custom configuration settings to the configuration object.

```ruby
Rails.app.config.my_custom_setting = true
```

When defining a nested configuration, use the `config.x` namespace:

```ruby
Rails.app.config.x.payment_processing.schedule = :daily
Rails.app.config.x.payment_processing.retries  = 3
```

These options are then available through the configuration object:

```ruby
Rails.app.config.my_custom_setting             # => true

Rails.app.config.x.payment_processing.schedule # => :daily
Rails.app.config.x.payment_processing.retries  # => 3
Rails.app.config.x.payment_processing.not_set  # => nil
```

You can define environment specific configuration options in a YAML
file and load them in your Rails app
using [`Rails.app.config_for`](https://api.rubyonrails.org/classes/Rails/Application.html#method-i-config_for). Rails will detect the
environment and automatically load the appropriate settings.

```yaml
# config/payment.yml

production:
  environment: production
  merchant_id: production_merchant_id
  public_key:  production_public_key
  private_key: production_private_key

development:
  environment: sandbox
  merchant_id: development_merchant_id
  public_key:  development_public_key
  private_key: development_private_key

shared:
  adapter: processor_name
```

The YAML file must be in the `config/` folder, and contain top-level
keys corresponding to the environments. A `shared` key can contain common
settings which will be merged with the environment-specific options when
Rails loads the file.

Load the file in your `application.rb`, or in an initializer:

```ruby
# config/application.rb

module MyApp
  class Application < Rails::Application
    config.payment = config_for(:payment)
  end
end
```

```ruby
# config/initializers/payments.rb

Rails.app.config.payment = Rails.app.config_for(:payment)
```

You can access the settings via the
configuration object:

```ruby
# In the `development` environment
Rails.app.config.payment.merchant_id
# => development_merchant_id

# In the `production` environment
Rails.app.config.payment.merchant_id
# => production_merchant_id
```

Initialization Events and Hooks
-------------------------------

Sometimes you might need to run code at specific times during the
initialization process — for example, to apply a
configuration setting after a gem has initialized.

Since the load order of your application's initializers is not
guaranteed, nor is the fact that they will be run after all gem's have initialized, Rails provides a number of initialization
events. You can hook into these events to run code at specific times.

The events are listed below in the order in which they are run:

* `before_configuration`: Run when your application class in `config/application.rb` is
loaded, before the class body is executed. Engines may use this hook to run code
before the application itself gets configured.

* `before_initialize`: Run directly before the Railties and Engines are initialized.

* `to_prepare`: Run after the initializers are run for all Railties (including the application
itself) and Engines, and after the middleware stack is built, but before eager loading. More
importantly, it will run upon every code reload in `development`, but only
once (during boot-up) in `production` and `test`.

* `before_eager_load`: Run directly before eager loading occurs. Eager loading is enabled
in `production` by default, but disabled in other environments.

* `after_initialize`: Run after the application has been initialized, and after the
files in `config/initializers/` are executed.

You can access these hooks via the Rails configuration object:

```ruby
module MyRailsApp
  class Application < Rails::Application
    config.before_configuration do
      # ...
    end

    config.before_initialize do
      # ...
    end

    config.to_prepare do
      # ...
    end

    config.before_eager_load do
      # ...
    end

    config.after_initialize do
      # ...
    end
  end
end
```

or

```ruby
Rails.app.config.before_configuration do
  # ...
end

Rails.app.config.before_initialize do
  # ...
end

Rails.app.config.to_prepare do
  # ...
end

Rails.app.config.before_eager_load do
  # ...
end

Rails.app.config.after_initialize do
  # ...
end
```

You can define multiple blocks for each hook, and they'll be invoked
sequentially. For example, you may define a `to_prepare` block
in `config/application.rb`, and another in an initializer file, and
they'll both be run one after the other.

WARNING: Some parts of your application, notably routing, are not yet set
up at the point where the `after_initialize` block is called.

### Load Hooks

Rails is modular, and composed of several frameworks such as Active
Record, Action Dispatch etc. Load hooks allow you to hook into the
loading of these frameworks to run your own initialization code. This
way, your application won't cause conflicts by arbitrarily
triggering a framework to load during initialization, or try to invoke
code from a framework that hasn't been loaded yet.

Use `ActiveSupport.on_load` to define a load hook:

```ruby
# Called after Active Record has been loaded. The block is evaluated
# in the context of the target (in this case `ActiveRecord::Base`),
# hence we can include a module directly.
ActiveSupport.on_load(:active_record) do
  include MyActiveRecordHelper
end
```

Here's another example of using a load hook to apply a configuration setting in
Active Record:

```ruby
ActiveSupport.on_load(:active_record) do
  self.include_root_in_json = true
end
```

Search the Rails source code for `ActiveSupport.run_load_hooks` to find
all the components that support lazy load hooks, the name of their
hooks, when they're invoked, and the object within which the blocks
are evaluated. All available hooks are also
[listed below](#list-of-load-hooks)

For example, if you search for
`ActiveSupport.run_load_hooks(:active_record`, you'll find it in
`activerecord/lib/activerecord/base.rb` as:

```ruby
ActiveSupport.run_load_hooks(:active_record, Base)
```

You can see that `Base` is passed as an argument when the hooks are
run, meaning that's the hooks will be evaluated within the context
of that object.

#### List of Load Hooks

Here's a list of all load hooks triggered by Rails and its components.

| Class                                | Hook                                 |
| -------------------------------------| ------------------------------------ |
| `ActionCable`                        | `action_cable`                       |
| `ActionCable::Channel::Base`         | `action_cable_channel`               |
| `ActionCable::Connection::Base`      | `action_cable_connection`            |
| `ActionCable::Connection::TestCase`  | `action_cable_connection_test_case`  |
| `ActionController::API`              | `action_controller_api`              |
| `ActionController::API`              | `action_controller`                  |
| `ActionController::Base`             | `action_controller_base`             |
| `ActionController::Base`             | `action_controller`                  |
| `ActionController::Live`             | `action_controller_live`             |
| `ActionController::TestCase`         | `action_controller_test_case`        |
| `ActionDispatch::IntegrationTest`    | `action_dispatch_integration_test`   |
| `ActionDispatch::Response`           | `action_dispatch_response`           |
| `ActionDispatch::Request`            | `action_dispatch_request`            |
| `ActionDispatch::SystemTestCase`     | `action_dispatch_system_test_case`   |
| `ActionMailbox::Base`                | `action_mailbox`                     |
| `ActionMailbox::InboundEmail`        | `action_mailbox_inbound_email`       |
| `ActionMailbox::Record`              | `action_mailbox_record`              |
| `ActionMailbox::TestCase`            | `action_mailbox_test_case`           |
| `ActionMailer::Base`                 | `action_mailer`                      |
| `ActionMailer::TestCase`             | `action_mailer_test_case`            |
| `ActionText::Content`                | `action_text_content`                |
| `ActionText::Record`                 | `action_text_record`                 |
| `ActionText::RichText`               | `action_text_rich_text`              |
| `ActionText::EncryptedRichText`      | `action_text_encrypted_rich_text`    |
| `ActionView::Base`                   | `action_view`                        |
| `ActionView::TestCase`               | `action_view_test_case`              |
| `ActiveJob::Base`                    | `active_job`                         |
| `ActiveJob::TestCase`                | `active_job_test_case`               |
| `ActiveModel::Model`                 | `active_model`                       |
| `ActiveModel::Translation`           | `active_model_translation`           |
| `ActiveRecord::Base`                 | `active_record`                      |
| `ActiveRecord::DatabaseConfigurations` | `active_record_database_configurations` |
| `ActiveRecord::Encryption`           | `active_record_encryption`           |
| `ActiveRecord::TestFixtures`         | `active_record_fixtures`             |
| `ActiveRecord::ConnectionAdapters::PostgreSQLAdapter`    | `active_record_postgresqladapter`    |
| `ActiveRecord::ConnectionAdapters::Mysql2Adapter`        | `active_record_mysql2adapter`        |
| `ActiveRecord::ConnectionAdapters::TrilogyAdapter`       | `active_record_trilogyadapter`       |
| `ActiveRecord::ConnectionAdapters::SQLite3Adapter`       | `active_record_sqlite3adapter`       |
| `ActiveStorage::Attachment`          | `active_storage_attachment`          |
| `ActiveStorage::VariantRecord`       | `active_storage_variant_record`      |
| `ActiveStorage::Blob`                | `active_storage_blob`                |
| `ActiveStorage::Record`              | `active_storage_record`              |
| `ActiveSupport::TestCase`            | `active_support_test_case`           |
| `i18n`                               | `i18n`                               |


Rails Environment Settings
--------------------------

Some parts of Rails can be configured externally by defining environment variables. The
following environment variables are read by various parts of Rails:

* `ENV["RAILS_ENV"]` defines the Rails environment (`production`, `development`, or `test`).

* `ENV["RAILS_RELATIVE_URL_ROOT"]` is used by the routing code to recognize URLs
when you [deploy your application to a subdirectory](#deploy-to-a-subdirectory-relative-url-root).

* `ENV["RAILS_CACHE_ID"]` and `ENV["RAILS_APP_VERSION"]` are used to generate expanded
cache keys in Rails' caching code. This allows you to have multiple separate caches
for the same application.

Configuring Rails Components
----------------------------

A variety of aspects withing Rails and its contituent components can be
configured using the configuration object. This section lists all the
options available for use, and what they control.

Some components such as Action Mailer may hold their settings under their
own namespace — for example `ActionMailer::Base.options`. Never use this
API directly. These components integrate with Rails configuration object
to ensure settings are loaded correctly. Always use the Rails
configuration object instead: `Rails.app.config.action_mailer.options`.

NOTE: If you need to apply configuration directly to a class, use a
[lazy load hook](https://api.rubyonrails.org/classes/ActiveSupport/LazyLoadHooks.html)
in an initializer to avoid autoloading the class before
initialization has completed.

Each version of Rails loads of number of defaults for the settings
listen below. This can be seen in your `application.rb`:

```ruby
module MyRailsApp
  class Application < Rails::Application
    # Load default configuration for Rails 8.1
    config.load_defaults 8.1

    # ...
  end
end
```

This design gives enables you to load the default settings for an
older version of Rails than the one you're running, if you need to.
This makes Rails upgrades easier as you can incrementally make the app
changes required for compatibility with the latest defaults without
being stuck on any particular Rails version.

The complete list of default values for all Rails versions can be found
in the [Default Configuration Values](default_configuration_values.md) guide.

### General Configuration Options

The following methods are used to configure a `Rails::Railtie` object,
such as a subclass of `Rails::Engine` or `Rails::Application`.

#### `config.action_on_early_load_hook`

Controls what happens when a load hook is violated before the Rails application is initialized.
The value is `:log` by default, which will log when a load hook is invoked early. The value can alternatively be `:raise`, which will raise a `LoadError` instead of logging.

#### `config.add_autoload_paths_to_load_path`

Says whether autoload paths have to be added to `$LOAD_PATH`. It is recommended to be set to `false` in `:zeitwerk` mode early, in `config/application.rb`. Zeitwerk uses absolute paths internally, and applications running in `:zeitwerk` mode do not need `require_dependency`, so models, controllers, jobs, etc. do not need to be in `$LOAD_PATH`. Setting this to `false` saves Ruby from checking these directories when resolving `require` calls with relative paths, and saves Bootsnap work and RAM, since it does not need to build an index for them.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 7.1                   | `false`              |

The `lib` directory is not affected by this flag, it is added to `$LOAD_PATH` always.

#### `config.after_initialize`

Takes a block which will be run _after_ Rails has finished initializing the application. That includes the initialization of the framework itself, engines, and all the application's initializers in `config/initializers`. Note that this block _will_ be run for rake tasks. Useful for configuring values set up by other initializers:

```ruby
config.after_initialize do
  ActionView::Base.sanitized_allowed_tags.delete "div"
end
```

#### `config.after_routes_loaded`

Takes a block which will be run after Rails has finished loading the application routes. This block will also be run whenever routes are reloaded.

```ruby
config.after_routes_loaded do
  # Code that does something with Rails.application.routes
end
```

#### `config.allow_concurrency`

Controls whether requests should be handled concurrently. This should only
be set to `false` if application code is not thread safe. Defaults to `true`.

#### `config.asset_host`

Sets the host for the assets. Useful when CDNs are used for hosting assets, or when you want to work around the concurrency constraints built-in in browsers using different domain aliases. Shorter version of `config.action_controller.asset_host`.

#### `config.assume_ssl`

Makes application believe that all requests are arriving over SSL. This is useful when proxying through a load balancer that terminates SSL, the forwarded request will appear as though it's HTTP instead of HTTPS to the application. This makes redirects and cookie security target HTTP instead of HTTPS. This middleware makes the server assume that the proxy already terminated SSL, and that the request really is HTTPS.

#### `config.autoflush_log`

Enables writing log file output immediately instead of buffering. Defaults to
`true`.

#### `config.autoload_lib(ignore:)`

This method adds `lib` to `config.autoload_paths` and `config.eager_load_paths`.

Normally, the `lib` directory has subdirectories that should not be autoloaded or eager loaded. Please, pass their name relative to `lib` in the required `ignore` keyword argument. For example,

```ruby
config.autoload_lib(ignore: %w(assets tasks generators))
```

Please, see more details in the [autoloading guide](autoloading_and_reloading_constants.html).

#### `config.autoload_lib_once(ignore:)`

The method `config.autoload_lib_once` is similar to `config.autoload_lib`, except that it adds `lib` to `config.autoload_once_paths` instead.

By calling `config.autoload_lib_once`, classes and modules in `lib` can be autoloaded, even from application initializers, but won't be reloaded.

#### `config.autoload_once_paths`

Accepts an array of paths from which Rails will autoload constants that won't be wiped per request. Relevant if reloading is enabled, which it is by default in the `development` environment. Otherwise, all autoloading happens only once. All elements of this array must also be in `autoload_paths`. Default is an empty array.

#### `config.autoload_paths`

Accepts an array of paths from which Rails will autoload constants. Default is an empty array. Since [Rails 6](upgrading_ruby_on_rails.html#autoloading), it is not recommended to adjust this. See [Autoloading and Reloading Constants](autoloading_and_reloading_constants.html#config-autoload-paths).

#### `config.beginning_of_week`

Sets the default beginning of week for the
application. Accepts a valid day of week as a symbol (e.g. `:monday`).

#### `config.cache_classes`

Old setting equivalent to `!config.enable_reloading`. Supported for backwards compatibility.

#### `config.cache_store`

Configures which cache store to use for Rails caching. Options include one of the symbols `:memory_store`, `:file_store`, `:mem_cache_store`, `:null_store`, `:redis_cache_store`, or an object that implements the cache API. Defaults to `:file_store`. See [Cache Stores](caching_with_rails.html#other-cache-stores) for per-store configuration options.

#### `config.colorize_logging`

Specifies whether or not to use ANSI color codes when logging information. Defaults to `true`.

#### `config.consider_all_requests_local`

Is a flag. If `true` then any error will cause detailed debugging information to be dumped in the HTTP response, and the `Rails::Info` controller will show the application runtime context in `/rails/info/properties`. `true` by default in the development and test environments, and `false` in production. For finer-grained control, set this to `false` and implement `show_detailed_exceptions?` in controllers to specify which requests should provide debugging information on errors.

#### `config.console`

Allows you to set the class that will be used as console when you run `bin/rails console`. It's best to run it in the `console` block:

```ruby
console do
  # this block is called only when running console,
  # so we can safely require pry here
  require "pry"
  config.console = Pry
end
```

#### `config.content_security_policy_nonce_auto`

See [Adding a Nonce](security.html#adding-a-nonce) in the Security Guide

#### `config.content_security_policy_nonce_directives`

See [Adding a Nonce](security.html#adding-a-nonce) in the Security Guide

#### `config.content_security_policy_nonce_generator`

See [Adding a Nonce](security.html#adding-a-nonce) in the Security Guide

#### `config.content_security_policy_report_only`

See [Reporting Violations](security.html#reporting-violations) in the Security
Guide

#### `config.credentials.content_path`

The path of the encrypted credentials file.

Defaults to `config/credentials/#{Rails.env}.yml.enc` if it exists, or
`config/credentials.yml.enc` otherwise.

NOTE: In order for the `bin/rails credentials` commands to recognize this value,
it must be set in `config/application.rb` or `config/environments/#{Rails.env}.rb`.

#### `config.credentials.key_path`

The path of the encrypted credentials key file.

Defaults to `config/credentials/#{Rails.env}.key` if it exists, or
`config/master.key` otherwise.

NOTE: In order for the `bin/rails credentials` commands to recognize this value,
it must be set in `config/application.rb` or `config/environments/#{Rails.env}.rb`.

#### `config.debug_exception_response_format`

Sets the format used in responses when errors occur in the development environment. Defaults to `:api` for API only apps and `:default` for normal apps.

#### `config.disable_sandbox`

Controls whether or not someone can start a console in sandbox mode. This is helpful to avoid a long running session of sandbox console, that could lead a database server to run out of memory. Defaults to `false`.

#### `config.dom_testing_default_html_version`

Controls whether an HTML4 parser or an HTML5 parser is used by default by the test helpers in Action View, Action Dispatch, and `rails-dom-testing`.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:html4`             |
| 7.1                   | `:html5` (see NOTE)  |

NOTE: Nokogiri's HTML5 parser is not supported on JRuby, so on JRuby platforms Rails will fall back to `:html4`.

#### `config.eager_load`

When `true`, eager loads all registered `config.eager_load_namespaces`. This includes your application, engines, Rails frameworks, and any other registered namespace.

#### `config.eager_load_namespaces`

Registers namespaces that are eager loaded when `config.eager_load` is set to `true`. All namespaces in the list must respond to the `eager_load!` method.

#### `config.eager_load_paths`

Accepts an array of paths from which Rails will eager load on boot if `config.eager_load` is true. Defaults to every folder in the `app` directory of the application.

#### `config.enable_reloading`

If `config.enable_reloading` is true, application classes and modules are reloaded in between web requests if they change. Defaults to `true` in the `development` environment, and `false` in the `production` environment.

The predicate `config.reloading_enabled?` is also defined.

#### `config.encoding`

Sets up the application-wide encoding. Defaults to UTF-8.

#### `config.exceptions_app`

Sets the exceptions application invoked by the `ShowException` middleware when an exception happens.
Defaults to `ActionDispatch::PublicExceptions.new(Rails.public_path)`.

#### `config.file_watcher`

Is the class used to detect file updates in the file system when `config.reload_classes_only_on_change` is `true`. Rails ships with `ActiveSupport::FileUpdateChecker`, the default, and `ActiveSupport::EventedFileUpdateChecker`. Custom classes must conform to the `ActiveSupport::FileUpdateChecker` API.

Using `ActiveSupport::EventedFileUpdateChecker` depends on the [listen](https://github.com/guard/listen) gem:

```ruby
group :development do
  gem "listen", "~> 3.5"
end
```

On Linux and macOS no additional gems are needed, but some are required
[for *BSD](https://github.com/guard/listen#on-bsd) and
[for Windows](https://github.com/guard/listen#on-windows).

Note that [some setups are unsupported](https://github.com/guard/listen#issues--limitations).

#### `config.filter_parameters`

Used for filtering out the parameters that you don't want shown in the logs,
such as passwords or credit card numbers. It also filters out sensitive values
of database columns when calling `#inspect` on an Active Record object. By
default, Rails filters out passwords by adding the following filters in
`config/initializers/filter_parameter_logging.rb`.

```ruby
Rails.application.config.filter_parameters += [
  :passw, :email, :secret, :token, :_key, :crypt, :salt, :certificate, :otp, :ssn, :cvv, :cvc
]
```

Parameters filter works by partial matching regular expression.

#### `config.filter_redirect`

Used for filtering out redirect urls from application logs.

```ruby
Rails.application.config.filter_redirect += ["s3.amazonaws.com", /private-match/]
```

The redirect filter works by testing that urls include strings or match regular
expressions.

#### `config.force_ssl`

Forces all requests to be served over HTTPS, and sets "https://" as the default protocol when generating URLs. Enforcement of HTTPS is handled by the `ActionDispatch::SSL` middleware, which can be configured via `config.ssl_options`.

#### `config.helpers_paths`

Defines an array of additional paths to load view helpers.

#### `config.host_authorization`

Accepts a hash of options to configure the [HostAuthorization
middleware](#actiondispatch-hostauthorization)

#### `config.hosts`

An array of strings, regular expressions, or `IPAddr` used to validate the
`Host` header. Used by the [HostAuthorization
middleware](#actiondispatch-hostauthorization) to help prevent DNS rebinding
attacks.

#### `config.javascript_path`

Sets the path where your app's JavaScript lives relative to the `app` directory and the default value is `javascript`.
An app's configured `javascript_path` will be excluded from `autoload_paths`.

#### `config.log_file_size`

Defines the maximum size of the Rails log file in bytes. Defaults to `104_857_600` (100 MiB) in development and test, and unlimited in all other environments.

#### `config.log_formatter`

Defines the formatter of the Rails logger. This option defaults to an instance of `ActiveSupport::Logger::SimpleFormatter` for all environments. If you are setting a value for `config.logger` you must manually pass the value of your formatter to your logger before it is wrapped in an `ActiveSupport::TaggedLogging` instance, Rails will not do it for you.

#### `config.log_level`

Defines the verbosity of the Rails logger. This option defaults to `:debug` for all environments except production, where it defaults to `:info`. The available log levels are: `:debug`, `:info`, `:warn`, `:error`, `:fatal`, and `:unknown`.

#### `config.log_tags`

Accepts a list of methods that the `request` object responds to, a `Proc` that accepts the `request` object, or something that responds to `to_s`. This makes it easy to tag log lines with debug information like subdomain and request id - both very helpful in debugging multi-user production applications.

#### `config.logger`

Is the logger that will be used for `Rails.logger` and any related Rails logging such as `ActiveRecord::Base.logger`. It defaults to an instance of `ActiveSupport::TaggedLogging` that wraps an instance of `ActiveSupport::Logger` which outputs a log to the `log/` directory. You can supply a custom logger, to get full compatibility you must follow these guidelines:

* To support a formatter, you must manually assign a formatter from the `config.log_formatter` value to the logger.
* To support tagged logs, the log instance must be wrapped with `ActiveSupport::TaggedLogging`.
* To support silencing, the logger must include `ActiveSupport::LoggerSilence` module. The `ActiveSupport::Logger` class already includes these modules.

```ruby
class MyLogger < ::Logger
  include ActiveSupport::LoggerSilence
end

mylogger           = MyLogger.new(STDOUT)
mylogger.formatter = config.log_formatter
config.logger      = ActiveSupport::TaggedLogging.new(mylogger)
```

#### `config.middleware`

Allows you to configure the application's middleware. This is covered in depth in the [Configuring Middleware](#configuring-middleware) section below.

#### `config.precompile_filter_parameters`

When `true`, will precompile [`config.filter_parameters`](#config-filter-parameters)
using [`ActiveSupport::ParameterFilter.precompile_filters`][].

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.1                   | `true`               |

[`ActiveSupport::ParameterFilter.precompile_filters`]: https://api.rubyonrails.org/classes/ActiveSupport/ParameterFilter.html#method-c-precompile_filters

#### `config.public_file_server.enabled`

Configures whether Rails should serve static files from the public directory.
Defaults to `true`.

If the server software (e.g. NGINX or Apache) should serve static files instead,
set this value to `false`.

#### `config.railties_order`

Allows manually specifying the order that Railties/Engines are loaded. The
default value is `[:all]`.

```ruby
config.railties_order = [Blog::Engine, :main_app, :all]
```

#### `config.rake_eager_load`

When `true`, eager load the application when running Rake tasks. Defaults to `false`.

#### `config.relative_url_root`

Can be used to tell Rails that you are [deploying to a subdirectory](
configuring.html#deploy-to-a-subdirectory-relative-url-root). The default
is `ENV['RAILS_RELATIVE_URL_ROOT']`.

#### `config.reload_classes_only_on_change`

Enables or disables reloading of classes only when tracked files change. By default tracks everything on autoload paths and is set to `true`. If `config.enable_reloading` is `false`, this option is ignored.

#### `config.require_master_key`

Causes the app to not boot if a master key hasn't been made available through `ENV["RAILS_MASTER_KEY"]` or the `config/master.key` file.

#### `config.revision`

Sets the application revision for deployment tracking and error reporting. Must be a string.
When not set, Rails first checks `ENV["REVISION"]`, then tries reading from a `REVISION` file in the application root, and if both are absent
it attempts to get the current commit from the local git repository (default: `nil`).

```ruby
config.revision = ENV["GIT_SHA"]
```

Revision can be accessed via `Rails.app.revision`.

#### `config.sandbox_by_default`

When `true`, Rails console starts in sandbox mode by default, and `--no-sandbox` must be specified to start Rails console without sandbox mode. This helps prevent accidental writes to production databases. Defaults to `false`.

#### `config.secret_key_base`

The fallback for specifying the input secret for an application's key generator.
It is recommended to leave this unset, and instead to specify a `secret_key_base`
in `config/credentials.yml.enc`. See the [`secret_key_base` API documentation](
https://api.rubyonrails.org/classes/Rails/Application.html#method-i-secret_key_base)
for more information and alternative configuration methods.

#### `config.server_timing`

When `true`, adds the [`ServerTiming` middleware](#actiondispatch-servertiming)
to the middleware stack. Defaults to `false`, but is set to `true` in the
default generated `config/environments/development.rb` file.

#### `config.session_options`

Additional options passed to `config.session_store`. You should use
`config.session_store` to set this instead of modifying it yourself.

```ruby
config.session_store :cookie_store, key: "_your_app_session"
config.session_options # => {key: "_your_app_session"}
```

#### `config.session_store`

Specifies what class to use to store the session. Possible values are `:cache_store`, `:cookie_store`, `:mem_cache_store`, a custom store, or `:disabled`. `:disabled` tells Rails not to deal with sessions.

This setting is configured via a regular method call, rather than a setter. This allows additional options to be passed:

```ruby
config.session_store :cookie_store, key: "_your_app_session"
```

If a custom store is specified as a symbol, it will be resolved to the `ActionDispatch::Session` namespace:

```ruby
# use ActionDispatch::Session::MyCustomStore as the session store
config.session_store :my_custom_store
```

The default store is a cookie store with the application name as the session key.

#### `config.silence_healthcheck_path`

Specifies the path of the health check that should be silenced in the logs. Uses `Rails::Rack::SilenceRequest` to implement the silencing. All in service of keeping health checks from clogging the production logs, especially for early-stage applications.

```
config.silence_healthcheck_path = "/up"
```

#### `config.ssl_options`

Configuration options for the [`ActionDispatch::SSL`](https://api.rubyonrails.org/classes/ActionDispatch/SSL.html) middleware.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `{}`                 |
| 5.0                   | `{ hsts: { subdomains: true } }` |

#### `config.time_zone`

Sets the default time zone for the application and enables time zone awareness for Active Record.

#### `config.x`

Used to easily add nested custom configuration to the application config object

  ```ruby
  config.x.payment_processing.schedule = :daily
  Rails.configuration.x.payment_processing.schedule # => :daily
  ```

See [Custom Configuration](#custom-configuration)

#### `config.yjit`

Enables YJIT as of Ruby 3.3, to bring sizeable performance improvements. If you are
deploying to a memory constrained environment you may want to set this to `false`.
Additionally, you can pass a hash to configure YJIT options such as `{ stats: true }`.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.2                   | `true`               |
| 8.1                   | `!Rails.env.local?`  |

### Configuring Assets

#### `config.assets.css_compressor`

Defines the CSS compressor to use. It is set by default by `sass-rails`. The unique alternative value at the moment is `:yui`, which uses the `yui-compressor` gem.

#### `config.assets.js_compressor`

Defines the JavaScript compressor to use. Possible values are `:terser`, `:closure`, `:uglifier`, and `:yui`, which require the use of the `terser`, `closure-compiler`, `uglifier`, or `yui-compressor` gems respectively.

#### `config.assets.gzip`

A flag that enables the creation of gzipped version of compiled assets, along with non-gzipped assets. Set to `true` by default.

#### `config.assets.paths`

Contains the paths which are used to look for assets. Appending paths to this configuration option will cause those paths to be used in the search for assets.

#### `config.assets.precompile`

Allows you to specify additional assets (other than `application.css` and `application.js`) which are to be precompiled when `bin/rails assets:precompile` is run.

#### `config.assets.unknown_asset_fallback`

Allows you to modify the behavior of the asset pipeline when an asset is not in the pipeline, if you use sprockets-rails 3.2.0 or newer.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 5.1                   | `false`              |

#### `config.assets.prefix`

Defines the prefix where assets are served from. Defaults to `/assets`.

#### `config.assets.manifest`

Defines the full path to be used for the asset precompiler's manifest file. Defaults to a file named `manifest-<random>.json` in the `config.assets.prefix` directory within the public folder.

#### `config.assets.digest`

Enables the use of SHA256 fingerprints in asset names. Set to `true` by default.

#### `config.assets.debug`

Disables the concatenation and compression of assets.

#### `config.assets.version`

Is an option string that is used in SHA256 hash generation. This can be changed to force all files to be recompiled.

#### `config.assets.compile`

Is a boolean that can be used to turn on live Sprockets compilation in production.

#### `config.assets.logger`

Accepts a logger conforming to the interface of Log4r or the default Ruby `Logger` class. Defaults to the same configured at `config.logger`. Setting `config.assets.logger` to `false` will turn off served assets logging.

#### `config.assets.quiet`

Disables logging of assets requests. Set to `true` by default in `config/environments/development.rb`.

### Configuring Generators

Rails allows you to alter what generators are used with the `config.generators` method. This method takes a block:

```ruby
config.generators do |g|
  g.orm :active_record
  g.test_framework :test_unit
end
```

The full set of methods that can be used in this block are as follows:

* `force_plural` allows pluralized model names. Defaults to `false`.
* `helper` defines whether or not to generate helpers. Defaults to `true`.
* `integration_tool` defines which integration tool to use to generate integration tests. Defaults to `:test_unit`.
* `system_tests` defines which integration tool to use to generate system tests. Defaults to `:test_unit`.
* `orm` defines which orm to use. Defaults to `false` and will use Active Record by default.
* `resource_controller` defines which generator to use for generating a controller when using `bin/rails generate resource`. Defaults to `:controller`.
* `resource_route` defines whether a resource route definition should be generated
  or not. Defaults to `true`.
* `scaffold_controller` different from `resource_controller`, defines which generator to use for generating a _scaffolded_ controller when using `bin/rails generate scaffold`. Defaults to `:scaffold_controller`.
* `test_framework` defines which test framework to use. Defaults to `false` and will use minitest by default.
* `template_engine` defines which template engine to use, such as ERB or Haml. Defaults to `:erb`.
* `apply_rubocop_autocorrect_after_generate!` applies RuboCop's autocorrect feature after Rails generators are run.

### Configuring Middleware

Every Rails application comes with a standard set of middleware which it uses in this order in the development environment:

#### `ActionDispatch::HostAuthorization`

Prevents against DNS rebinding and other `Host` header attacks.
It is included in the development environment by default with the following configuration:

```ruby
Rails.application.config.hosts = [
  IPAddr.new("0.0.0.0/0"),        # All IPv4 addresses.
  IPAddr.new("::/0"),             # All IPv6 addresses.
  "localhost",                    # The localhost reserved domain.
  ENV["RAILS_DEVELOPMENT_HOSTS"]  # Additional comma-separated hosts for development.
]
```

In other environments `Rails.application.config.hosts` is empty and no
`Host` header checks will be done. If you want to guard against header
attacks on production, you have to manually permit the allowed hosts
with:

```ruby
Rails.application.config.hosts << "product.com"
```

Adding a specific port will make sure only that port is authorized:

```ruby
Rails.application.config.hosts << "product.com:3000"
```

The host of a request is checked against the `hosts` entries with the case
operator (`#===`), which lets `hosts` support entries of type `Regexp`,
`Proc` and `IPAddr` to name a few. Here is an example with a regexp.

```ruby
# Allow requests from subdomains like `www.product.com` and
# `beta1.product.com`.
Rails.application.config.hosts << /.*\.product\.com/
```

The provided regexp will be wrapped with both anchors (`\A` and `\z`) so it
must match the entire hostname. `/product.com/`, for example, once anchored,
would fail to match `www.product.com`.

A special case is supported that allows you to permit the domain and all sub-domains:

```ruby
# Allow requests from the domain itself `product.com` and subdomains like `www.product.com` and `beta1.product.com`.
Rails.application.config.hosts << ".product.com"
```

You can exclude certain requests from Host Authorization checks by setting
`config.host_authorization.exclude`:

```ruby
# Exclude requests for the /healthcheck/ path from host checking
Rails.application.config.host_authorization = {
  exclude: ->(request) { request.path.include?("healthcheck") }
}
```

When a request comes to an unauthorized host, a default Rack application
will run and respond with `403 Forbidden`. This can be customized by setting
`config.host_authorization.response_app`. For example:

```ruby
Rails.application.config.host_authorization = {
  response_app: -> env do
    [400, { "Content-Type" => "text/plain" }, ["Bad Request"]]
  end
}
```

#### `ActionDispatch::ServerTiming`

Adds the [`Server-Timing`][] header to the response, which includes performance
metrics from the server. This data can be viewed by inspecting the response in
the Network panel of the browser's Developer Tools. Most browsers provide a
Timing tab that visualizes the data.

[`Server-Timing`]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Server-Timing

#### `ActionDispatch::SSL`

Forces every request to be served using HTTPS. Enabled if `config.force_ssl` is set to `true`. Options passed to this can be configured by setting `config.ssl_options`.

#### `ActionDispatch::Static`

Is used to serve static assets. Disabled if `config.public_file_server.enabled` is `false`. Set `config.public_file_server.index_name` if you need to serve a static directory index file that is not named `index`. For example, to serve `main.html` instead of `index.html` for directory requests, set `config.public_file_server.index_name` to `"main"`.

#### `ActionDispatch::Executor`

Allows thread safe code reloading. Disabled if `config.allow_concurrency` is `false`, which causes `Rack::Lock` to be loaded. `Rack::Lock` wraps the app in mutex so it can only be called by a single thread at a time.

#### `ActiveSupport::Cache::Strategy::LocalCache`

Serves as a basic memory backed cache. This cache is not thread safe and is intended only for serving as a temporary memory cache for a single thread.

#### `Rack::Runtime`

Sets an `X-Runtime` header, containing the time (in seconds) taken to execute the request.

#### `Rails::Rack::Logger`

Notifies the logs that the request has begun. After request is complete, flushes all the logs.

#### `ActionDispatch::ShowExceptions`

Rescues any exception returned by the application and renders nice exception pages if the request is local or if `config.consider_all_requests_local` is set to `true`. If `config.action_dispatch.show_exceptions` is set to `:none`, exceptions will be raised regardless.

#### `ActionDispatch::RequestId`

Makes a unique X-Request-Id header available to the response and enables the `ActionDispatch::Request#uuid` method. Configurable with `config.action_dispatch.request_id_header`.

#### `ActionDispatch::RemoteIp`

Checks for IP spoofing attacks and gets valid `client_ip` from request headers. Configurable with the `config.action_dispatch.ip_spoofing_check`, and `config.action_dispatch.trusted_proxies` options.

#### `Rack::Sendfile`

Intercepts responses whose body is being served from a file and replaces it with a server specific X-Sendfile header. Configurable with `config.action_dispatch.x_sendfile_header`.

#### `ActionDispatch::Callbacks`

Runs the prepare callbacks before serving the request.

#### `ActionDispatch::Cookies`

Sets cookies for the request.

#### `ActionDispatch::Session::CookieStore`

Is responsible for storing the session in cookies. An alternate middleware can be used for this by changing [`config.session_store`](#config-session-store).

#### `ActionDispatch::Flash`

Sets up the `flash` keys. Only available if [`config.session_store`](#config-session-store) is set to a value.

#### `Rack::MethodOverride`

Allows the method to be overridden if `params[:_method]` is set. This is the middleware which supports the PATCH, PUT, and DELETE HTTP method types.

#### `Rack::Head`

Returns an empty body for all HEAD requests. It leaves all other requests unchanged.

#### Adding Custom Middleware

Besides these usual middleware, you can add your own by using the `config.middleware.use` method:

```ruby
config.middleware.use Magical::Unicorns
```

This will put the `Magical::Unicorns` middleware on the end of the stack. You can use `insert_before` if you wish to add a middleware before another.

```ruby
config.middleware.insert_before Rack::Head, Magical::Unicorns
```

Or you can insert a middleware to exact position by using indexes. For example, if you want to insert `Magical::Unicorns` middleware on top of the stack, you can do it, like so:

```ruby
config.middleware.insert_before 0, Magical::Unicorns
```

There's also `insert_after` which will insert a middleware after another:

```ruby
config.middleware.insert_after Rack::Head, Magical::Unicorns
```

Middlewares can also be completely swapped out and replaced with others:

```ruby
config.middleware.swap ActionController::Failsafe, Lifo::Failsafe
```

Middlewares can be moved from one place to another:

```ruby
config.middleware.move_before ActionDispatch::Flash, Magical::Unicorns
```

This will move the `Magical::Unicorns` middleware before
`ActionDispatch::Flash`. You can also move it after:

```ruby
config.middleware.move_after ActionDispatch::Flash, Magical::Unicorns
```

They can also be removed from the stack completely:

```ruby
config.middleware.delete Rack::MethodOverride
```

### Configuring i18n

All these configuration options are delegated to the `I18n` library.

#### `config.i18n.available_locales`

Defines the permitted available locales for the app. Defaults to all locale keys found in locale files, usually only `:en` on a new application.

#### `config.i18n.default_locale`

Sets the default locale of an application used for i18n. Defaults to `:en`.

#### `config.i18n.enforce_available_locales`

Ensures that all locales passed through i18n must be declared in the `available_locales` list, raising an `I18n::InvalidLocale` exception when setting an unavailable locale. Defaults to `true`. It is recommended not to disable this option unless strongly required, since this works as a security measure against setting any invalid locale from user input.

#### `config.i18n.load_path`

Sets the path Rails uses to look for locale files. Defaults to `config/locales/**/*.{yml,rb}`.

#### `config.i18n.raise_on_missing_translations`

Determines whether an error should be raised for missing translations. If `true`, views and controllers raise `I18n::MissingTranslationData`. If `:strict`, models also raise the error. This defaults to `false`.

#### `config.i18n.fallbacks`

Sets fallback behavior for missing translations. Here are 3 usage examples for this option:

  * You can set the option to `true` for using default locale as fallback, like so:

    ```ruby
    config.i18n.fallbacks = true
    ```

  * Or you can set an array of locales as fallback, like so:

    ```ruby
    config.i18n.fallbacks = [:tr, :en]
    ```

  * Or you can set different fallbacks for locales individually. For example, if you want to use `:tr` for `:az` and `:de`, `:en` for `:da` as fallbacks, you can do it, like so:

    ```ruby
    config.i18n.fallbacks = { az: :tr, da: [:de, :en] }
    #or
    config.i18n.fallbacks.map = { az: :tr, da: [:de, :en] }
    ```

### Configuring Active Model

#### `config.active_model.i18n_customize_full_message`

Controls whether the [`Error#full_message`][ActiveModel::Error#full_message] format can be overridden in an i18n locale file. Defaults to `false`.

When set to `true`, `full_message` will look for a format at the attribute and model level of the locale files. The default format is `"%{attribute} %{message}"`, where `attribute` is the name of the attribute, and `message` is the validation-specific message. The following example overrides the format for all `Person` attributes, as well as the format for a specific `Person` attribute (`age`).

```ruby
class Person
  include ActiveModel::Validations

  attr_accessor :name, :age

  validates :name, :age, presence: true
end
```

```yml
en:
  activemodel: # or activerecord:
    errors:
      models:
        person:
          # Override the format for all Person attributes:
          format: "Invalid %{attribute} (%{message})"
          attributes:
            age:
              # Override the format for the age attribute:
              format: "%{message}"
              blank: "Please fill in your %{attribute}"
```

```irb
irb> person = Person.new.tap(&:valid?)

irb> person.errors.full_messages
=> [
  "Invalid Name (can't be blank)",
  "Please fill in your Age"
]

irb> person.errors.messages
=> {
  :name => ["can't be blank"],
  :age  => ["Please fill in your Age"]
}
```

[ActiveModel::Error#full_message]: https://api.rubyonrails.org/classes/ActiveModel/Error.html#method-i-full_message

### Configuring Active Record

`config.active_record` includes a variety of configuration options:

#### `config.active_record.logger`

Accepts a logger conforming to the interface of Log4r or the default Ruby Logger class, which is then passed on to any new database connections made. You can retrieve this logger by calling `logger` on either an Active Record model class or an Active Record model instance. Set to `nil` to disable logging.

#### `config.active_record.primary_key_prefix_type`

Lets you adjust the naming for primary key columns. By default, Rails assumes that primary key columns are named `id` (and this configuration option doesn't need to be set). There are two other choices:

* `:table_name` would make the primary key for the Customer class `customerid`.
* `:table_name_with_underscore` would make the primary key for the Customer class `customer_id`.

#### `config.active_record.table_name_prefix`

Lets you set a global string to be prepended to table names. If you set this to `northwest_`, then the Customer class will look for `northwest_customers` as its table. The default is an empty string.

#### `config.active_record.table_name_suffix`

Lets you set a global string to be appended to table names. If you set this to `_northwest`, then the Customer class will look for `customers_northwest` as its table. The default is an empty string.

#### `config.active_record.schema_migrations_table_name`

Lets you set a string to be used as the name of the schema migrations table.

#### `config.active_record.internal_metadata_table_name`

Lets you set a string to be used as the name of the internal metadata table.

#### `config.active_record.protected_environments`

Lets you set an array of names of environments where destructive actions should be prohibited.

#### `config.active_record.pluralize_table_names`

Specifies whether Rails will look for singular or plural table names in the database. If set to `true` (the default), then the Customer class will use the `customers` table. If set to `false`, then the Customer class will use the `customer` table.

WARNING: Some Rails generators and installers (notably `active_storage:install`
and `action_text:install`) create tables with plural names regardless of this
setting. If you set `pluralize_table_names` to `false`, you will need to
manually rename those tables after installation to maintain consistency.
This limitation exists because these installers use fixed table names
in their migrations for compatibility reasons.

#### `config.active_record.default_timezone`

Determines whether to use `Time.local` (if set to `:local`) or `Time.utc` (if set to `:utc`) when pulling dates and times from the database. The default is `:utc`.

#### `config.active_record.schema_format`

Controls the format for dumping the database schema to a file. The options are `:ruby` (the default) for a database-independent version that depends on migrations, or `:sql` for a set of (potentially database-dependent) SQL statements. This can be overridden per-database by setting `schema_format` in your database configuration.

#### `config.active_record.error_on_ignored_order`

Specifies if an error should be raised if the order of a query is ignored during a batch query. The options are `true` (raise error) or `false` (warn). Default is `false`.

#### `config.active_record.timestamped_migrations`

Controls whether migrations are numbered with serial integers or with timestamps. The default is `true`, to use timestamps, which are preferred if there are multiple developers working on the same application.

#### `config.active_record.automatically_invert_plural_associations`

Controls whether Active Record will automatically look for inverse relations with a pluralized name.

Example:

```ruby
class Post < ApplicationRecord
  has_many :comments
end

class Comment < ApplicationRecord
  belongs_to :post
end
```

In the above case Active Record used to only look for a `:comment` (singular) association in `Post`, and won't find it.

With this option enabled, it will also look for a `:comments` association. In the vast majority of cases
having the inverse association discovered is beneficial as it can prevent some useless queries, but
it may cause backward compatibility issues with legacy code that doesn't expect it.

This behavior can be disabled on a per-model basis:

```ruby
class Comment < ApplicationRecord
  self.automatically_invert_plural_associations = false

  belongs_to :post
end
```

And on a per-association basis:

```ruby
class Comment < ApplicationRecord
  self.automatically_invert_plural_associations = true

  belongs_to :post, inverse_of: nil
end
```

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |

#### `config.active_record.validate_migration_timestamps`

Controls whether to validate migration timestamps. When set, an error will be raised if the
timestamp prefix for a migration is more than a day ahead of the timestamp associated with the
current time. This is done to prevent forward-dating of migration files, which can impact migration
generation and other migration commands. `config.active_record.timestamped_migrations` must be set to `true`.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.2                   | `true`               |

#### `config.active_record.db_warnings_action`

Controls the action to be taken when an SQL query produces a warning. The following options are available:

  * `:ignore` - Database warnings will be ignored. This is the default.

  * `:log` - Database warnings will be logged via `ActiveRecord.logger` at the `:warn` level.

  * `:raise` - Database warnings will be raised as `ActiveRecord::SQLWarning`.

  * `:report` - Database warnings will be reported to subscribers of Rails' error reporter.

  * Custom proc - A custom proc can be provided. It should accept a `SQLWarning` error object.

    For example:

    ```ruby
    config.active_record.db_warnings_action = ->(warning) do
      # Report to custom exception reporting service
      Bugsnag.notify(warning.message) do |notification|
        notification.add_metadata(:warning_code, warning.code)
        notification.add_metadata(:warning_level, warning.level)
      end
    end
    ```

#### `config.active_record.db_warnings_ignore`

Specifies an allowlist of warning codes and messages that will be ignored, regardless of the configured `db_warnings_action`.
The default behavior is to report all warnings. Warnings to ignore can be specified as Strings or Regexps. For example:

  ```ruby
  config.active_record.db_warnings_action = :raise
  # The following warnings will not be raised
  config.active_record.db_warnings_ignore = [
    /Invalid utf8mb4 character string/,
    "An exact warning message",
    "1062", # MySQL Error 1062: Duplicate entry
  ]
  ```

#### `config.active_record.migration_strategy`

Controls the strategy class used to perform schema statement methods in a migration. The default class
delegates to the connection adapter. Custom strategies should inherit from `ActiveRecord::Migration::ExecutionStrategy`,
or may inherit from `DefaultStrategy`, which will preserve the default behavior for methods that aren't implemented:

```ruby
class CustomMigrationStrategy < ActiveRecord::Migration::DefaultStrategy
  def drop_table(*)
    raise "Dropping tables is not supported!"
  end
end

config.active_record.migration_strategy = CustomMigrationStrategy
```

You can also configure migration strategies on a per-adapter basis by setting the `migration_strategy` class on the adapter itself.
This is useful when you want to customize migration behavior for a specific database type.

For example, to use a custom migration strategy for PostgreSQL:

```ruby
class CustomPostgresStrategy < ActiveRecord::Migration::DefaultStrategy
  def drop_table(*)
    # Custom logic specific to PostgreSQL
  end
end

ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.migration_strategy = CustomPostgresStrategy
```

By assigning to the adapter class, all migrations run through connections using that adapter will use the specified strategy.

#### `config.active_record.schema_versions_formatter`

Controls the formatter class used by schema dumper to format versions information. Custom class can be provided
to change the default behavior:

```ruby
class CustomSchemaVersionsFormatter
  def initialize(connection)
    @connection = connection
  end

  def format(versions)
    # Special sorting of versions to reduce the likelihood of conflicts.
    sorted_versions = versions.sort { |a, b| b.to_s.reverse <=> a.to_s.reverse }

    sql = +"INSERT INTO schema_migrations (version) VALUES\n"
    sql << sorted_versions.map { |v| "(#{@connection.quote(v)})" }.join(",\n")
    sql << ";"
    sql
  end
end

config.active_record.schema_versions_formatter = CustomSchemaVersionsFormatter
```

#### `config.active_record.lock_optimistically`

Controls whether Active Record will use optimistic locking and is `true` by default.

#### `config.active_record.cache_timestamp_format`

Controls the format of the timestamp value in the cache key. Default is `:usec`.

#### `config.active_record.record_timestamps`

Is a boolean value which controls whether or not timestamping of `create` and `update` operations on a model occur. The default value is `true`.

#### `config.active_record.partial_inserts`

Is a boolean value and controls whether or not partial writes are used when creating new records (i.e. whether inserts only set attributes that are different from the default).

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 7.0                   | `false`              |

#### `config.active_record.partial_updates`

Is a boolean value and controls whether or not partial writes are used when updating existing records (i.e. whether updates only set attributes that are dirty). Note that when using partial updates, you should also use optimistic locking `config.active_record.lock_optimistically` since concurrent updates may write attributes based on a possibly stale read state. The default value is `true`.

#### `config.active_record.maintain_test_schema`

Is a boolean value which controls whether Active Record should try to keep your
test database schema up-to-date with `db/schema.rb` (or `db/structure.sql`) when
you run your tests. The default is `true`.

#### `config.active_record.dump_schema_after_migration`

Is a flag which controls whether or not schema dump should happen
(`db/schema.rb` or `db/structure.sql`) when you run migrations. This is set to
`false` in `config/environments/production.rb` which is generated by Rails. The
default value is `true` if this configuration is not set.

#### `config.active_record.dump_schema_migrations`

Controls whether Ruby schema dumps include the migration versions recorded in
the `schema_migrations` table. This can be overridden per database by setting
`:dump_schema_migrations` in the database configuration.

#### `config.active_record.dump_schema_migrations_sort_by`

When `config.active_record.dump_schema_migrations` is enabled, migration
versions are ordered by their reversed strings by default, to help avoid merge
conflicts. This option provides a way to configure that:

```ruby
# Linear order.
config.active_record.dump_schema_migrations_sort_by = :itself

# Hash-based order.
require "digest/md5"

config.active_record.dump_schema_migrations_sort_by = ->(version) {
  Digest::MD5.hexdigest(version)
}
```

The value has to be a proc (or respond to `to_proc`) which is called with a
string as argument.

Please note that the order of the versions does not matter for Active Record,
the `schema_migrations` table acts as a set.

#### `config.active_record.dump_schemas`

Controls which database schemas will be dumped when calling `db:schema:dump`.
The options are `:schema_search_path` (the default) which dumps any schemas listed in `schema_search_path`,
`:all` which always dumps all schemas regardless of the `schema_search_path`,
or a string of comma separated schemas.

#### `config.active_record.before_committed_on_all_records`

Enable before_committed! callbacks on all enrolled records in a transaction.
The previous behavior was to only run the callbacks on the first copy of a record
if there were multiple copies of the same record enrolled in the transaction.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.1                   | `true`               |

#### `config.active_record.belongs_to_required_by_default`

Is a boolean value and controls whether a record fails validation if
`belongs_to` association is not present.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `nil`                |
| 5.0                   | `true`               |

#### `config.active_record.belongs_to_required_validates_foreign_key`

Enable validating only parent-related columns for presence when the parent is mandatory.
The previous behavior was to validate the presence of the parent record, which performed an extra query
to get the parent every time the child record was updated, even when parent has not changed.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 7.1                   | `false`              |

#### `config.active_record.marshalling_format_version`

Define which format to use when an Active Record object is serialized with Marshal.

As of Rails 8.0, only the `7.1` is supported.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| 7.1                   | `7.1`                |

#### `config.active_record.action_on_strict_loading_violation`

Enables raising or logging an exception if strict_loading is set on an
association. The default value is `:raise` in all environments. It can be
changed to `:log` to send violations to the logger instead of raising.

#### `config.active_record.strict_loading_by_default`

Is a boolean value that either enables or disables strict_loading mode by
default. Defaults to `false`.

#### `config.active_record.strict_loading_mode`

Sets the mode in which strict loading is reported. Defaults to `:all`. It can be
changed to `:n_plus_one_only` to only report when loading associations that will
lead to an N + 1 query.

#### `config.active_record.index_nested_attribute_errors`

Allows errors for nested `has_many` relationships to be displayed with an index
as well as the error. Defaults to `false`.

#### `config.active_record.use_schema_cache_dump`

Enables users to get schema cache information from `db/schema_cache.yml`
(generated by `bin/rails db:schema:cache:dump`), instead of having to send a
query to the database to get this information. Defaults to `true`.

#### `config.active_record.cache_versioning`

Indicates whether to use a stable `#cache_key` method that is accompanied by a
changing version in the `#cache_version` method.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 5.2                   | `true`               |

#### `config.active_record.collection_cache_versioning`

Enables the same cache key to be reused when the object being cached of type
`ActiveRecord::Relation` changes by moving the volatile information (max
updated at and count) of the relation's cache key into the cache version to
support recycling cache key.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 6.0                   | `true`               |

#### `config.active_record.has_many_inversing`

Enables setting the inverse record when traversing `belongs_to` to `has_many`
associations.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 6.1                   | `true`               |

#### `config.active_record.automatic_scope_inversing`

Enables automatically inferring the `inverse_of` for associations with a scope.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.0                   | `true`               |

#### `config.active_record.destroy_association_async_job`

Allows specifying the job that will be used to destroy the associated records in background. It defaults to `ActiveRecord::DestroyAssociationAsyncJob`.

#### `config.active_record.destroy_association_async_batch_size`

Allows specifying the maximum number of records that will be destroyed in a background job by the `dependent: :destroy_async` association option. All else equal, a lower batch size will enqueue more, shorter-running background jobs, while a higher batch size will enqueue fewer, longer-running background jobs. This option defaults to `nil`, which will cause all dependent records for a given association to be destroyed in the same background job.

#### `config.active_record.queues.destroy`

Allows specifying the Active Job queue to use for destroy jobs. When this option
is `nil`, purge jobs are sent to the default Active Job queue (see
[`config.active_job.default_queue_name`][]). It defaults to `nil`.

#### `config.active_record.enumerate_columns_in_select_statements`

When `true`, will always include column names in `SELECT` statements, and avoid wildcard `SELECT * FROM ...` queries. This avoids prepared statement cache errors when adding columns to a PostgreSQL database for example. Defaults to `false`.

#### `config.active_record.verify_foreign_keys_for_fixtures`

Ensures all foreign key constraints are valid after fixtures are loaded in tests. Supported by PostgreSQL and SQLite only.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.0                   | `true`               |

#### `config.active_record.raise_on_assign_to_attr_readonly`

Enable raising on assignment to attr_readonly attributes. The previous
behavior would allow assignment but silently not persist changes to the
database.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.1                   | `true`               |

#### `config.active_record.run_commit_callbacks_on_first_saved_instances_in_transaction`

When multiple Active Record instances change the same record within a transaction, Rails runs `after_commit` or `after_rollback` callbacks for only one of them. This option specifies how Rails chooses which instance receives the callbacks.

When `true`, transactional callbacks are run on the first instance to save, even though its instance state may be stale.

When `false`, transactional callbacks are run on the instances with the freshest instance state. Those instances are chosen as follows:

- In general, run transactional callbacks on the last instance to save a given record within the transaction.
- There are two exceptions:
    - If the record is created within the transaction, then updated by another instance, `after_create_commit` callbacks will be run on the second instance. This is instead of the `after_update_commit` callbacks that would naively be run based on that instance’s state.
    - If the record is destroyed within the transaction, then `after_destroy_commit` callbacks will be fired on the last destroyed instance, even if a stale instance subsequently performed an update (which will have affected 0 rows).

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 7.1                   | `false`              |

#### `config.active_record.default_column_serializer`

The serializer implementation to use if none is explicitly specified for a given
column.

Historically `serialize` and `store` while allowing to use alternative serializer
implementations, would use `YAML` by default, but it's not a very efficient format
and can be the source of security vulnerabilities if not carefully employed.

As such it is recommended to prefer stricter, more limited formats for database
serialization.

Unfortunately there isn't really any suitable defaults available in Ruby's standard
library. `JSON` could work as a format, but the `json` gems will cast unsupported
types to strings which may lead to bugs.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `YAML`               |
| 7.1                   | `nil`                |

#### `config.active_record.run_after_transaction_callbacks_in_order_defined`

When `true`, `after_commit` callbacks are executed in the order they are defined in a model. When `false`, they are executed in reverse order.

All other callbacks are always executed in the order they are defined in a model (unless you use `prepend: true`).

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.1                   | `true`               |

#### `config.active_record.query_log_tags_enabled`

Specifies whether or not to enable adapter-level query comments. Defaults to `false`, but is set to `true` in the default generated `config/environments/development.rb` file.When this is set to `true` database prepared statements will be automatically disabled. If prepared statements are desired in conjunction with `query_log_tags` you must explicitly opt-out of ActiveRecords disabling mechanism: `config.active_record.disable_preprared_statments = false`.

Note: High cardinality comments can cause degraded db performance as the database may not be able to rely on a query plan cache. If forcing prepared statements with query log tags high cardinality values should be avoided. For example, `:request_id` or `admin_id`. Even basic `controller#action` tags can cause high cardinality on basic queries such as a current_user lookup since it will happen across many endpoints.

#### `config.active_record.query_log_tags`

Define an `Array` specifying the key/value tags to be inserted in an SQL comment. Defaults to
`[ :application, :controller, :action, :job ]`. The available tags are: `:application`, `:controller`,
`:namespaced_controller`, `:action`, `:job`, and `:source_location`.

WARNING: Calculating the `:source_location` of a query can be slow, so you should consider its impact if using it in a production environment.

#### `config.active_record.query_log_tags_format`

A `Symbol` specifying the formatter to use for tags. Valid values are `:sqlcommenter` and `:legacy`.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:legacy`            |
| 7.1                   | `:sqlcommenter`      |

#### `config.active_record.cache_query_log_tags`

Specifies whether or not to enable caching of query log tags. For applications
that have a large number of queries, caching query log tags can provide a
performance benefit when the context does not change during the lifetime of the
request or job execution. Defaults to `false`.

#### `config.active_record.query_log_tags_prepend_comment`

Specifies whether or not to prepend query log tags comment to the query.

By default comments are appended at the end of the query. Certain databases, such as MySQL will
truncate the query text. This is the case for slow query logs and the results of querying
some InnoDB internal tables where the length of the query is more than 1024 bytes.
In order to not lose the log tags comments from the queries, you can prepend the comments using this option.

Defaults to `false`.

#### `config.active_record.schema_cache_ignored_tables`

**Note:** This configuration is deprecated in favor of
[`config.active_record.schema_ignored_tables`](#config-active-record-schema-ignored-tables),
and will be removed in a future Rails version. It is now an alias for that
option, so setting it also excludes the tables from the schema file.

#### `config.active_record.schema_ignored_tables`

Define the list of tables that should be ignored when generating the schema
cache and the schema file. It accepts an `Array` of strings, representing the
table names, or regular expressions.

**Note:** This configuration replaces the deprecated
[`config.active_record.schema_cache_ignored_tables`](#config-active-record-schema-cache-ignored-tables)
and [`ActiveRecord::SchemaDumper.ignore_tables`](#activerecord-schemadumper-ignore-tables)
options.

#### `config.active_record.verbose_query_logs`

Specifies if source locations of methods that call database queries should be logged below relevant queries. By default, the flag is `true` in development and `false` in all other environments.

#### `config.active_record.sqlite3_adapter_strict_strings_by_default`

Specifies whether the SQLite3Adapter should be used in a strict strings mode.
The use of a strict strings mode disables double-quoted string literals.

SQLite has some quirks around double-quoted string literals.
It first tries to consider double-quoted strings as identifier names, but if they don't exist
it then considers them as string literals. Because of this, typos can silently go unnoticed.
For example, it is possible to create an index for a non existing column.
See [SQLite documentation](https://www.sqlite.org/quirks.html#double_quoted_string_literals_are_accepted) for more details.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.1                   | `true`               |

#### `config.active_record.postgresql_adapter_decode_bytea`

Specifies whether the PostgresqlAdapter should decode bytea columns.

```ruby
ActiveRecord::Base.connection
     .select_value("select '\\x48656c6c6f'::bytea").encoding #=> Encoding::BINARY
```


The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 8.2                   | `true`               |

#### `config.active_record.postgresql_adapter_decode_dates`

Specifies whether the PostgresqlAdapter should decode date columns.

```ruby
ActiveRecord::Base.connection
     .select_value("select '2024-01-01'::date").class #=> Date
```


The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.2                   | `true`               |

#### `config.active_record.postgresql_adapter_decode_money`

Specifies whether the PostgresqlAdapter should decode money columns.

```ruby
ActiveRecord::Base.connection
     .select_value("select '12.34'::money").class #=> BigDecimal
```


The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 8.2                   | `true`               |


#### `config.active_record.async_query_executor`

Specifies how asynchronous queries are pooled.

It defaults to `nil`, which means `load_async` is disabled and instead directly executes queries in the foreground.
For queries to actually be performed asynchronously, it must be set to either `:global_thread_pool` or `:multi_thread_pool`.

`:global_thread_pool` will use a single pool for all databases the application connects to. This is the preferred configuration
for applications with only a single database, or applications which only ever query one database shard at a time.

`:multi_thread_pool` will use one pool per database, and each pool size can be configured individually in `database.yml` through the
`max_threads` and `min_threads` properties. This can be useful to applications regularly querying multiple databases at a time, and that need to more precisely define the max concurrency.

#### `config.active_record.global_executor_concurrency`

Used in conjunction with `config.active_record.async_query_executor = :global_thread_pool`, defines how many asynchronous
queries can be executed concurrently.

Defaults to `4`.

This number must be considered in accordance with the database connection pool size configured in `database.yml`. The connection pool
should be large enough to accommodate both the foreground threads (ie. web server or job worker threads) and background threads.

For each process, Rails will create one global query executor that uses this many threads to process async queries. Thus, the pool size
should be at least `thread_count + global_executor_concurrency + 1`. For example, if your web server has a maximum of 3 threads,
and `global_executor_concurrency` is set to 4, then your pool size should be at least 8.

#### `config.active_record.yaml_column_permitted_classes`

Defaults to `[Symbol]`. Allows applications to include additional permitted classes to `safe_load()` on the `ActiveRecord::Coders::YAMLColumn`.

#### `config.active_record.use_yaml_unsafe_load`

Defaults to `false`. Allows applications to opt into using `unsafe_load` on the `ActiveRecord::Coders::YAMLColumn`.

#### `config.active_record.raise_int_wider_than_64bit`

Defaults to `true`. Determines whether to raise an exception or not when
the PostgreSQL adapter is provided an integer that is wider than signed
64bit representation.

#### `config.active_record.generate_secure_token_on`

Controls when to generate a value for `has_secure_token` declarations. By
default, generate the value when the model is initialized:

```ruby
class User < ApplicationRecord
  has_secure_token
end

record = User.new
record.token # => "fwZcXX6SkJBJRogzMdciS7wf"
```

With `config.active_record.generate_secure_token_on = :create`, generate the
value when the model is created:

```ruby
# config/application.rb

config.active_record.generate_secure_token_on = :create

# app/models/user.rb
class User < ApplicationRecord
  has_secure_token on: :create
end

record = User.new
record.token # => nil
record.save!
record.token # => "fwZcXX6SkJBJRogzMdciS7wf"
```

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:create`            |
| 7.1                   | `:initialize`        |


#### `config.active_record.permanent_connection_checkout`

Controls whether `ActiveRecord::Base.connection` raises an error, emits a deprecation warning, or neither.

`ActiveRecord::Base.connection` checkouts a database connection from the pool and keeps it leased until the end of
the request or job. This behavior can be undesirable in environments that use many more threads or fibers than there
is available connections.

This configuration can be used to track down and eliminate code that calls `ActiveRecord::Base.connection` and
migrate it to use `ActiveRecord::Base.with_connection` instead.

The value can be set to `:disallowed`, `:deprecated`, or `true` to respectively raise an error, emit a deprecation
warning, or neither.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |

#### `config.active_record.database_cli`

Controls which CLI tool will be used for accessing the database when running `bin/rails dbconsole`. By default
the standard tool for the database will be used (e.g. `psql` for PostgreSQL and `mysql` for MySQL). The option
takes a hash which specifies the tool per-database system, and an array can be used where fallback options are
required:

```ruby
# config/application.rb

config.active_record.database_cli = { postgresql: "pgcli", mysql: %w[ mycli mysql ] }
```

#### `config.active_record.use_legacy_signed_id_verifier`

Controls whether signed IDs are generated and verified using legacy options. Can be set to:

* `:generate_and_verify` (default) - Generate and verify signed IDs using the following legacy options:

    ```ruby
    { digest: "SHA256", serializer: JSON, url_safe: true }
    ```

* `:verify` - Generate and verify signed IDs using options from [`Rails.application.message_verifiers`][], but fall back to verifying with the same options as `:generate_and_verify`.

* false - Generate and verify signed IDs using options from [`Rails.application.message_verifiers`][] only.

The purpose of this setting is to provide a smooth transition to a unified configuration for all message verifiers. Having a unified configuration makes it more straightforward to rotate secrets and upgrade signing algorithms.

WARNING: Setting this to false may cause old signed IDs to become unreadable if `Rails.application.message_verifiers` is not properly configured. Use [`MessageVerifiers#rotate`][ActiveSupport::MessageVerifiers#rotate] or [`MessageVerifiers#prepend`][ActiveSupport::MessageVerifiers#prepend] to configure `Rails.application.message_verifiers` with the appropriate options, such as `:digest` and `:url_safe`.

[`Rails.application.message_verifiers`]: https://api.rubyonrails.org/classes/Rails/Application.html#method-i-message_verifiers
[ActiveSupport::MessageVerifiers#rotate]: https://api.rubyonrails.org/classes/ActiveSupport/MessageVerifiers.html#method-i-rotate
[ActiveSupport::MessageVerifiers#prepend]: https://api.rubyonrails.org/classes/ActiveSupport/MessageVerifiers.html#method-i-prepend

#### `ActiveRecord::ConnectionAdapters::Mysql2Adapter.emulate_booleans` and `ActiveRecord::ConnectionAdapters::TrilogyAdapter.emulate_booleans`

Controls whether the Active Record MySQL adapter will consider all `tinyint(1)` columns as booleans. Defaults to `true`.

#### `ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.create_unlogged_tables`

Controls whether database tables created by PostgreSQL should be "unlogged", which can speed
up performance but adds a risk of data loss if the database crashes. It is
highly recommended that you do not enable this in a production environment.
Defaults to `false` in all environments.

To enable this for tests:

```ruby
# config/environments/test.rb

ActiveSupport.on_load(:active_record_postgresqladapter) do
  self.create_unlogged_tables = true
end
```

#### `ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.datetime_type`

Controls what native type the Active Record PostgreSQL adapter should use when you call `datetime` in
a migration or schema. It takes a symbol which must correspond to one of the
configured `NATIVE_DATABASE_TYPES`. The default is `:timestamp`, meaning
`t.datetime` in a migration will create a "timestamp without time zone" column.

To use "timestamp with time zone":

```ruby
# config/application.rb

ActiveSupport.on_load(:active_record_postgresqladapter) do
  self.datetime_type = :timestamptz
end
```

You should run `bin/rails db:migrate` to rebuild your schema.rb if you change this.

#### `ActiveRecord::SchemaDumper.ignore_tables`

Accepts an array of tables that should _not_ be included in any generated schema file.

**Note:** This configuration is deprecated in favor of
[`config.active_record.schema_ignored_tables`](#config-active-record-schema-ignored-tables),
and will be removed in a future Rails version. It is now an alias for that
option, so setting it also excludes the tables from the schema cache.

#### `ActiveRecord::SchemaDumper.fk_ignore_pattern`

Allows setting a different regular expression that will be used to decide
whether a foreign key's name should be dumped to db/schema.rb or not. By
default, foreign key names starting with `fk_rails_` are not exported to the
database schema dump. Defaults to `/^fk_rails_[0-9a-f]{10}$/`.

#### `config.active_record.encryption.support_unencrypted_data`

When `true`, unencrypted data can be read normally. When `false`, it will raise errors. Default: `false`.

#### `config.active_record.encryption.extend_queries`

When `true`, queries referencing deterministically encrypted attributes will be modified to include additional values if needed. Those additional values will be the clean version of the value (when `config.active_record.encryption.support_unencrypted_data` is `true`) and values encrypted with previous encryption schemes, if any (as provided with the `previous:` option). Default: `false`.

#### `config.active_record.encryption.encrypt_fixtures`

When `true`, encryptable attributes in fixtures will be automatically encrypted when loaded. Default: `false`.

#### `config.active_record.encryption.store_key_references`

When `true`, a reference to the encryption key is stored in the headers of the encrypted message. This makes for faster decryption when multiple keys are in use. Default: `false`.

#### `config.active_record.encryption.add_to_filter_parameters`

When `true`, encrypted attribute names are added automatically to [`config.filter_parameters`](#config-filter-parameters) and won't be shown in logs. Default: `true`.

#### `config.active_record.encryption.excluded_from_filter_parameters`

You can configure a list of params that won't be filtered out when `config.active_record.encryption.add_to_filter_parameters` is true. Default: `[]`.

#### `config.active_record.encryption.validate_column_size`

Adds a validation based on the column size. This is recommended to prevent storing huge values using highly compressible payloads. Default: `true`.

#### `config.active_record.encryption.primary_key`

The key or lists of keys used to derive root data encryption keys. The way they are used depends on the key provider configured. It's preferred to configure it via the `active_record_encryption.primary_key` credential.

#### `config.active_record.encryption.deterministic_key`

The key or list of keys used for deterministic encryption. It's preferred to configure it via the `active_record_encryption.deterministic_key` credential.

#### `config.active_record.encryption.key_derivation_salt`

The salt used when deriving keys. It's preferred to configure it via the `active_record_encryption.key_derivation_salt` credential.

#### `config.active_record.encryption.forced_encoding_for_deterministic_encryption`

The default encoding for attributes encrypted deterministically. You can disable
forced encoding by setting this option to `nil`. It's `Encoding::UTF_8` by default.

#### `config.active_record.encryption.hash_digest_class`

Sets the digest algorithm used by Active Record Encryption.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is      |
| --------------------- | ------------------------- |
| (original)            | `OpenSSL::Digest::SHA1`   |
| 7.1                   | `OpenSSL::Digest::SHA256` |

#### `config.active_record.encryption.support_sha1_for_non_deterministic_encryption`

Enables support for decrypting existing data encrypted using a SHA-1 digest
class. When `false`, it will only support the digest configured in
`config.active_record.encryption.hash_digest_class`.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 7.1                   | `false`              |

#### `config.active_record.encryption.compressor`

The compressor used to compress encrypted payloads. The default is `Zlib`. You can use your own compressor by setting this to a class that responds to `deflate` and `inflate`.

#### `config.active_record.protocol_adapters`

When using a URL to configure the database connection, this option provides a mapping from the protocol to the underlying
database adapter. For example, this means the environment can specify `DATABASE_URL=mysql://localhost/database` and Rails will map
`mysql` to the `mysql2` adapter, but the application can also override these mappings:

```ruby
config.active_record.protocol_adapters.mysql = "trilogy"
```

If no mapping is found, the protocol is used as the adapter name.

#### `config.active_record.deprecated_associations_options`

If present, this has to be a hash with keys `:mode` and/or `:backtrace`:

```ruby
config.active_record.deprecated_associations_options = { mode: :notify, backtrace: true }
```

* In `:warn` mode, accessing the deprecated association is reported by the
  Active Record logger. This is the default mode.

* In `:raise` mode, usage raises an `ActiveRecord::DeprecatedAssociationError`
  with a similar message and a clean backtrace in the exception object.

* In `:notify` mode, a `deprecated_association.active_record` Active Support
  notification is published. Please, see details about its payload in the
  [Active Support Instrumentation guide](active_support_instrumentation.html).

Backtraces are disabled by default. If `:backtrace` is true, warnings include a
clean backtrace in the message, and notifications have a `:backtrace` key in the
payload with an array of clean `Thread::Backtrace::Location` objects. Exceptions
always have a clean stack trace.

Clean backtraces are computed using the Active Record backtrace cleaner.

#### `config.active_record.raise_on_missing_required_finder_order_columns`

Raises an error when order dependent finder methods (e.g. `#first`, `#second`) are called without `order` values
on the relation, and the model does not have any order columns (`implicit_order_column`, `query_constraints`, or
`primary_key`) to fall back on.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 8.1                   | `true`               |

#### `config.active_record.shuffle_unordered_selects`

Shuffles the rows of every `SELECT` Active Record generates that has no `ORDER BY` clause.

The order of such a query is not specified: the database is free to return the rows in any order, and that
order can change when an index is added, when the data grows, or when the query planner changes its mind.
Enabling this option makes the lack of order explicit, so code and tests that accidentally depend on the
order a particular database happens to return today fail immediately instead of breaking later.

```ruby
# config/environments/test.rb
config.active_record.shuffle_unordered_selects = true
```

The order is fully random and drawn again on every execution, so a query cannot accidentally settle into an
order that an assertion keeps passing against.

The option is best effort, and two things bound what it can surface.

The first is that Active Record has to recognise the query, which it does from the Arel it built. A query that
reaches it as already-compiled SQL is left alone: SQL you wrote yourself, and association loading, `find` and
`find_by`, which are served from a precompiled statement by `ActiveRecord::StatementCache`. Relations, `pluck`,
calculations and eager loading are covered, inside a query cache block or out.

The second is that rows are shuffled after the database has returned them, so the option cannot change *which*
rows come back. Queries ending in `LIMIT 1` are unaffected — `find`, `find_by`, `take`, `pick`, `exists?`,
`has_one` and `belongs_to` — which makes this weaker than SQLite's `reverse_unordered_selects` pragma. A query
with an `ORDER BY` is never shuffled even when that ordering is not a total order, so ties on a non-unique
column stay hidden. And the SQL in your log is the SQL that was sent, so replaying it by hand will not
reproduce the order your application saw.

This is a development aid intended for the test or development environments, and it is never enabled by
`config.load_defaults`.

The default value is `false`.

### Configuring Action Controller

`config.action_controller` includes a number of configuration settings:

#### `config.action_controller.asset_host`

Sets the host for the assets. Useful when CDNs are used for hosting assets rather than the application server itself. You should only use this if you have a different configuration for Action Mailer, otherwise use `config.asset_host`.

#### `config.action_controller.perform_caching`

Configures whether the application should perform the caching features provided by the Action Controller component. Set to `false` in the development environment, `true` in production. If it's not specified, the default will be `true`.

#### `config.action_controller.default_static_extension`

Configures the extension used for cached pages. Defaults to `.html`.

#### `config.action_controller.include_all_helpers`

Configures whether all view helpers are available everywhere or are scoped to the corresponding controller. If set to `false`, `UsersHelper` methods are only available for views rendered as part of `UsersController`. If `true`, `UsersHelper` methods are available everywhere. The default configuration behavior (when this option is not explicitly set to `true` or `false`) is that all view helpers are available to each controller.

#### `config.action_controller.logger`

Accepts a logger conforming to the interface of Log4r or the default Ruby Logger class, which is then used to log information from Action Controller. Set to `nil` to disable logging.

#### `config.action_controller.request_forgery_protection_token`

Sets the token parameter name for RequestForgery. Calling `protect_from_forgery` sets it to `:authenticity_token` by default.

#### `config.action_controller.allow_forgery_protection`

Enables or disables CSRF protection. By default this is `false` in the test environment and `true` in all other environments.

#### `config.action_controller.forgery_protection_origin_check`

Configures whether the HTTP `Origin` header should be checked against the site's origin as an additional CSRF defense.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 5.0                   | `true`               |

#### `config.action_controller.per_form_csrf_tokens`

Configures whether CSRF tokens are only valid for the method/action they were generated for.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 5.0                   | `true`               |

#### `config.action_controller.forgery_protection_verification_strategy`

Configures how Rails verifies requests for CSRF protection. Available strategies are:

* `:header_only` - Uses the `Sec-Fetch-Site` header sent by modern browsers to verify
  that requests originate from the same site. Requests without a valid header are rejected.
  This is simpler and more secure but only works with browsers that support the
  [Fetch Metadata Request Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Sec-Fetch-Site).

* `:header_or_legacy_token` - A hybrid approach that checks the `Sec-Fetch-Site` header first.
  If the header indicates same-origin or same-site, the request is allowed. When the
  header is missing or has the value "none", it falls back to checking the authenticity
  token. This supports older browsers while logging when fallback occurs.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is         |
| --------------------- | ---------------------------- |
| (original)            | `:header_or_legacy_token`    |
| 8.2                   | `:header_only`               |

#### `config.action_controller.default_protect_from_forgery`

Determines whether forgery protection is added on `ActionController::Base`.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 5.2                   | `true`               |

#### `config.action_controller.default_protect_from_forgery_with`

Configures the default strategy used when calling `protect_from_forgery` without the `:with` option.
Defaults to `:null_session`, but will change to `:exception` in a future version of Rails.

Applications can opt into the new behavior early by setting:

```ruby
config.action_controller.default_protect_from_forgery_with = :exception
```

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:null_session`      |
| 8.2                   | `:exception`         |

#### `config.action_controller.relative_url_root`

Can be used to tell Rails that you are [deploying to a subdirectory](
configuring.html#deploy-to-a-subdirectory-relative-url-root). The default is
[`config.relative_url_root`](#config-relative-url-root).

#### `config.action_controller.permit_all_parameters`

Sets all the parameters for mass assignment to be permitted by default. The default value is `false`.

#### `config.action_controller.action_on_unpermitted_parameters`

Controls behavior when parameters that are not explicitly permitted are found. The default value is `:log` in test and development environments, `false` otherwise. The values can be:

* `false` to take no action
* `:log` to emit an `ActiveSupport::Notifications.instrument` event on the `unpermitted_parameters.action_controller` topic and log at the DEBUG level
* `:raise` to raise a `ActionController::UnpermittedParameters` exception

#### `config.action_controller.always_permitted_parameters`

Sets a list of permitted parameters that are permitted by default. The default values are `['controller', 'action']`.

#### `config.action_controller.enable_fragment_cache_logging`

Determines whether to log fragment cache reads and writes in verbose format as follows:

```
Read fragment views/v1/2914079/v1/2914079/recordings/70182313-20160225015037000000/d0bdf2974e1ef6d31685c3b392ad0b74 (0.6ms)
Rendered messages/_message.html.erb in 1.2 ms [cache hit]
Write fragment views/v1/2914079/v1/2914079/recordings/70182313-20160225015037000000/3b4e249ac9d168c617e32e84b99218b5 (1.1ms)
Rendered recordings/threads/_thread.html.erb in 1.5 ms [cache miss]
```

By default it is set to `false` which results in following output:

```
Rendered messages/_message.html.erb in 1.2 ms [cache hit]
Rendered recordings/threads/_thread.html.erb in 1.5 ms [cache miss]
```

#### `config.action_controller.raise_on_missing_callback_actions`

Raises an `AbstractController::ActionNotFound` when the action specified in callback's `:only` or `:except` options is missing in the controller.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.1                   | `true` (development and test), `false` (other envs)|


#### `config.action_controller.raise_on_open_redirects`

Protect an application from unintentionally redirecting to an external host
(also known as an "open redirect") by making external redirects opt-in.

When this configuration is set to `true`, an
`ActionController::Redirecting::UnsafeRedirectError` will be raised when a URL
with an external host is passed to [redirect_to][]. If an open redirect should
be allowed, then `allow_other_host: true` can be added to the call to
`redirect_to`.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |

[redirect_to]: https://api.rubyonrails.org/classes/ActionController/Redirecting.html#method-i-redirect_to

#### `config.action_controller.action_on_open_redirect`

Controls how Rails handles open redirect attempts (redirects to external hosts).

**Note:** This configuration replaces the deprecated [`config.action_controller.raise_on_open_redirects`](#config-action-controller-raise-on-open-redirects)
option, which will be removed in a future Rails version. The new configuration provides more
flexible control over open redirect protection.

When set to `:log`, Rails will log a warning when an open redirect is detected.
When set to `:notify`, Rails will publish an `open_redirect.action_controller`
notification event. When set to `:raise`, Rails will raise an
`ActionController::Redirecting::UnsafeRedirectError`.

If `raise_on_open_redirects` is set to `true`, it will take precedence
over this configuration for backward compatibility, effectively forcing `:raise`
behavior.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:log`               |
| 7.0                   | `:raise`             |

#### `config.action_controller.action_on_path_relative_redirect`

Controls how Rails handles paths relative URL redirects.

When set to `:log` (default), Rails will log a warning when a path relative URL redirect
is detected. When set to `:notify`, Rails will publish an
`unsafe_redirect.action_controller` notification event. When set to `:raise`, Rails
will raise an `ActionController::Redirecting::UnsafeRedirectError`.

This helps detect potentially unsafe redirects that could be exploited for open
redirect attacks.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:log`               |
| 8.1                   | `:raise`             |


#### `config.action_controller.log_query_tags_around_actions`

Determines whether controller context for query tags will be automatically
updated via an `around_filter`. The default value is `true`.

#### `config.action_controller.wrap_parameters_by_default`

Before Rails 7.0, new applications were generated with an initializer named
`wrap_parameters.rb` that enabled parameter wrapping in `ActionController::Base`
for JSON requests.

Setting this configuration value to `true` has the same behavior as the
initializer, allowing applications to remove the initializer if they do not wish
to customize parameter wrapping behavior.

Regardless of this value, applications can continue to customize the parameter
wrapping behavior as before in an initializer or per controller.

See [`ParamsWrapper`][params_wrapper] for more information on parameter
wrapping.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.0                   | `true`               |

[params_wrapper]: https://api.rubyonrails.org/classes/ActionController/ParamsWrapper.html

#### `config.action_controller.allowed_redirect_hosts`

Specifies a list of allowed hosts for redirects. `redirect_to` will allow redirects to them without raising an
`UnsafeRedirectError` error.

#### `ActionController::Base.wrap_parameters`

Configures the [`ParamsWrapper`](https://api.rubyonrails.org/classes/ActionController/ParamsWrapper.html). This can be called at
the top level, or on individual controllers.

#### `config.action_controller.escape_json_responses`

Configures the JSON renderer to escape HTML entities and Unicode characters that are invalid in JavaScript.

This is useful if you relied on the JSON response having those characters escaped to embed the JSON document in
\<script> tags in HTML.

This is mainly for compatibility when upgrading Rails applications, otherwise you can use the `:escape` option for
`render json:` in specific controller actions.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 8.1                   | `false`              |

#### `config.action_controller.rescue_from_event_backtrace`

Configures the `event_backtrace` attribute in the payload of `rescue_from_handled.action_controller` notifications, and `action_controller.rescue_from_handled` events.

* `:array` - Stores the backtrace as an array of strings.
* `nil` - Stores the backtrace as the first string of the backtrace, stripping the `Rails.root` from the controller path.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `nil`                |
| 8.2                   | `:array`             |

### Configuring Action Dispatch

#### `config.action_dispatch.cookies_serializer`

Specifies which serializer to use for cookies. Accepts the same values as
[`config.active_support.message_serializer`](#config-active-support-message-serializer),
plus `:hybrid` which is an alias for `:json_allow_marshal`.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:marshal`           |
| 7.0                   | `:json`              |

#### `config.action_dispatch.debug_exception_log_level`

Configures the log level used by the [`ActionDispatch::DebugExceptions`][]
middleware when logging uncaught exceptions during requests.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:fatal`             |
| 7.1                   | `:error`             |

[`ActionDispatch::DebugExceptions`]: https://api.rubyonrails.org/classes/ActionDispatch/DebugExceptions.html

#### `config.action_dispatch.default_headers`

Is a hash with HTTP headers that are set by default in each response.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | <pre><code>{<br>  "X-Frame-Options" => "SAMEORIGIN",<br>  "X-XSS-Protection" => "1; mode=block",<br>  "X-Content-Type-Options" => "nosniff",<br>  "X-Download-Options" => "noopen",<br>  "X-Permitted-Cross-Domain-Policies" => "none",<br>  "Referrer-Policy" => "strict-origin-when-cross-origin"<br>}</code></pre> |
| 7.0                   | <pre><code>{<br>  "X-Frame-Options" => "SAMEORIGIN",<br>  "X-XSS-Protection" => "0",<br>  "X-Content-Type-Options" => "nosniff",<br>  "X-Download-Options" => "noopen",<br>  "X-Permitted-Cross-Domain-Policies" => "none",<br>  "Referrer-Policy" => "strict-origin-when-cross-origin"<br>}</code></pre> |
| 7.1                   | <pre><code>{<br>  "X-Frame-Options" => "SAMEORIGIN",<br>  "X-XSS-Protection" => "0",<br>  "X-Content-Type-Options" => "nosniff",<br>  "X-Permitted-Cross-Domain-Policies" => "none",<br>  "Referrer-Policy" => "strict-origin-when-cross-origin"<br>}</code></pre> |
| 8.2                   | <pre><code>{<br>  "X-Frame-Options" => "SAMEORIGIN"<br>  "X-Content-Type-Options" => "nosniff",<br>  "X-Permitted-Cross-Domain-Policies" => "none",<br>  "Referrer-Policy" => "strict-origin-when-cross-origin"<br>}</code></pre> |

#### `config.action_dispatch.default_charset`

Specifies the default character set for all renders. Defaults to `nil`.

#### `config.action_dispatch.tld_length`

Sets the TLD (top-level domain) length for the application. Defaults to `1`.

#### `config.action_dispatch.domain_extractor`

Configures the domain extraction strategy used by Action Dispatch for parsing host names into domain and subdomain components. This must be an object that responds to `domain_from(host, tld_length)` and `subdomains_from(host, tld_length)` methods.

Defaults to `ActionDispatch::Http::URL::DomainExtractor`, which provides the standard domain parsing logic. You can provide a custom extractor to implement specialized domain parsing behavior:

```ruby
class CustomDomainExtractor
  def self.domain_from(host, tld_length)
    # Custom domain extraction logic
  end

  def self.subdomains_from(host, tld_length)
    # Custom subdomain extraction logic
  end
end

config.action_dispatch.domain_extractor = CustomDomainExtractor
```

#### `config.action_dispatch.ignore_accept_header`

Is used to determine whether to ignore accept headers from a request. Defaults to `false`.

#### `config.action_dispatch.strict_accept_header`

Controls whether an `Accept` header containing `*/*` forces an HTML response.
When enabled, Rails honors more specific types instead — e.g. `Accept:
application/json, */*` returns JSON instead of HTML.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 8.2                   | `true`               |

#### `config.action_dispatch.x_sendfile_header`

Specifies server specific X-Sendfile header. This is useful for accelerated file sending from server. For example it can be set to 'X-Sendfile' for Apache.

#### `config.action_dispatch.http_auth_salt`

Sets the HTTP Auth salt value. Defaults
to `'http authentication'`.

#### `config.action_dispatch.signed_cookie_salt`

Sets the signed cookies salt value.
Defaults to `'signed cookie'`.

#### `config.action_dispatch.encrypted_cookie_salt`

Sets the encrypted cookies salt value. Defaults to `'encrypted cookie'`.

#### `config.action_dispatch.encrypted_signed_cookie_salt`

Sets the signed encrypted cookies salt value. Defaults to `'signed encrypted
cookie'`.

#### `config.action_dispatch.authenticated_encrypted_cookie_salt`

Sets the authenticated encrypted cookie salt. Defaults to `'authenticated
encrypted cookie'`.

#### `config.action_dispatch.encrypted_cookie_cipher`

Sets the cipher to be used for encrypted cookies. This defaults to
`"aes-256-gcm"`.

#### `config.action_dispatch.signed_cookie_digest`

Sets the digest to be used for signed cookies. This defaults to `"SHA1"`.

#### `config.action_dispatch.cookies_rotations`

Allows rotating secrets, ciphers, and digests for encrypted and signed cookies.

#### `config.action_dispatch.use_authenticated_cookie_encryption`

Controls whether signed and encrypted cookies use the AES-256-GCM cipher or the
older AES-256-CBC cipher.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 5.2                   | `true`               |

#### `config.action_dispatch.use_cookies_with_metadata`

Enables writing cookies with the purpose metadata embedded.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 6.0                   | `true`               |

#### `config.action_dispatch.perform_deep_munge`

Configures whether `deep_munge` method should be performed on the parameters.
See [Security Guide](security.html#unsafe-query-generation) for more
information. It defaults to `true`.

#### `config.action_dispatch.rescue_responses`

Configures what exceptions are assigned to an HTTP status. It accepts a hash and you can specify pairs of exception/status.

```ruby
# It's good to use #[]= or #merge! to respect the default values
config.action_dispatch.rescue_responses["MyAuthenticationError"] = :unauthorized
```

Use `ActionDispatch::ExceptionWrapper.rescue_responses` to observe the configuration. By default, it is defined as:

```ruby
{
  "ActionController::RoutingError" => :not_found,
  "AbstractController::ActionNotFound" => :not_found,
  "ActionController::MethodNotAllowed" => :method_not_allowed,
  "ActionController::UnknownHttpMethod" => :method_not_allowed,
  "ActionController::NotImplemented" => :not_implemented,
  "ActionController::UnknownFormat" => :not_acceptable,
  "ActionDispatch::Http::MimeNegotiation::InvalidType" => :not_acceptable,
  "ActionController::MissingExactTemplate" => :not_acceptable,
  "ActionController::InvalidAuthenticityToken" => :unprocessable_entity,
  "ActionController::InvalidCrossOriginRequest" => :unprocessable_entity,
  "ActionDispatch::Http::Parameters::ParseError" => :bad_request,
  "ActionController::BadRequest" => :bad_request,
  "ActionController::ParameterMissing" => :bad_request,
  "Rack::QueryParser::ParameterTypeError" => :bad_request,
  "Rack::QueryParser::InvalidParameterError" => :bad_request,
  "ActiveRecord::RecordNotFound" => :not_found,
  "ActiveRecord::StaleObjectError" => :conflict,
  "ActiveRecord::RecordInvalid" => :unprocessable_entity,
  "ActiveRecord::RecordNotSaved" => :unprocessable_entity
}
```

Any exceptions that are not configured will be mapped to 500 Internal Server Error.

#### `config.action_dispatch.wrapper_exceptions`

Configures which exceptions are unwrapped. Wrapper exceptions will have their cause reported by the exception wrapper
instead of themselves.

```ruby
config.action_dispatch.wrapper_exceptions += [WrapperException]

begin
  raise OriginalException
rescue OriginalException
  raise WrapperException
end
```

In the above example the `WrapperException` will be unwrapped and the `OriginalException` will be reported.

Use `ActionDispatch::ExceptionWrapper.wrapper_exceptions` to observe the configuration. By default, it is defined as:

```ruby
[
  "ActionView::Template::Error"
]
```

#### `config.action_dispatch.silent_exceptions`

Configures which exceptions should not fall back to showing framework-level backtraces when there is no application
backtrace. This is useful for silencing noisy backtraces for exceptions raised at the framework or plugin level.

Use `ActionDispatch::ExceptionWrapper.silent_exceptions` to observe the configuration. By default, it is defined as:

```ruby
[
  "ActionController::RoutingError",
  "ActionDispatch::Http::MimeNegotiation::InvalidType"
]
```

#### `config.action_dispatch.rescue_templates`

Configures the templates used to render exceptions. It accepts a hash and you can specify pairs of exception => template.

Use `ActionDispatch::ExceptionWrapper.rescue_templates` to observe the configuration. By default, it is defined as:

```ruby
{
  "ActionView::MissingTemplate"            => "missing_template",
  "ActionController::RoutingError"         => "routing_error",
  "AbstractController::ActionNotFound"     => "unknown_action",
  "ActiveRecord::StatementInvalid"         => "invalid_statement",
  "ActionView::Template::Error"            => "template_error",
  "ActionController::MissingExactTemplate" => "missing_exact_template",
}
```

All exceptions that are not configured will map to Rails' built in diagnostics template.

#### `config.action_dispatch.cookies_same_site_protection`

Configures the default value of the `SameSite` attribute when setting cookies.
When set to `nil`, the `SameSite` attribute is not added. To allow the value of
the `SameSite` attribute to be configured dynamically based on the request, a
proc may be specified. For example:

```ruby
config.action_dispatch.cookies_same_site_protection = ->(request) do
  :strict unless request.user_agent == "TestAgent"
end
```

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `nil`                |
| 6.1                   | `:lax`               |

#### `config.action_dispatch.ssl_default_redirect_status`

Configures the default HTTP status code used when redirecting non-GET/HEAD
requests from HTTP to HTTPS in the `ActionDispatch::SSL` middleware.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `307`                |
| 6.1                   | `308`                |

#### `config.action_dispatch.log_rescued_responses`

Enables logging those unhandled exceptions configured in `rescue_responses`. It
defaults to `true`.

#### `config.action_dispatch.show_exceptions`

The `config.action_dispatch.show_exceptions` configuration controls how Action Pack (specifically the [`ActionDispatch::ShowExceptions`](/configuring.html#actiondispatch-showexceptions) middleware) handles exceptions raised while responding to requests.

Setting the value to `:all` configures Action Pack to rescue from exceptions and render corresponding error pages. For example, Action Pack would rescue from an `ActiveRecord::RecordNotFound` exception and render the contents of `public/404.html` with a `404 Not Found` status code.

Setting the value to `:rescuable` configures Action Pack to rescue from exceptions defined in [`config.action_dispatch.rescue_responses`](/configuring.html#config-action-dispatch-rescue-responses), and raise all others. For example, Action Pack would rescue from `ActiveRecord::RecordNotFound`, but would raise a `NoMethodError`.

Setting the value to `:none` configures Action Pack to raise all exceptions.

* `:all` - render error pages for all exceptions
* `:rescuable` - render error pages for exceptions declared by [`config.action_dispatch.rescue_responses`](/configuring.html#config-action-dispatch-rescue-responses)
* `:none` - raise all exceptions

| Starting with version | The default value is  |
| --------------------- | --------------------- |
| (original)            | `true`                |
| 7.1                   | `:all`                |

#### `config.action_dispatch.strict_freshness`

Configures whether the `ActionDispatch::ETag` middleware should prefer the `ETag` header over the `Last-Modified` header when both are present in the response.

If set to `true`, when both headers are present only the `ETag` is considered as specified by RFC 7232 section 6.

If set to `false`, when both headers are present, both headers are checked and both need to match for the response to be considered fresh.

| Starting with version | The default value is  |
| --------------------- | --------------------- |
| (original)            | `false`               |
| 8.0                   | `true`                |

#### `config.action_dispatch.always_write_cookie`

Cookies will be written at the end of a request if they marked as insecure, if the request is made over SSL, or if the request is made to an onion service.

If set to `true`, cookies will be written even if this criteria is not met.

This defaults to `true` in `development`, and `false` in all other environments.

#### `config.action_dispatch.verbose_redirect_logs`

Specifies if source locations of redirects should be logged below relevant log lines. By default, the flag is `true` in development and `false` in all other environments.

#### `ActionDispatch::Callbacks.before`

Takes a block of code to run before the request.

#### `ActionDispatch::Callbacks.after`

Takes a block of code to run after the request.

### Configuring Action View

`config.action_view` includes a small number of configuration settings:

#### `config.action_view.cache_template_loading`

Controls whether or not templates should be reloaded on each request. Defaults to `!config.enable_reloading`.

#### `config.action_view.field_error_proc`

Provides an HTML generator for displaying errors that come from Active Model. The block is evaluated within
the context of an Action View template. The default is

```ruby
Proc.new { |html_tag, instance| content_tag :div, html_tag, class: "field_with_errors" }
```

#### `config.action_view.default_form_builder`

Tells Rails which form builder to use by default. The default is
`ActionView::Helpers::FormBuilder`. If you want your form builder class to be
loaded after initialization (so it's reloaded on each request in development),
you can pass it as a `String`.

#### `config.action_view.logger`

Accepts a logger conforming to the interface of Log4r or the default Ruby Logger class, which is then used to log information from Action View. Set to `nil` to disable logging.

#### `config.action_view.erb_trim_mode`

Controls if certain ERB syntax should trim. It defaults to `'-'`, which turns on trimming of tail spaces and newline when using `<%= -%>` or `<%= =%>`. Setting this to anything else will turn off trimming support.

#### `config.action_view.erb_implementation`

Controls the default ERB implementation to use. It defaults to `Erubi`.

#### `config.action_view.escape_ignore_list`

Control whether template should be escaped based on the mime type. Defaults to `["text/plain"]`.

#### `config.action_view.strip_trailing_newlines`

Strip trailing newlines from rendered output. Defaults to `false`.

#### `config.action_view.frozen_string_literal`

Compiles the ERB template with the `# frozen_string_literal: true` magic comment, making all string literals frozen and saving allocations. Set to `true` to enable it for all views.

#### `config.action_view.embed_authenticity_token_in_remote_forms`

Allows you to set the default behavior for `authenticity_token` in forms with
`remote: true`. By default it's set to `false`, which means that remote forms
will not include `authenticity_token`, which is helpful when you're
fragment-caching the form. Remote forms get the authenticity from the `meta`
tag, so embedding is unnecessary unless you support browsers without
JavaScript. In such case you can either pass `authenticity_token: true` as a
form option or set this config setting to `true`.

#### `config.action_view.prefix_partial_path_with_controller_namespace`

Determines whether or not partials are looked up from a subdirectory in templates rendered from namespaced controllers. For example, consider a controller named `Admin::ArticlesController` which renders this template:

```erb
<%= render @article %>
```

The default setting is `true`, which uses the partial at `/admin/articles/_article.erb`. Setting the value to `false` would render `/articles/_article.erb`, which is the same behavior as rendering from a non-namespaced controller such as `ArticlesController`.

#### `config.action_view.automatically_disable_submit_tag`

Determines whether `submit_tag` should automatically disable on click, this
defaults to `true`.

#### `config.action_view.debug_missing_translation`

Determines whether to wrap the missing translations key in a `<span>` tag or not. This defaults to `true`.

#### `config.action_view.form_with_generates_remote_forms`

Determines whether `form_with` generates remote forms or not.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| 5.1                   | `true`               |
| 6.1                   | `false`              |

#### `config.action_view.form_with_generates_ids`

Determines whether `form_with` generates ids on inputs.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 5.2                   | `true`               |

#### `config.action_view.default_enforce_utf8`

Determines whether forms are generated with a hidden tag that forces older versions of Internet Explorer to submit forms encoded in UTF-8.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 6.0                   | `false`              |

#### `config.action_view.image_loading`

Specifies a default value for the `loading` attribute of `<img>` tags rendered by the `image_tag` helper. For example, when set to `"lazy"`, `<img>` tags rendered by `image_tag` will include `loading="lazy"`, which [instructs the browser to wait until an image is near the viewport to load it](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/loading#lazy). (This value can still be overridden per image by passing e.g. `loading: "eager"` to `image_tag`.) Defaults to `nil`.

#### `config.action_view.image_decoding`

Specifies a default value for the `decoding` attribute of `<img>` tags rendered by the `image_tag` helper. Defaults to `nil`.

#### `config.action_view.annotate_rendered_view_with_filenames`

Determines whether to annotate rendered view with template file names. This defaults to `false`.

#### `config.action_view.preload_links_header`

Determines whether `javascript_include_tag` and `stylesheet_link_tag` will generate a `link` header that preload assets.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `nil`                |
| 6.1                   | `true`               |

#### `config.action_view.button_to_generates_button_tag`

When `false`, `button_to` will render a `<button>` or an `<input>` inside a
`<form>` depending on how content is passed (`<form>` omitted for brevity):

```erb
<%= button_to "Content", "/" %>
# => <input type="submit" value="Content">

<%= button_to "/" do %>
  Content
<% end %>
# => <button type="submit">Content</button>
```

Setting this value to `true` makes `button_to` generate a `<button>` tag inside
the `<form>` in both cases.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.0                   | `true`               |

#### `config.action_view.apply_stylesheet_media_default`

Determines whether `stylesheet_link_tag` will render `screen` as the default
value for the `media` attribute when it's not provided.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 7.0                   | `false`              |

#### `config.action_view.prepend_content_exfiltration_prevention`

Determines whether or not the `form_tag` and `button_to` helpers will produce HTML tags prepended with browser-safe (but technically invalid) HTML that guarantees their contents cannot be captured by any preceding unclosed tags. The default value is `false`.

#### `config.action_view.sanitizer_vendor`

Configures the set of HTML sanitizers used by Action View by setting `ActionView::Helpers::SanitizeHelper.sanitizer_vendor`. The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is                 | Which parses markup as |
|-----------------------|--------------------------------------|------------------------|
| (original)            | `Rails::HTML4::Sanitizer`            | HTML4                  |
| 7.1                   | `Rails::HTML5::Sanitizer` (see NOTE) | HTML5                  |

NOTE: `Rails::HTML5::Sanitizer` is not supported on JRuby, so on JRuby platforms Rails will fall back to `Rails::HTML4::Sanitizer`.

#### `config.action_view.remove_hidden_field_autocomplete`

When enabled, hidden inputs generated by `form_tag`, `token_tag`, `method_tag`, and the hidden parameter fields included in `button_to` forms will omit the `autocomplete="off"` attribute.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 8.1                   | `true`               |

#### `config.action_view.render_tracker`

Configures the strategy for tracking dependencies between Action View templates.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:regex`             |
| 8.1                   | `:ruby`              |

### Configuring Action Mailbox

`config.action_mailbox` provides the following configuration options:

#### `config.action_mailbox.logger`

Contains the logger used by Action Mailbox. It accepts a logger conforming to the interface of Log4r or the default Ruby Logger class. The default is `Rails.logger`.

```ruby
config.action_mailbox.logger = ActiveSupport::Logger.new(STDOUT)
```

#### `config.action_mailbox.incinerate_after`

Accepts an `ActiveSupport::Duration` indicating how long after processing `ActionMailbox::InboundEmail` records should be destroyed. It defaults to `30.days`.

```ruby
# Incinerate inbound emails 14 days after processing.
config.action_mailbox.incinerate_after = 14.days
```

#### `config.action_mailbox.queues.incineration`

Accepts a symbol indicating the Active Job queue to use for incineration jobs.
When this option is `nil`, incineration jobs are sent to the default Active Job
queue (see [`config.active_job.default_queue_name`][]).

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:action_mailbox_incineration` |
| 6.1                   | `nil`                |

#### `config.action_mailbox.queues.routing`

Accepts a symbol indicating the Active Job queue to use for routing jobs. When
this option is `nil`, routing jobs are sent to the default Active Job queue (see
[`config.active_job.default_queue_name`][]).

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:action_mailbox_routing` |
| 6.1                   | `nil`                |

#### `config.action_mailbox.storage_service`

Accepts a symbol indicating the Active Storage service to use for uploading emails. When this option is `nil`, emails are uploaded to the default Active Storage service (see `config.active_storage.service`).

### Configuring Action Mailer

There are a number of settings available on `config.action_mailer`:

#### `config.action_mailer.asset_host`

Sets the host for the assets. Useful when CDNs are used for hosting assets rather than the application server itself. You should only use this if you have a different configuration for Action Controller, otherwise use `config.asset_host`.

#### `config.action_mailer.logger`

Accepts a logger conforming to the interface of Log4r or the default Ruby Logger class, which is then used to log information from Action Mailer. Set to `nil` to disable logging.

#### `config.action_mailer.delivery_method`

Defines the delivery method. The following options are available:

* `:smtp` - Sends email using SMTP. Configure it with
  [`config.action_mailer.smtp_settings`][]. This is the default.
* `:sendmail` - Sends email using sendmail. Configure it with
  [`config.action_mailer.sendmail_settings`][].
* `:file` - Saves emails to files. Configure it with
  [`config.action_mailer.file_settings`][].
* `:test` - Saves emails to the `ActionMailer::Base.deliveries` array.

You can also use a custom delivery method by either:

* Setting `config.action_mailer.delivery_method` to a custom delivery method
  object. The object must accept settings during initialization and respond to
  `deliver!(mail)`. See the Mail gem's [`Mail::SMTP` delivery method][] for an
  example implementation.
* Registering a custom delivery method with
  [`ActionMailer::Base.add_delivery_method`][] and setting
  `config.action_mailer.delivery_method` to its registered name.

See the [configuration section in the Action Mailer guide][] for configuration
examples.

[`config.action_mailer.smtp_settings`]: #config-action-mailer-smtp-settings
[`config.action_mailer.sendmail_settings`]: #config-action-mailer-sendmail-settings
[`config.action_mailer.file_settings`]: #config-action-mailer-file-settings
[`ActionMailer::Base.add_delivery_method`]: https://api.rubyonrails.org/classes/ActionMailer/DeliveryMethods/ClassMethods.html#method-i-add_delivery_method
[`Mail::SMTP` delivery method]: https://github.com/mikel/mail/blob/master/lib/mail/network/delivery_methods/smtp.rb
[configuration section in the Action Mailer guide]: action_mailer_basics.html#action-mailer-configuration

#### `config.action_mailer.smtp_settings`

Allows detailed configuration for the `:smtp` delivery method. It accepts a hash of options, which can include any of these options:

* `:address` - Allows you to use a remote mail server. Just change it from its default "localhost" setting.
* `:port` - On the off chance that your mail server doesn't run on port 25, you can change it.
* `:domain` - If you need to specify a HELO domain, you can do it here.
* `:user_name` - If your mail server requires authentication, set the username in this setting.
* `:password` - If your mail server requires authentication, set the password in this setting.
* `:authentication` - If your mail server requires authentication, you need to specify the authentication type here. This is a symbol and one of `:plain`, `:login`, `:cram_md5`.
* `:enable_starttls` - Use STARTTLS when connecting to your SMTP server and fail if unsupported. It defaults to `false`.
* `:enable_starttls_auto` - Detects if STARTTLS is enabled in your SMTP server and starts to use it. It defaults to `true`.
* `:openssl_verify_mode` - When using TLS, you can set how OpenSSL checks the certificate. This is useful if you need to validate a self-signed and/or a wildcard certificate. This can be the name of one of the OpenSSL verify constants, `'none'` or `'peer'` - or the constant directly `OpenSSL::SSL::VERIFY_NONE` or `OpenSSL::SSL::VERIFY_PEER`, respectively.
* `:ssl/:tls` - Enables the SMTP connection to use SMTP/TLS (SMTPS: SMTP over direct TLS connection).
* `:open_timeout` - Number of seconds to wait while attempting to open a connection.
* `:read_timeout` - Number of seconds to wait until timing-out a read(2) call.

Additionally, it is possible to pass any [configuration option `Mail::SMTP` respects](https://github.com/mikel/mail/blob/master/lib/mail/network/delivery_methods/smtp.rb).

#### `config.action_mailer.smtp_timeout`

Prior to version 2.8.0, the `mail` gem did not configure any default timeouts
for its SMTP requests. This configuration enables applications to configure
default values for both `:open_timeout` and `:read_timeout` in the `mail` gem so
that requests do not end up stuck indefinitely.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `nil`                |
| 7.0                   | `5`                  |

#### `config.action_mailer.sendmail_settings`

Allows detailed configuration for the `:sendmail` delivery method. It accepts a hash of options, which can include any of these options:

* `:location` - The location of the sendmail executable. Defaults to `/usr/sbin/sendmail`.
* `:arguments` - The command line arguments. Defaults to `%w[ -i ]`.

#### `config.action_mailer.file_settings`

Configures the `:file` delivery method. It accepts a hash of options, which can include:

* `:location` - The location where files are saved. Defaults to `"#{Rails.root}/tmp/mails"`.
* `:extension` - The file extension. Defaults to the empty string.

#### `config.action_mailer.raise_delivery_errors`

Specifies whether to raise an error if email delivery cannot be completed. It defaults to `true`.

#### `config.action_mailer.perform_deliveries`

Specifies whether mail will actually be delivered and is `true` by default. It can be convenient to set it to `false` for testing.

#### `config.action_mailer.default_options`

Configures Action Mailer defaults. Use to set options like `from` or `reply_to` for every mailer. These default to:

```ruby
{
  mime_version:  "1.0",
  charset:       "UTF-8",
  content_type: "text/plain",
  parts_order:  ["text/plain", "text/enriched", "text/html"]
}
```

Assign a hash to set additional options:

```ruby
config.action_mailer.default_options = {
  from: "noreply@example.com"
}
```

#### `config.action_mailer.observers`

Registers observers which will be notified when mail is delivered.

```ruby
config.action_mailer.observers = ["MailObserver"]
```

#### `config.action_mailer.interceptors`

Registers interceptors which will be called before mail is sent.

```ruby
config.action_mailer.interceptors = ["MailInterceptor"]
```

#### `config.action_mailer.preview_interceptors`

Registers interceptors which will be called before mail is previewed.

```ruby
config.action_mailer.preview_interceptors = ["MyPreviewMailInterceptor"]
```

#### `config.action_mailer.preview_paths`

Specifies the locations of mailer previews. Appending paths to this configuration option will cause those paths to be used in the search for mailer previews.

```ruby
config.action_mailer.preview_paths << "#{Rails.root}/lib/mailer_previews"
```

#### `config.action_mailer.show_previews`

Enable or disable mailer previews. By default this is `true` in development.

```ruby
config.action_mailer.show_previews = false
```

#### `config.action_mailer.perform_caching`

Specifies whether the mailer templates should perform fragment caching or not. If it's not specified, the default will be `true`.

#### `config.action_mailer.deliver_later_queue_name`

Specifies the Active Job queue to use for the default delivery job (see
`config.action_mailer.delivery_job`). When this option is set to `nil`, delivery
jobs are sent to the default Active Job queue (see
[`config.active_job.default_queue_name`][]).

Mailer classes can override this to use a different queue. Note that this only applies when using the default delivery job. If your mailer is using a custom job, its queue will be used.

Ensure that your Active Job adapter is also configured to process the specified queue, otherwise delivery jobs may be silently ignored.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:mailers`           |
| 6.1                   | `nil`                |

#### `config.action_mailer.delivery_job`

Specifies delivery job for mail.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `ActionMailer::MailDeliveryJob` |
| 6.0                   | `"ActionMailer::MailDeliveryJob"` |

#### `config.action_mailer.raise_on_missing_callback_actions`

Mirrors `config.action_controller.raise_on_missing_callback_actions`, but applies to mailers. Defaults to `false`.

### Configuring Active Support

There are a few configuration options available in Active Support:

#### `config.active_support.bare`

Enables or disables the loading of `active_support/all` when booting Rails. Defaults to `nil`, which means `active_support/all` is loaded.

#### `config.active_support.test_order`

Sets the order in which the test cases are executed. Possible values are `:random` and `:sorted`. Defaults to `:random`.

#### `config.active_support.escape_html_entities_in_json`

Enables or disables the escaping of HTML entities in JSON serialization. Defaults to `true`.

#### `config.active_support.use_standard_json_time_format`

Enables or disables serializing dates to ISO 8601 format. Defaults to `true`.

#### `config.active_support.time_precision`

Sets the precision of JSON encoded time values. Defaults to `3`.

#### `config.active_support.hash_digest_class`

Allows configuring the digest class to use to generate non-sensitive digests, such as the ETag header.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `OpenSSL::Digest::MD5` |
| 5.2                   | `OpenSSL::Digest::SHA1` |
| 7.0                   | `OpenSSL::Digest::SHA256` |

#### `config.active_support.key_generator_hash_digest_class`

Allows configuring the digest class to use to derive secrets from the configured secret base, such as for encrypted cookies.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `OpenSSL::Digest::SHA1` |
| 7.0                   | `OpenSSL::Digest::SHA256` |

#### `config.active_support.use_authenticated_message_encryption`

Specifies whether to use AES-256-GCM authenticated encryption as the default cipher for encrypting messages instead of AES-256-CBC.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 5.2                   | `true`               |

#### `config.active_support.message_serializer`

Specifies the default serializer used by [`ActiveSupport::MessageEncryptor`][]
and [`ActiveSupport::MessageVerifier`][] instances. To make migrating between
serializers easier, the provided serializers include a fallback mechanism to
support multiple deserialization formats:

| Serializer | Serialize and deserialize | Fallback deserialize |
| ---------- | ------------------------- | -------------------- |
| `:marshal` | `Marshal` | `ActiveSupport::JSON`, `ActiveSupport::MessagePack` |
| `:json` | `ActiveSupport::JSON` | `ActiveSupport::MessagePack` |
| `:json_allow_marshal` | `ActiveSupport::JSON` | `ActiveSupport::MessagePack`, `Marshal` |
| `:message_pack` | `ActiveSupport::MessagePack` | `ActiveSupport::JSON` |
| `:message_pack_allow_marshal` | `ActiveSupport::MessagePack` | `ActiveSupport::JSON`, `Marshal` |

WARNING: `Marshal` is a potential vector for deserialization attacks in cases
where a message signing secret has been leaked. _If possible, choose a
serializer that does not support `Marshal`._

INFO: The `:message_pack` and `:message_pack_allow_marshal` serializers support
roundtripping some Ruby types that are not supported by JSON, such as `Symbol`.
They can also provide improved performance and smaller payload sizes. However,
they require the [`msgpack` gem](https://rubygems.org/gems/msgpack).

Each of the above serializers will emit a [`message_serializer_fallback.active_support`][]
event notification when they fall back to an alternate deserialization format,
allowing you to track how often such fallbacks occur.

Alternatively, you can specify any serializer object that responds to `dump` and
`load` methods. For example:

```ruby
config.active_support.message_serializer = YAML
```

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:marshal`           |
| 7.1                   | `:json_allow_marshal` |

[`ActiveSupport::MessageEncryptor`]: https://api.rubyonrails.org/classes/ActiveSupport/MessageEncryptor.html
[`ActiveSupport::MessageVerifier`]: https://api.rubyonrails.org/classes/ActiveSupport/MessageVerifier.html
[`message_serializer_fallback.active_support`]: active_support_instrumentation.html#message-serializer-fallback-active-support

#### `config.active_support.use_message_serializer_for_metadata`

When `true`, enables a performance optimization that serializes message data and
metadata together. This changes the message format, so messages serialized this
way cannot be read by older (< 7.1) versions of Rails. However, messages that
use the old format can still be read, regardless of whether this optimization is
enabled.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.1                   | `true`               |

#### `config.active_support.cache_format_version`

Specifies which serialization format to use for the cache. Possible values are
`7.0`, and `7.1`.

`7.0` serializes cache entries more efficiently.

`7.1` further improves efficiency, and allows expired and version-mismatched
cache entries to be detected without deserializing their values. It also
includes an optimization for bare string values such as view fragments.

All formats are backward and forward compatible, meaning cache entries written
in one format can be read when using another format. This behavior makes it
easy to migrate between formats without invalidating the entire cache.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| 7.0                   | `7.0`                |
| 7.1                   | `7.1`                |

#### `config.active_support.deprecation`

Configures the behavior of deprecation warnings. See
[`Deprecation::Behavior`][deprecation_behavior] for a description of the
available options.

In the default generated `config/environments` files, this is set to `:log` for
development and `:stderr` for test, and it is omitted for production in favor of
[`config.active_support.report_deprecations`](#config-active-support-report-deprecations).

[deprecation_behavior]: https://api.rubyonrails.org/classes/ActiveSupport/Deprecation/Behavior.html#method-i-behavior-3D

#### `config.active_support.disallowed_deprecation`

Configures the behavior of disallowed deprecation warnings. See
[`Deprecation::Behavior`][deprecation_behavior] for a description of the
available options.

This option is intended for development and test. For production, favor
[`config.active_support.report_deprecations`](#config-active-support-report-deprecations).

#### `config.active_support.disallowed_deprecation_warnings`

Configures deprecation warnings that the Application considers disallowed. This allows, for example, specific deprecations to be treated as hard failures.

#### `config.active_support.report_deprecations`

When `false`, disables all deprecation warnings, including disallowed deprecations, from the [application’s deprecators](https://api.rubyonrails.org/classes/Rails/Application.html#method-i-deprecators). This includes all the deprecations from Rails and other gems that may add their deprecator to the collection of deprecators, but may not prevent all deprecation warnings emitted from ActiveSupport::Deprecation.

In the default generated `config/environments` files, this is set to `false` for production.

#### `config.active_support.isolation_level`

Configures the locality of most of Rails internal state. If you use a fiber based server or job processor (e.g. `falcon`), you should set it to `:fiber`. Otherwise it is best to use `:thread` locality. Defaults to `:thread`.

#### `config.active_support.executor_around_test_case`

Configure the test suite to call `Rails.application.executor.wrap` around test cases.
This makes test cases behave closer to an actual request or job.
Several features that are normally disabled in test, such as Active Record query cache
and asynchronous queries will then be enabled.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.0                   | `true`               |

#### `ActiveSupport::Logger.silencer`

Is set to `false` to disable the ability to silence logging in a block. The default is `true`.

#### `ActiveSupport::Cache::Store.logger`

Specifies the logger to use within cache store operations.

#### `ActiveSupport.utc_to_local_returns_utc_offset_times`

Configures [`ActiveSupport::TimeZone.utc_to_local`][] to return a time with a UTC
offset instead of a UTC time incorporating that offset.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 6.1                   | `true`               |

[`ActiveSupport::TimeZone.utc_to_local`]: https://api.rubyonrails.org/classes/ActiveSupport/TimeZone.html#method-i-utc_to_local

#### `config.active_support.raise_on_invalid_cache_expiration_time`

Specifies whether an `ArgumentError` should be raised if `Rails.cache`
[`fetch`][ActiveSupport::Cache::Store#fetch] or [`write`][ActiveSupport::Cache::Store#write]
are given an invalid `expires_at` or `expires_in` time.

Options are `true` and `false`. If `false`, the exception will be reported
as `handled` and logged instead.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.1                   | `true`               |

[ActiveSupport::Cache::Store#fetch]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-fetch
[ActiveSupport::Cache::Store#write]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-write

#### `ActiveSupport.raise_on_invalid_time_zone_parse`

Specifies whether [`ActiveSupport::TimeZone#parse`][] raises `ArgumentError`
for strings that contain no recognizable date information (e.g. `"foobar"`).

Historically, `TimeZone#parse` had two different behaviors for invalid
strings: it returned `nil` when the string contained no recognizable date
information, but raised `ArgumentError` when the string looked like a date
but contained out-of-range values (e.g. `"9000"`, which is interpreted as
month 90).

When set to `true`, both cases raise `ArgumentError`, which matches the
Ruby standard library's `Time.parse` and makes failures less likely to
go unnoticed.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 8.2                   | `true`               |

[`ActiveSupport::TimeZone#parse`]: https://api.rubyonrails.org/classes/ActiveSupport/TimeZone.html#method-i-parse

#### `config.active_support.event_reporter_context_store`

Configures a custom context store for the Event Reporter. The context store is used to manage metadata that should be attached to every event emitted by the reporter.

By default, the Event Reporter uses `ActiveSupport::EventContext` which stores context in fiber-local storage.

To use a custom context store, set this config to a class that implements the context store interface:

```ruby
# config/application.rb
config.active_support.event_reporter_context_store = CustomContextStore

class CustomContextStore
  class << self
    def context
      # Return the context hash
    end

    def set_context(context_hash)
      # Append context_hash to the existing context store
    end

    def clear
      # Clear the stored context
    end
  end
end
```

Defaults to `nil`, which means the default `ActiveSupport::EventContext` store is used.

#### `config.active_support.escape_js_separators_in_json`

Specifies whether LINE SEPARATOR (U+2028) and PARAGRAPH SEPARATOR (U+2029) are escaped when generating JSON.

Historically these characters were not valid inside JavaScript literal strings but that changed in ECMAScript 2019.
As such it's no longer a concern in modern browsers: https://caniuse.com/mdn-javascript_builtins_json_json_superset.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `true`               |
| 8.1                   | `false`              |

### Configuring Active Job

`config.active_job` provides the following configuration options:

#### `config.active_job.queue_adapter`

Sets the adapter for the queuing backend. The default adapter is `:async`. For an up-to-date list of built-in adapters see the [ActiveJob::QueueAdapters API documentation](https://api.rubyonrails.org/classes/ActiveJob/QueueAdapters.html).

```ruby
# Be sure to have the adapter's gem in your Gemfile
# and follow the adapter's specific installation
# and deployment instructions.
config.active_job.queue_adapter = :solid_queue
```

#### `config.active_job.default_queue_name`

Can be used to change the default queue name. By default this is `"default"`.

```ruby
config.active_job.default_queue_name = :medium_priority
```

[`config.active_job.default_queue_name`]: #config-active-job-default-queue-name

#### `config.active_job.queue_name_prefix`

Allows you to set an optional, non-blank, queue name prefix for all jobs. By default it is blank and not used.

The following configuration would queue the given job on the `production_high_priority` queue when run in production:

```ruby
config.active_job.queue_name_prefix = Rails.env
```

```ruby
class GuestsCleanupJob < ActiveJob::Base
  queue_as :high_priority
  #....
end
```

#### `config.active_job.queue_name_delimiter`

Has a default value of `'_'`. If `queue_name_prefix` is set, then `queue_name_delimiter` joins the prefix and the non-prefixed queue name.

The following configuration would queue the provided job on the `video_server.low_priority` queue:

```ruby
# prefix must be set for delimiter to be used
config.active_job.queue_name_prefix = "video_server"
config.active_job.queue_name_delimiter = "."
```

```ruby
class EncoderJob < ActiveJob::Base
  queue_as :low_priority
  #....
end
```

#### `config.active_job.logger`

Accepts a logger conforming to the interface of Log4r or the default Ruby Logger class, which is then used to log information from Active Job. You can retrieve this logger by calling `logger` on either an Active Job class or an Active Job instance. Set to `nil` to disable logging.

#### `config.active_job.custom_serializers`

Allows to set custom argument serializers. Defaults to `[]`.

#### `config.active_job.enqueue_after_transaction_commit`

Controls whether jobs enqueued inside an Active Record transaction are deferred
until after the transaction commits. When false, jobs are enqueued immediately.
Individual jobs can override the global setting:

```ruby
class NotificationJob < ApplicationJob
  self.enqueue_after_transaction_commit = false
end
```

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 8.2                   | `true`               |

#### `config.active_job.log_arguments`

Controls if the arguments of a job are logged. Defaults to `true`.

#### `config.active_job.verbose_enqueue_logs`

Specifies if source locations of methods that enqueue background jobs should be logged below relevant enqueue log lines. By default, the flag is `true` in development and `false` in all other environments.

#### `config.active_job.retry_jitter`

Controls the amount of "jitter" (random variation) applied to the delay time calculated when retrying failed jobs.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `0.0`                |
| 6.1                   | `0.15`               |

#### `config.active_job.log_query_tags_around_perform`

Determines whether job context for query tags will be automatically updated via
an `around_perform`. The default value is `true`.

### Configuring Action Cable

#### `config.action_cable.url`

Accepts a string for the URL for where you are hosting your Action Cable
server. You would use this option if you are running Action Cable servers that
are separated from your main application.

#### `config.action_cable.mount_path`

Accepts a string for where to mount Action Cable, as part of the main server
process. Defaults to `/cable`. You can set this as nil to not mount Action
Cable as part of your normal Rails server.

You can find more detailed configuration options in the
[Action Cable Overview](action_cable_overview.html#configuration).

#### `config.action_cable.precompile_assets`

Determines whether the Action Cable assets should be added to the asset pipeline precompilation. It
has no effect if Sprockets is not used. The default value is `true`.

#### `config.action_cable.allow_same_origin_as_host`

Determines whether an origin matching the cable server itself will be permitted.
The default value is `true`.

Set to false to disable automatic access for same-origin requests, and strictly allow
only the configured origins.

#### `config.action_cable.allowed_request_origins`

Determines the request origins which will be accepted by the cable server.
The default value is `/https?:\/\/localhost:\d+/` in the `development` environment.

### Configuring Active Storage

`config.active_storage` provides the following configuration options:

#### `config.active_storage.variant_processor`

Accepts a symbol `:mini_magick`, `:vips`, or `:disabled` specifying whether or not variant
processing and blob analysis will be performed with MiniMagick or ruby-vips.

It also accepts a class. The class must implement the interface defined by
`ActiveStorage::Transformers::Transformer`. Active Storage then uses it for variant processing:

```ruby
config.active_storage.variant_processor = CustomTransformer
```

Note that the built-in image analyzers accept a blob only when `variant_processor` is `:vips` or
`:mini_magick`, so setting this configuration to a custom class requires adding a custom analyzer to
[`config.active_storage.analyzers`](#config-active-storage-analyzers) as well.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:mini_magick`       |
| 7.0                   | `:vips`              |

#### `config.active_storage.analyzers`

Accepts an array of classes indicating the analyzers available for Active Storage blobs.
By default, this is defined as:

```ruby
config.active_storage.analyzers = [
  ActiveStorage::Analyzer::ImageAnalyzer::Vips,
  ActiveStorage::Analyzer::ImageAnalyzer::ImageMagick,
  ActiveStorage::Analyzer::VideoAnalyzer,
  ActiveStorage::Analyzer::AudioAnalyzer
]
```

The image analyzers can extract width and height of an image blob; the video analyzer can extract width, height, duration, angle, aspect ratio, and presence/absence of video/audio channels of a video blob; the audio analyzer can extract duration and bit rate of an audio blob.

If you want to disable analyzers, you can set this to an empty array:

```ruby
config.active_storage.analyzers = []
```

#### `config.active_storage.analyze`

Controls when attachment analysis (image/video/audio metadata extraction) is performed:

* `:immediately` - Analyze before validation, making metadata available for validations (e.g. image dimensions, video duration)
* `:later` - Analyze after upload from local IO or via background job for direct uploads
* `:lazily` - Skip automatic analysis; analyze on-demand

When set to `:immediately`, you can validate file properties in model validations:

```ruby
class User < ApplicationRecord
  has_one_attached :avatar

  validate :validate_avatar_dimensions, if: -> { avatar.attached? }

  def validate_avatar_dimensions
    if avatar.metadata[:width] < 200 || avatar.metadata[:height] < 200
      errors.add(:avatar, "must be at least 200x200 pixels")
    end
  end
end
```

Attachments with `process: :immediately` variants implicitly use immediate analysis to ensure metadata is available before processing.

NOTE: Direct uploads bypass the server so the file isn't locally available for analysis. In this case, `:immediately` falls back to `:later`, analyzing via background job after upload completes. Metadata isn't available for validation.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `:later`             |
| 8.2                   | `:immediately`       |

#### `config.active_storage.previewers`

Accepts an array of classes indicating the image previewers available in Active Storage blobs.
By default, this is defined as:

```ruby
config.active_storage.previewers = [ActiveStorage::Previewer::PopplerPDFPreviewer, ActiveStorage::Previewer::MuPDFPreviewer, ActiveStorage::Previewer::VideoPreviewer]
```

`PopplerPDFPreviewer` and `MuPDFPreviewer` can generate a thumbnail from the first page of a PDF blob; `VideoPreviewer` from the relevant frame of a video blob.

#### `config.active_storage.paths`

Accepts a hash of options indicating the locations of previewer/analyzer commands. The default is `{}`, meaning the commands will be looked for in the default path. Can include any of these options:

* `:ffprobe` - The location of the ffprobe executable.
* `:mutool` - The location of the mutool executable.
* `:ffmpeg` - The location of the ffmpeg executable.

```ruby
config.active_storage.paths[:ffprobe] = "/usr/local/bin/ffprobe"
```

#### `config.active_storage.variable_content_types`

Accepts an array of strings indicating the content types that Active Storage
can transform through the variant processor.
By default, this is defined as:

```ruby
config.active_storage.variable_content_types = %w(image/png image/gif image/jpeg image/tiff image/bmp image/vnd.adobe.photoshop image/vnd.microsoft.icon image/webp image/avif image/heic image/heif)
```

#### `config.active_storage.web_image_content_types`

Accepts an array of strings regarded as web image content types in which
variants can be processed without being converted to the fallback PNG format.
For example, if you want to use `AVIF` variants in your application you can add
`image/avif` to this array.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is                            |
| --------------------- | ----------------------------------------------- |
| (original)            | `%w(image/png image/jpeg image/gif)`            |
| 7.2                   | `%w(image/png image/jpeg image/gif image/webp)` |

#### `config.active_storage.content_types_to_serve_as_binary`

Accepts an array of strings indicating the content types that Active Storage will always serve as an attachment, rather than inline.
By default, this is defined as:

```ruby
config.active_storage.content_types_to_serve_as_binary = %w(text/html image/svg+xml application/postscript application/x-shockwave-flash text/xml application/xml application/xhtml+xml application/mathml+xml text/cache-manifest)
```

#### `config.active_storage.content_types_allowed_inline`

Accepts an array of strings indicating the content types that Active Storage allows to serve as inline.
By default, this is defined as:

```ruby
config.active_storage.content_types_allowed_inline = %w(image/webp image/avif image/png image/gif image/jpeg image/tiff image/vnd.adobe.photoshop image/vnd.microsoft.icon application/pdf)
```

#### `config.active_storage.queues.analysis`

Accepts a symbol indicating the Active Job queue to use for analysis jobs. When
this option is `nil`, analysis jobs are sent to the default Active Job queue
(see [`config.active_job.default_queue_name`][]).

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| 6.0                   | `:active_storage_analysis` |
| 6.1                   | `nil`                |

#### `config.active_storage.queues.mirror`

Accepts a symbol indicating the Active Job queue to use for direct upload
mirroring jobs. When this option is `nil`, mirroring jobs are sent to the
default Active Job queue (see [`config.active_job.default_queue_name`][]). The
default is `nil`.

#### `config.active_storage.queues.preview_image`

Accepts a symbol indicating the Active Job queue to use for preprocessing
previews of images. When this option is `nil`, jobs are sent to the default
Active Job queue (see [`config.active_job.default_queue_name`][]). The default
is `nil`.

#### `config.active_storage.queues.purge`

Accepts a symbol indicating the Active Job queue to use for purge jobs. When
this option is `nil`, purge jobs are sent to the default Active Job queue (see
[`config.active_job.default_queue_name`][]).

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| 6.0                   | `:active_storage_purge` |
| 6.1                   | `nil`                |

#### `config.active_storage.queues.transform`

Accepts a symbol indicating the Active Job queue to use for preprocessing
variants. When this option is `nil`, jobs are sent to the default Active Job
queue (see [`config.active_job.default_queue_name`][]). The default is `nil`.

#### `config.active_storage.logger`

Can be used to set the logger used by Active Storage. Accepts a logger conforming to the interface of Log4r or the default Ruby Logger class.

```ruby
config.active_storage.logger = ActiveSupport::Logger.new(STDOUT)
```

#### `config.active_storage.service_urls_expire_in`

Determines the default expiry of URLs generated by:

* [`ActiveStorage::Blob#url`][]
* [`ActiveStorage::Blob#service_url_for_direct_upload`][]
* [`ActiveStorage::Preview#url`][]
* [`ActiveStorage::Variant#url`][]

The default is 5 minutes.

[`ActiveStorage::Blob#url`]: https://api.rubyonrails.org/classes/ActiveStorage/Blob.html#method-i-url
[`ActiveStorage::Blob#service_url_for_direct_upload`]: https://api.rubyonrails.org/classes/ActiveStorage/Blob.html#method-i-service_url_for_direct_upload
[`ActiveStorage::Preview#url`]: https://api.rubyonrails.org/classes/ActiveStorage/Preview.html#method-i-url
[`ActiveStorage::Variant#url`]: https://api.rubyonrails.org/classes/ActiveStorage/Variant.html#method-i-url

#### `config.active_storage.urls_expire_in`

Determines the default expiry of URLs in the Rails application generated by Active Storage. The default is nil.

#### `config.active_storage.touch_attachment_records`

Directs ActiveStorage::Attachments to touch its corresponding record when updated. The default is true.

#### `config.active_storage.routes_prefix`

Can be used to set the route prefix for the routes served by Active Storage.
Accepts any value supported by `scope`, such as a string path prefix or a hash of
routing options.

```ruby
config.active_storage.routes_prefix = "/files"
```

For example, to serve the Active Storage routes from a specific subdomain:

```ruby
config.active_storage.routes_prefix = { path: "/files", subdomain: "assets" }
```

The default is `/rails/active_storage`.

#### `config.active_storage.track_variants`

Determines whether variants are recorded in the database.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 6.1                   | `true`               |

#### `config.active_storage.draw_routes`

Can be used to toggle Active Storage route generation. The default is `true`.

#### `config.active_storage.resolve_model_to_route`

Can be used to globally change how Active Storage files are delivered.

Allowed values are:

* `:rails_storage_redirect`: Redirect to signed, short-lived service URLs.
* `:rails_storage_proxy`: Proxy files by downloading them.

The default is `:rails_storage_redirect`.

#### `config.active_storage.video_preview_arguments`

Can be used to alter the way ffmpeg generates video preview images.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `"-y -vframes 1 -f image2"` |
| 7.0                   | `"-vf 'select=eq(n\\,0)+eq(key\\,1)+gt(scene\\,0.015)"`<sup><mark><strong><em>1</em></strong></mark></sup> <br> `+ ",loop=loop=-1:size=2,trim=start_frame=1'"`<sup><mark><strong><em>2</em></strong></mark></sup><br> `+ " -frames:v 1 -f image2"` <br><br> <ol><li>Select the first video frame, plus keyframes, plus frames that meet the scene change threshold.</li> <li>Use the first video frame as a fallback when no other frames meet the criteria by looping the first (one or) two selected frames, then dropping the first looped frame.</li></ol> |

#### `config.active_storage.video_preview_input_arguments`

Arguments passed to ffmpeg before `-i` when generating video preview images.
ffmpeg's flags are position dependent, so arguments that apply to the input,
such as `-codec_whitelist` and `-protocol_whitelist`, belong here.

The default value is `""`.

See [Media Processing of File Uploads](security.html#media-processing-of-file-uploads)
in the Security Guide.

#### `config.active_storage.ffprobe_arguments`

Arguments passed to ffprobe before the file path when analyzing videos and
audio. Applies to both `ActiveStorage::Analyzer::VideoAnalyzer` and
`ActiveStorage::Analyzer::AudioAnalyzer`. Arguments that make ffprobe reject a
file will fail that file's analysis.

The default value is `""`.

See [Media Processing of File Uploads](security.html#media-processing-of-file-uploads)
in the Security Guide.

#### `config.active_storage.multiple_file_field_include_hidden`

In Rails 7.1 and beyond, Active Storage `has_many_attached` relationships will
default to _replacing_ the current collection instead of _appending_ to it. Thus
to support submitting an _empty_ collection, when `multiple_file_field_include_hidden`
is `true`, the [`file_field`](https://api.rubyonrails.org/classes/ActionView/Helpers/FormBuilder.html#method-i-file_field)
helper will render an auxiliary hidden field, similar to the auxiliary field
rendered by the [`checkbox`](https://api.rubyonrails.org/classes/ActionView/Helpers/FormBuilder.html#method-i-checkbox)
helper.

The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `false`              |
| 7.0                   | `true`               |

#### `config.active_storage.precompile_assets`

Determines whether the Active Storage assets should be added to the asset pipeline precompilation. It
has no effect if Sprockets is not used. The default value is `true`.

#### `config.active_storage.streaming_max_ranges`

Defines how many ranges a byte range request may contain.

`ActiveStorage::Streaming` allows requesting partial resources using HTTP Range Requests,
but that feature can be abused for denial of service attacks.

By default only a single range of byte is allowed, which allows for retries and the vast majority
of use cases. If you need multiple byte range support, you can increase that setting.

| Starting with version | The default value is |
| --------------------- | -------------------- |
| (original)            | `1`                  |

### Configuring Action Text

#### `config.action_text.attachment_tag_name`

Accepts a string for the HTML tag used to wrap attachments. Defaults to `"action-text-attachment"`.

#### `config.action_text.sanitizer_vendor`

Configures the HTML sanitizer used by Action Text by setting `ActionText::ContentHelper.sanitizer` to an instance of the class returned from the vendor's `.safe_list_sanitizer` method. The default value depends on the `config.load_defaults` target version:

| Starting with version | The default value is                 | Which parses markup as |
|-----------------------|--------------------------------------|------------------------|
| (original)            | `Rails::HTML4::Sanitizer`            | HTML4                  |
| 7.1                   | `Rails::HTML5::Sanitizer` (see NOTE) | HTML5                  |

NOTE: `Rails::HTML5::Sanitizer` is not supported on JRuby, so on JRuby platforms Rails will fall back to `Rails::HTML4::Sanitizer`.

#### `Regexp.timeout`


See Ruby's documentation for [`Regexp.timeout=`](https://docs.ruby-lang.org/en/master/Regexp.html#method-c-timeout-3D).

### Configuring a Database

Just about every Rails application will interact with a database. You can connect to the database by setting an environment variable `ENV['DATABASE_URL']` or by using a configuration file called `config/database.yml`.

Using the `config/database.yml` file you can specify all the information needed to access your database:

```yaml
development:
  adapter: postgresql
  database: blog_development
  pool: 5
```

This will connect to the database named `blog_development` using the `postgresql` adapter. This same information can be stored in a URL and provided via an environment variable like this:

```ruby
ENV["DATABASE_URL"] # => "postgresql://localhost/blog_development?pool=5"
```

The `config/database.yml` file contains sections for three different environments in which Rails can run by default:

* The `development` environment is used on your development/local computer as you interact manually with the application.
* The `test` environment is used when running automated tests.
* The `production` environment is used when you deploy your application for the world to use.

If you wish, you can manually specify a URL inside of your `config/database.yml`

```yaml
development:
  url: postgresql://localhost/blog_development?pool=5
```

The `config/database.yml` file can contain ERB tags `<%= %>`. Anything in the tags will be evaluated as Ruby code. You can use this to pull out data from an environment variable or to perform calculations to generate the needed connection information.

When using a `ENV['DATABASE_URL']` or a `url` key in your `config/database.yml`
file, Rails allows mapping the protocol in the URL to a database adapter that
can be configured from within the application. This allows the adapter to be
configured without modifying the URL set in the deployment environment. See:
[`config.active_record.protocol_adapters`](#config-active-record-protocol-adapters).

TIP: You don't have to update the database configurations manually. If you look at the options of the application generator, you will see that one of the options is named `--database`. This option allows you to choose an adapter from a list of the most used relational databases. You can even run the generator repeatedly: `cd .. && rails new blog --database=mysql`. When you confirm the overwriting of the `config/database.yml` file, your application will be configured for MySQL instead of SQLite. Detailed examples of the common database connections are below.

### Connection Preference

Since there are two ways to configure your connection (using `config/database.yml` or using an environment variable) it is important to understand how they can interact.

If you have an empty `config/database.yml` file but your `ENV['DATABASE_URL']` is present, then Rails will connect to the database via your environment variable:

```bash
$ cat config/database.yml

$ echo $DATABASE_URL
postgresql://localhost/my_database
```

If you have a `config/database.yml` but no `ENV['DATABASE_URL']` then this file will be used to connect to your database:

```bash
$ cat config/database.yml
development:
  adapter: postgresql
  database: my_database
  host: localhost

$ echo $DATABASE_URL
```

If you have both `config/database.yml` and `ENV['DATABASE_URL']` set then Rails will merge the configuration together. To better understand this we must see some examples.

When duplicate connection information is provided the environment variable will take precedence:

```bash
$ cat config/database.yml
development:
  adapter: sqlite3
  database: NOT_my_database
  host: localhost

$ echo $DATABASE_URL
postgresql://localhost/my_database

$ bin/rails runner 'puts ActiveRecord::Base.configurations.inspect'
#<ActiveRecord::DatabaseConfigurations:0x00007fc8eab02880 @configurations=[
  #<ActiveRecord::DatabaseConfigurations::UrlConfig:0x00007fc8eab020b0
    @env_name="development", @spec_name="primary",
    @config={"adapter"=>"postgresql", "database"=>"my_database", "host"=>"localhost"}
    @url="postgresql://localhost/my_database">
  ]
```

Here the adapter, host, and database match the information in `ENV['DATABASE_URL']`.

If non-duplicate information is provided you will get all unique values, environment variable still takes precedence in cases of any conflicts.

```bash
$ cat config/database.yml
development:
  adapter: sqlite3
  pool: 5

$ echo $DATABASE_URL
postgresql://localhost/my_database

$ bin/rails runner 'puts ActiveRecord::Base.configurations.inspect'
#<ActiveRecord::DatabaseConfigurations:0x00007fc8eab02880 @configurations=[
  #<ActiveRecord::DatabaseConfigurations::UrlConfig:0x00007fc8eab020b0
    @env_name="development", @spec_name="primary",
    @config={"adapter"=>"postgresql", "database"=>"my_database", "host"=>"localhost", "pool"=>5}
    @url="postgresql://localhost/my_database">
  ]
```

Since pool is not in the `ENV['DATABASE_URL']` provided connection information its information is merged in. Since `adapter` is duplicate, the `ENV['DATABASE_URL']` connection information wins.

The only way to explicitly not use the connection information in `ENV['DATABASE_URL']` is to specify an explicit URL connection using the `"url"` sub key:

```bash
$ cat config/database.yml
development:
  url: sqlite3:NOT_my_database

$ echo $DATABASE_URL
postgresql://localhost/my_database

$ bin/rails runner 'puts ActiveRecord::Base.configurations.inspect'
#<ActiveRecord::DatabaseConfigurations:0x00007fc8eab02880 @configurations=[
  #<ActiveRecord::DatabaseConfigurations::UrlConfig:0x00007fc8eab020b0
    @env_name="development", @spec_name="primary",
    @config={"adapter"=>"sqlite3", "database"=>"NOT_my_database"}
    @url="sqlite3:NOT_my_database">
  ]
```

Here the connection information in `ENV['DATABASE_URL']` is ignored, note the different adapter and database name.

Since it is possible to embed ERB in your `config/database.yml` it is best practice to explicitly show you are using the `ENV['DATABASE_URL']` to connect to your database. This is especially useful in production since you should not commit secrets like your database password into your source control (such as Git).

```bash
$ cat config/database.yml
production:
  url: <%= ENV['DATABASE_URL'] %>
```

Now the behavior is clear, that we are only using the connection information in `ENV['DATABASE_URL']`.

#### Configuring an SQLite3 Database

Rails comes with built-in support for [SQLite3](https://www.sqlite.org), which is a lightweight serverless database application. While Rails better configures SQLite for production workloads, a busy production environment may overload SQLite. Rails defaults to using an SQLite database when creating a new project because it is a zero configuration database that just works, but you can always change it later.

Here's the section of the default configuration file (`config/database.yml`) with connection information for the development environment:

```yaml
development:
  adapter: sqlite3
  database: storage/development.sqlite3
  pool: 5
  timeout: 5000
```

[SQLite extensions](https://sqlite.org/loadext.html) are supported when using `sqlite3` gem v2.4.0 or later by configuring `extensions`:

``` yaml
development:
  adapter: sqlite3
  extensions:
    - SQLean::UUID                     # module name responding to `.to_path`
    - .sqlpkg/nalgeon/crypto/crypto.so # or a filesystem path
    - <%= AppExtensions.location %>    # or ruby code returning a path
```

Many useful features can be added to SQLite through extensions. You may wish to browse the [SQLite extension hub](https://sqlpkg.org/) or use gems like [`sqlpkg-ruby`](https://github.com/fractaledmind/sqlpkg-ruby) and [`sqlean-ruby`](https://github.com/flavorjones/sqlean-ruby) that simplify extension management.

Other configuration options are described in the [SQLite3Adapter documentation]( https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SQLite3Adapter.html).

#### Configuring a MySQL or MariaDB Database

If you choose to use MySQL or MariaDB instead of the shipped SQLite3 database, your `config/database.yml` will look a little different. Here's the development section:

```yaml
development:
  adapter: mysql2
  encoding: utf8mb4
  database: blog_development
  pool: 5
  username: root
  password:
  socket: /tmp/mysql.sock
```

If your development database has a root user with an empty password, this configuration should work for you. Otherwise, change the username and password in the `development` section as appropriate.

NOTE: If your MySQL version is 5.5 or 5.6 and want to use the `utf8mb4` character set by default, please configure your MySQL server to support the longer key prefix by enabling `innodb_large_prefix` system variable.

Advisory Locks are enabled by default on MySQL and are used to make database migrations concurrent safe. You can disable advisory locks by setting `advisory_locks` to `false`:

```yaml
production:
  adapter: mysql2
  advisory_locks: false
```

#### Configuring a PostgreSQL Database

If you choose to use PostgreSQL, your `config/database.yml` will be customized to use PostgreSQL databases:

```yaml
development:
  adapter: postgresql
  encoding: unicode
  database: blog_development
  pool: 5
```

By default Active Record uses a database feature called advisory locks. You might need to disable this feature if you're using an external connection pooler like PgBouncer:

```yaml
production:
  adapter: postgresql
  advisory_locks: false
```

If enabled, Active Record will create up to `1000` prepared statements per database connection by default. To modify this behavior you can set `statement_limit` to a different value:

```yaml
production:
  adapter: postgresql
  statement_limit: 200
```

The more prepared statements in use: the more memory your database will require. If your PostgreSQL database is hitting memory limits, try lowering `statement_limit` or disabling prepared statements.

#### Configuring an SQLite3 Database for JRuby Platform

If you choose to use SQLite3 and are using JRuby, your `config/database.yml` will look a little different. Here's the development section:

```yaml
development:
  adapter: jdbcsqlite3
  database: storage/development.sqlite3
```

#### Configuring a MySQL or MariaDB Database for JRuby Platform

If you choose to use MySQL or MariaDB and are using JRuby, your `config/database.yml` will look a little different. Here's the development section:

```yaml
development:
  adapter: jdbcmysql
  database: blog_development
  username: root
  password:
```

#### Configuring a PostgreSQL Database for JRuby Platform

If you choose to use PostgreSQL and are using JRuby, your `config/database.yml` will look a little different. Here's the development section:

```yaml
development:
  adapter: jdbcpostgresql
  encoding: unicode
  database: blog_development
  username: blog
  password:
```

Change the username and password in the `development` section as appropriate.

#### Configuring Metadata Storage

By default Rails will store information about your Rails environment and schema
in an internal table named `ar_internal_metadata`.

To turn this off per connection, set `use_metadata_table` in your database
configuration. This is useful when working with a shared database and/or
database user that cannot create tables.

```yaml
development:
  adapter: postgresql
  use_metadata_table: false
```

#### Configuring Retry Behavior

By default, Rails will automatically reconnect to the database server and retry certain queries
if something goes wrong. Only safely retryable (idempotent) queries will be retried. The number
of retries can be specified in your the database configuration via `connection_retries`, or disabled
by setting the value to 0. The default number of retries is 1.

```yaml
development:
  adapter: mysql2
  connection_retries: 3
```

The database config also allows a `retry_deadline` to be configured. If a `retry_deadline` is configured,
an otherwise-retryable query will _not_ be retried if the specified time has elapsed while the query was
first tried. For example, a `retry_deadline` of 5 seconds means that if 5 seconds have passed since a query
was first attempted, we won't retry the query, even if it is idempotent and there are `connection_retries` left.

This value defaults to nil, meaning that all retryable queries are retried regardless of time elapsed.
The value for this config should be specified in seconds.

```yaml
development:
  adapter: mysql2
  retry_deadline: 5 # Stop retrying queries after 5 seconds
```

#### Configuring Query Cache

By default, Rails automatically caches the result sets returned by queries. If Rails encounters the same query
again for that request or job, it will use the cached result set as opposed to running the query against
the database again.

The query cache is stored in memory, and to avoid using too much memory, it automatically evicts the least recently
used queries when reaching a threshold. By default the threshold is `100`, but can be configured in the `database.yml`.

```yaml
development:
  adapter: mysql2
  query_cache: 200
```

To entirely disable query caching, it can be set to `false`

```yaml
development:
  adapter: mysql2
  query_cache: false
```

### Creating Rails Environments

By default Rails ships with three environments: "development", "test", and "production". While these are sufficient for most use cases, there are circumstances when you want more environments.

Imagine you have a server which mirrors the production environment but is only used for testing. Such a server is commonly called a "staging server". To define an environment called "staging" for this server, just create a file called `config/environments/staging.rb`. Since this is a production-like environment, you could copy the contents of `config/environments/production.rb` as a starting point and make the necessary changes from there. It's also possible to require and extend other environment configurations like this:

```ruby
# config/environments/staging.rb
require_relative "production"

Rails.application.configure do
  # Staging overrides
end
```

That environment is no different than the default ones, start a server with `bin/rails server -e staging`, a console with `bin/rails console -e staging`, `Rails.env.staging?` works, etc.

### Deploy to a Subdirectory (relative URL root)

By default Rails expects that your application is running at the root
(e.g. `/`). This section explains how to run your application inside a directory.

Let's assume we want to deploy our application to "/app1". Rails needs to know
this directory to generate the appropriate routes:

```ruby
config.relative_url_root = "/app1"
```

alternatively you can set the `RAILS_RELATIVE_URL_ROOT` environment
variable.

Rails will now prepend "/app1" when generating links.

#### Using Passenger

Passenger makes it easy to run your application in a subdirectory. You can find the relevant configuration in the [Passenger manual](https://www.phusionpassenger.com/library/deploy/apache/deploy/ruby/#deploying-an-app-to-a-sub-uri-or-subdirectory).

#### Using a Reverse Proxy

Deploying your application using a reverse proxy has definite advantages over traditional deploys. They allow you to have more control over your server by layering the components required by your application.

Many modern web servers can be used as a proxy server to balance third-party elements such as caching servers or application servers.

One such application server you can use is [Unicorn](https://bogomips.org/unicorn/) to run behind a reverse proxy.

In this case, you would need to configure the proxy server (NGINX, Apache, etc) to accept connections from your application server (Unicorn). By default Unicorn will listen for TCP connections on port 8080, but you can change the port or configure it to use sockets instead.

You can find more information in the [Unicorn readme](https://bogomips.org/unicorn/README.html) and understand the [philosophy](https://bogomips.org/unicorn/PHILOSOPHY.html) behind it.

Once you've configured the application server, you must proxy requests to it by configuring your web server appropriately. For example your NGINX config may include:

```nginx
upstream application_server {
  server 0.0.0.0:8080;
}

server {
  listen 80;
  server_name localhost;

  root /root/path/to/your_app/public;

  try_files $uri/index.html $uri.html @app;

  location @app {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://application_server;
  }

  # some other configuration
}
```

Be sure to read the [NGINX documentation](https://nginx.org/en/docs/) for the most up-to-date information.

