**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON <https://guides.rubyonrails.org>.**

Configuring Rails Applications
==============================

This guide covers the configuration and initialization features available
in Rails applications.

After reading this guide, you will know:

* How to use Rails' `Configuration` object.
* Where to place configuration code.
* The aspects of Rails that can be configured.
* The intialization events triggered by Rails.
* All the configuration options provided by all of Rails' components.
* How to configure the database connections used by Rails.
* How to create a custom Rails environment.
* How to deploy Rails to a subdirectory.

--------------------------------------------------------------------------------

The `Configuration` Object
--------------------------

Rails' configuration settings are all held in an instance of
[`Rails::Application::Configuration`](https://api.rubyonrails.org/classes/Rails/Application/Configuration.html).
It's instantiated when Rails boots, and can be accessed anywhere in
the application with `Rails.app.config` or `Rails.configuration`.

### Applying Configuration Settings

Rails offers three standard locations to add or modify values on the
configuration object:

1. `config/application.rb`.
2. Environment-specific configuration files.
3. Initializer files.

#### `config/application.rb`

The `config/application.rb` can be thought of as the entry point to your
Rails app. The configuration object is available via the `config`
class-level accessor in the application class:

```ruby#10,11
# config/application.rb

# ...

module MyRailsApp
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 8.2

    config.autoload_lib(ignore: %w[assets tasks])
    config.time_zone = "Central Time (US & Canada)"
  end
end
```

Rails will load a set of baseline defaults for all configuration options
based on a _target version_. This is done using [`config.load_defaults`][]:

```ruby#8
# config/application.rb

# ...

module MyRailsApp
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 8.2

    # ...
  end
end
```

Configuration defaults for an older version of Rails can be loaded using this
method. This allows for an easier upgrade path, as the latest configuration
defaults can be rolled out over a period of time.

The [Default Configuration Values guide](default_configuration_values.html)
documents all changed default settings for each target version of Rails.

[`config.load_defaults`]: https://api.rubyonrails.org/classes/Rails/Application/Configuration.html#method-i-load_defaults

#### Environment-specific configuration files

Rails can be booted in three _environments_ by default.

* `development`: Used on your local machine while developing the application.
* `test`: Used when running automated tests.
* `production`: Used when your application is live on the internet.

Each environment has bespoke configuration files:

* `config/environments/production.rb`
* `config/environments/development.rb`
* `config/environments/test.rb`

These files are all structured in a similar way:

```ruby
Rails.application.configure do
  # Settings specified here will take precedence over those in config/application.rb.

  # config.eager_load = true
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

See [Custom Rails Environments](#custom-rails-environments) for details
on creating additional environments.

#### Initializers

All Ruby files under `config/initializers/` are loaded by Rails when it
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

Custom configuration settings can be added to the configuration object:

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

Environment-specific configuration options can be defined in a YAML
file and loaded in your Rails app
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

```ruby#9
# config/application.rb

# ...

module MyApp
  class Application < Rails::Application
    # ...

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
guaranteed, nor is the fact that they will be run after all gems have
initialized, Rails provides a number of initialization
events. You can hook into these events to run code at specific times.

The events are listed below in the order in which they are run:

* `before_configuration`: Run when your application class in `config/application.rb` is
loaded, before the class body is executed. Engines may use this hook to run code
before the application itself gets configured.

* `before_initialize`: Run directly before Railties and Engines are initialized.

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

Multiple blocks can be defined for each hook, and they'll be invoked
sequentially. For example, you may define a `to_prepare` block
in `config/application.rb`, and another in an initializer file, and
they'll both be run one after the other.

WARNING: Some parts of your application, notably routing, are not yet set
up at the point where the `after_initialize` block is called.

### Load Hooks

Rails is modular, and composed of several components such as Active
Record, Action Dispatch etc. Load hooks allow you to hook into the
loading of these components to run your own initialization code. This
way, your application won't cause conflicts by arbitrarily
triggering a component to load during initialization, or try to invoke
code from a component that hasn't been loaded yet.

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

Some parts of Rails can be configured externally by defining environment
variables. The following environment variables are read by various parts
of Rails:

* `ENV["RAILS_ENV"]` defines the Rails environment (`production`,
  `development`, or `test`).

* `ENV["RAILS_RELATIVE_URL_ROOT"]` is used by the routing code to recognize
  URLs when you [deploy your application to a subdirectory](#deploying-to-a-subdirectory).

* `ENV["RAILS_CACHE_ID"]` and `ENV["RAILS_APP_VERSION"]` are used to generate
  expanded cache keys in Rails' caching code. This allows you to have multiple
  separate caches for the same application.

* `ENV[DATABASE_URL]` specifies the connection URL for the app's primary
  database.

Configuring Rails Components
----------------------------

A variety of aspects within Rails and its contituent components can be
configured using the configuration object. This section lists all the
options available for use, and what they control.

Some components such as Action Mailer may hold their settings under their
own namespace — for example `ActionMailer::Base.options`. Never use this
API directly. These components integrate with the Rails configuration object
to ensure settings are loaded in the correct order. Always use the Rails
configuration object instead: `Rails.app.config.action_mailer.options`.

NOTE: If you need to apply a configuration setting directly to a class, use a
[lazy load hook](https://api.rubyonrails.org/classes/ActiveSupport/LazyLoadHooks.html)
in an initializer to avoid autoloading the class before
initialization has completed.

Each version of Rails loads of number of defaults for the settings
listen below. This can be seen in your `application.rb`:

```ruby
module MyRailsApp
  class Application < Rails::Application
    # Load default configuration for Rails 8.2
    config.load_defaults 8.2

    # ...
  end
end
```

This design enables you to load the default settings for an
older version of Rails than the one you're running.
This makes Rails upgrades easier as you can incrementally make the app
changes required for compatibility with the latest defaults without
being stuck on any particular Rails version.

The complete list of default values for all Rails versions can be found
in the [Default Configuration Values](default_configuration_values.html)
guide.

**The defaults for all options listed in this guide use the value
loaded when the `config.load_defaults` option is set to the current version of
Rails.**

### General Configuration Options

The following methods are used to configure a `Rails::Railtie` object,
such as a subclass of `Rails::Engine` or `Rails::Application`.

#### `config.action_on_early_load_hook`

Controls what happens when a load hook is triggered before the Rails
application is initialized. It's set to `:log` by default. You can
alternatively set it to `:raise`, which will raise a `LoadError`
instead of logging the violation.

#### `config.add_autoload_paths_to_load_path`

Sets whether the Rails autoload paths are added to Ruby's `$LOAD_PATH`.
The default value is `false`.

Files in the autoload paths are required
by Rails' autoloader (powered by [Zeitwerk](https://github.com/fxn/zeitwerk))
using absolute paths, and hence don't need to be defined in Ruby's `$LOAD_PATH`.

Excluding these paths from the `$LOAD_PATH` reduces the work Ruby has to
do when resolving `require` calls with relative paths, and improves
Bootsnap's performance as it has to index fewer files.

The `lib` folder always added to `$LOAD_PATH`.

#### `config.after_initialize`

Takes a block which will be run _after_ Rails has finished initializing
the application. That includes the initialization of the framework
itself, engines, and all the application's initializers in
`config/initializers`. It's a useful place to configure values
set up by other initializers:

```ruby
config.after_initialize do
  ActionView::Base.sanitized_allowed_tags.delete "div"
end
```

This block _will_ be run for Rake tasks.

#### `config.after_routes_loaded`

Takes a block which will be run after Rails has
finished loading the application's routes. This block will also
be run whenever routes are reloaded.

```ruby
config.after_routes_loaded do
  # Code that does something with Rails.application.routes
end
```

#### `config.allow_concurrency`

A boolean controlling whether requests should be handled concurrently.
This should only be set to `false` if application code is not
thread safe.

The defaults value is `true`.

#### `config.asset_host`

Configures the host name for your application's assets. Set this when
serving assets using a CDN, or to work around the browser's concurrency
constraints by using different domain aliases.

This setting is shorthand for
[`config.action_controller.asset_host`](#config-action-controller-asset-host).

#### `config.assume_ssl`

When `true`, the application treats all requests as if they arrived
over an SSL connection.

When proxying requests through a load balancer that terminates SSL,
the forwarded request received by Rails will appear as though it
arrived over HTTP instead of HTTPS. This makes redirects and cookie
security target HTTP instead of HTTPS.

Enabling this option tells Rails to assume that the request arrived
over HTTPS, and that SSL was terminated before the requst hit the app.

The default value is `false`.

#### `config.autoflush_log`

A boolean that toggles whether writing to log files is
buffered (`false`) or written immediately (`true`).

The default value is `true`.

#### `config.autoload_once_paths`

Accepts an array of paths from which Rails will autoload constants
that won't be wiped after every request.

This option is relevant only when
[`config.enable_reloading`](#config-enable-reloading) is enabled, which
it is by default in the `development` environment. When that option is
disabled, all autoloading happens only once.

All elements of this array must also be in
[`config.autoload_paths`](#config-autoload-paths).

The default is an empty array.

#### `config.autoload_paths`

Accepts an array of paths from which Rails will autoload constants.
The default is an empty array.

Setting this option is not recommended, and it is retained for legacy reasons.
See [Autoloading and Reloading Constants](autoloading_and_reloading_constants.html#config-autoload-paths)
for more details.

#### `config.autoload_lib(ignore:)`

Adds the `lib` folder to [`config.autoload_paths`](#config-autoload-paths)
and [`config.eager_load_paths`](#config-eager-load-paths).

You may have sub-directories in the `lib` folder that should not be
autoloaded or eager loaded. Use the `ignore` option to exclude these
using their relative paths:

```ruby
config.autoload_lib(ignore: %w(assets tasks generators))
```

More details can be found in the
[autoloading guide](autoloading_and_reloading_constants.html).

#### `config.autoload_lib_once(ignore:)`

Adds the `lib` folder to
[`config.autoload_once_paths`](#config-autoload-once-paths).

This means that classes and modules in `lib` will be autoloaded when the
Rails app first boots, but they will not be reloaded automatically when
you make code changes.

Use the `ignore` option to exclude sub-folders from being autoloaded exactly
like [`config.autoload_lib`](#config-autoload-lib-ignore).

```ruby
config.autoload_lib_once(ignore: %w(assets tasks generators))
```

#### `config.beginning_of_week`

Sets the default beginning of week for the application.

Accepts a valid day of week as a symbol (`:monday`, `:tuesday`, etc.).

#### `config.cache_classes`

Legacy setting equivalent to `!config.enable_reloading`. Retained
for backwards compatibility. The default value is `nil`.

#### `config.cache_store`

Configures the cache store for Rails caching. Available options are:

* `:memory_store`
* `:file_store`
* `:mem_cache_store`
* `:null_store`
* `:redis_cache_store`
* `:solid_cache_store`

NOTE: `solid_cache_store` requires the [`solid_cache`](https://github.com/rails/solid_cache/)
gem which is installed by default.

You may also define an object that implements the [cache API](https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html).

The default values in each environment are:

| Environment     | Cache Store          |
|-----------------|----------------------|
| `development`   | `:memory_store`      |
| `test`          | `:null_store`        |
| `production`    | `:solid_cache_store` |

See [Cache Stores](caching_with_rails.html#other-cache-stores) for per-store
configuration options.

#### `config.colorize_logging`

Specifies whether or not to use ANSI color codes when logging
information. Defaults to `true`.

#### `config.consider_all_requests_local`

A boolean which controls whether error details will be written to the
HTTP response body for debugging.

When `true`, error details are returned in the HTTP response, and
can be viewed and debugged in the browser.

The default value in the `development` environment is `true`, and for
`production` it is `false`.

For more fine grained control, set this to `false` and
implement `show_detailed_exceptions?` in controllers to specify
which requests should provide debugging information on errors.

#### `config.console`

Sets the class that will be used as the console when you run
`bin/rails console`.

```ruby
# config/initializers/console.rb

# This block is called only when running the Rails console
Rails.app.console do
  require "pry"
  Rails.app.config.console = Pry
end
```

#### `config.content_security_policy_nonce_auto`

See [Adding a Nonce](security.html#adding-a-nonce) in the Security Guide.

#### `config.content_security_policy_nonce_directives`

See [Adding a Nonce](security.html#adding-a-nonce) in the Security Guide.

#### `config.content_security_policy_nonce_generator`

See [Adding a Nonce](security.html#adding-a-nonce) in the Security Guide.

#### `config.content_security_policy_report_only`

See [Reporting Violations](security.html#reporting-violations) in the Security
Guide.

#### `config.credentials.content_path`

The path to the encrypted credentials file.

Defaults to `config/credentials/#{Rails.env}.yml.enc` if it exists,
falling back to `config/credentials.yml.enc`.

NOTE: In order for the `bin/rails credentials` commands to recognize
this value, it must be set in `config/application.rb` or
`config/environments/#{Rails.env}.rb`.

#### `config.credentials.key_path`

The path of the encrypted credentials key file.

Defaults to `config/credentials/#{Rails.env}.key` if it exists, falling back
to `config/master.key`.

NOTE: In order for the `bin/rails credentials` commands to recognize this
value, it must be set in `config/application.rb` or
`config/environments/#{Rails.env}.rb`.

#### `config.debug_exception_response_format`

Sets the format used in responses when errors occur in the
development environment.

The default value is `:default`, and for API-only apps it is `:api`.

#### `config.disable_sandbox`

A boolean that controls whether or not the Rails console can be
started in sandbox mode.

A long running sandbox console sesion may lead a database server to
run out of memory. This setting can be used to prevent such an
occurrence.

The default value is `false`, meaning sandbox mode is enabled.

#### `config.dom_testing_default_html_version`

Sets the HTML parser used by the test helpers in Action View,
Action Dispatch, and [`rails-dom-testing`](https://github.com/rails/rails-dom-testing).

The default value is `:html5`, and `:html4` is a valid alternative.

NOTE: [Nokogiri's](https://nokogiri.org) HTML5 parser is not supported on JRuby,
so on JRuby platforms Rails will fall back to `:html4`.

#### `config.eager_load`

When `true`, all registered [`config.eager_load_namespaces`](#config-eager-load-namespaces)
will be eager loaded. This includes your application, engines, Rails frameworks,
and any other registered namespace.

#### `config.eager_load_namespaces`

Registers namespaces that are eager loaded when [`config.eager_load`](#config-eager-load)
is set to `true`. All namespaces in the list must respond to the `eager_load!`
method.

#### `config.eager_load_paths`

Accepts an array of paths from which Rails will eager load on boot
if [`config.eager_load`](#config-eager-load) is `true`.

Defaults to every folder in the `app` directory of the application.

#### `config.enable_reloading`

When this option is `true`, application classes and modules are
reloaded in between web requests if they change.

The default value is [`!config.cache_classes`](#config-cache-classes), which
will resolve to `true`.

The stock `config/environments/development.rb` sets it to `true`, and the
stock `config/environments/production.rb` sets it to `false`.

The predicate `config.reloading_enabled?` is also defined.

#### `config.encoding`

Sets up the application-wide encoding.

The defaults is the [`Encoding`](https://docs.ruby-lang.org/en/master/Encoding.html)
object for `UTF-8`.

#### `config.exceptions_app`

Sets the exceptions Rack application invoked by the [`ActionDispatch::ShowExceptions`][]
middleware when an exception happens.

Defaults to [`ActionDispatch::PublicExceptions.new(Rails.public_path)`](https://api.rubyonrails.org/classes/ActionDispatch/PublicExceptions.html).

#### `config.file_watcher`

Registers the class used to detect file updates in the file system when
[`config.reload_classes_only_on_change`](#config-reload-classes-only-on-change)
is `true`.

Rails ships with [`ActiveSupport::FileUpdateChecker`][] (the default), and
[`ActiveSupport::EventedFileUpdateChecker`][]. Custom classes must conform to
the [`ActiveSupport::FileUpdateChecker`][] API.

Using [`ActiveSupport::EventedFileUpdateChecker`][] depends on
the [listen](https://github.com/guard/listen) gem.

On Linux and macOS no additional gems are needed, but some are
required [for \*BSD](https://github.com/guard/listen#on-bsd) and
[for Windows](https://github.com/guard/listen#on-windows).

Note that [some setups are unsupported](https://github.com/guard/listen#issues--limitations).

[`ActiveSupport::FileUpdateChecker`]: https://api.rubyonrails.org/classes/ActiveSupport/FileUpdateChecker.html
[`ActiveSupport::EventedFileUpdateChecker`]: https://api.rubyonrails.org/classes/ActiveSupport/EventedFileUpdateChecker.html

#### `config.filter_parameters`

Registers an array if parameters that shouldn't be revealed in the logs,
such as passwords or credit card numbers. It also filters out sensitive values
of database columns when calling `#inspect` on an Active Record object.

By default, Rails filters out passwords by adding the following filters in
`config/initializers/filter_parameter_logging.rb`.

```ruby
Rails.application.config.filter_parameters += [
  :passw, :email, :secret, :token, :_key, :crypt, :salt, :certificate, :otp, :ssn, :cvv, :cvc
]
```

The filter works by partial matching regular expressions. Matched parameters
will be replaced in the logs with `[FILTERED]`.

#### `config.filter_redirect`

An array of values used to filter redirect urls from application logs.

```ruby
Rails.application.config.filter_redirect += ["s3.amazonaws.com", /private-match/]
```

The redirect filter tests whether a URL includes strings or matches regular
expressions defined in the array. Matched URLs will be replaced in the logs with
`[FILTERED]`

#### `config.force_ssl`

Setting this to `true` enables the
[HSTS HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security) for all HTTP responses which tells the client to communicate with
the host over HTTPS only.

It will also set `https://` as the default protocol when generating URLs.

This functionality is implemented by the [`ActionDispatch::SSL`](rails_on_rack.html#actiondispatch-ssl)
middleware, and can be configured via [`config.ssl_options`](#config-ssl-options).

#### `config.helpers_paths`

Registers an array of additional paths to load view helpers.

#### `config.hosts`

An array of strings, regular expressions, or
[`IPAddr`][] objects used to validate the `Host` HTTP header.

This is used by the [`ActionDispatch::HostAuthorization`][] middleware
to prevent [DNS rebinding attacks](security.html#dns-rebinding-attacks).

In `development`, a default set of hosts is defined in the constant
`ActionDispatch::HostAuthorization::ALLOWED_HOSTS_IN_DEVELOPMENT`. Additional
hosts may be specified in an environment variable: `ENV["RAILS_DEVELOPMENT_HOSTS"]`.

In other environments, this option is set to an empty array meaning no
`Host` header checks will be done.

Host authorization in production can be enabled by defining the hosts
for your application:

```ruby
config.hosts << "product.com"
```

A port number may also be specified:

```ruby
config.hosts << "product.com:3000"
```

Regular expressions, or [`IPAddr`] objects can also be supplied:

```ruby
config.hosts << /.*\.product\.com/
```

```ruby
config.hosts << IPAddr.new("10.0.0.0/8")
```

Regular expressions will be wrapped with the anchors `\A` and `\z`,
meaning it must match the entire hostname. For example, `/product.com/`,
once anchored, will not match `www.product.com`.

All subdomains may be permitted using the below pattern:

```ruby
# Allow requests from the domain itself `product.com` and
# all subdomains like `www.product.com` and `beta1.product.com`.
Rails.application.config.hosts << ".product.com"
```

[`IPAddr`]: https://docs.ruby-lang.org/en/master/IPAddr.html

#### `config.host_authorization`

A hash of options used to configure the [`ActionDispatch::HostAuthorization`][]
middleware which is used to prevent [DNS rebinding attacks](security.html#dns-rebinding-attacks).
The hash may contain the keys `:exclude` and `:response_app`.

The `:exclude` option can define a proc which receives the
[`request`](https://api.rubyonrails.org/classes/ActionDispatch/Request.html)
object, and returns a boolean stating whether the path should be excluded
from host authorization checks:

```ruby
# Exclude requests for the /healthcheck/ path from host checking
Rails.application.config.host_authorization = {
  exclude: ->(request) { request.path.include?("healthcheck") }
}
```

When host authorization fails, a default Rack app is used to
return a `403 Forbidden` response. The Rack app can be customized
using the `:response_app` option:

```ruby
Rails.application.config.host_authorization = {
  response_app: -> env do
    [400, { "Content-Type" => "text/plain" }, ["Bad Request"]]
  end
}
```

[`ActionDispatch::HostAuthorization`]: https://api.rubyonrails.org/classes/ActionDispatch/HostAuthorization.html

#### `config.javascript_path`

Sets the path where your app's JavaScript files are located relative to the
`app` directory. The default value is `javascript`.

The `javascript_path` is excluded from `autoload_paths`.

#### `config.log_file_size`

Defines the maximum size of the Rails log file in bytes.
Defaults to `104_857_600` (100 MiB) in the `development` and `test`
environments, and unlimited in `production`.

Ensure you configure log rotation on your server to prevent logs from
filling up the entire disk.

#### `config.log_formatter`

Defines the formatter of the Rails logger. The default is an
instance of `ActiveSupport::Logger::SimpleFormatter`.

If you register a custom logger using [`config.logger`](#config-logger) you
need to manually pass the formatter to your logger before registering it.
Rails will not automatically assign the value of this setting to a
custom logger.

#### `config.log_level`

Defines the verbosity of the Rails logger. This option defaults to `:debug`
for all environments except production, where it defaults
to `:info`.

The available log levels are:

* `:debug`
* `:info`
* `:warn`
* `:error`
* `:fatal`
* `:unknown`

#### `config.log_tags`

Registers _tags_ that will be prepended to each log entry.

It accepts an array of methods that the
[`request`](https://api.rubyonrails.org/classes/ActionDispatch/Request.html)
object responds to, or a `Proc` that accepts the `request` object, or an
object that responds to `to_s`.

#### `config.logger`

Configures the logger to use to write Rails application logs.
The default is [`ActiveSupport::Logger`][] with support for tags via
[`ActiveSupport::TaggedLogging`][].

The logger assigned using this method will be wrapped by an instance
of [`ActiveSupport::BroadcastLogger`][]. This class contains multiple
_broadcasts_ which are instances of logger objects such as
[`ActiveSupport::Logger`][].

This design allows Rails to log to multiple targets simultaneously, such a
file as well as `STDOUT`.

`Rails.logger` will always return an instance of
[`ActiveSupport::BroadcastLogger`][] even when you assign a custom logger.
Your custom logger will be assigned as a _broadcast_ within
[`ActiveSupport::BroadcastLogger`][].

These are the default log destinations for each environment:

| Environment   | Log destinations                    |
|---------------|-------------------------------------|
| `development` | `STDOUT` and `logs/development.log` |
| `test`        | `logs/test.log`                     |
| `production`  | `STDOUT`                            |

NOTE: When running the Rails console in non-`production` environments,
`STDERR` replaces `STDOUT` as a log destination.

Here's how you can configure a custom logger:

```ruby
class MyLogger < ::Logger
  # Add support for log silencing.
  include ActiveSupport::LoggerSilence
end

# Create the custom logger instance.
mylogger           = MyLogger.new(STDOUT)

# Manually assign the log formatter, as Rails doesn't
# automatically do this for custom loggers.
mylogger.formatter = config.log_formatter

# Add support for tagged logging.
# `ActiveSupport::TaggedLogging` is a module. The `new` method
# clones the supplied logger and its formatter, and `extend`s modules
# required for tagged logging on the cloned objects.
tagged_logger      = ActiveSupport::TaggedLogging.new(mylogger)

# Register the custom logger
config.logger      = tagged_logger
```

[`ActiveSupport::Logger`]: https://api.rubyonrails.org/classes/ActiveSupport/Logger.html
[`ActiveSupport::TaggedLogging`]: https://api.rubyonrails.org/classes/ActiveSupport/TaggedLogging.html
[`ActiveSupport::BroadcastLogger`]: https://api.rubyonrails.org/classes/ActiveSupport/BroadcastLogger.html

#### `config.middleware`

The application's Rack middleware can be configured using this option.

Further details are available in the [Rails on Rack guide](rails_on_rack.html#action-dispatch-middleware-stack).

#### `config.precompile_filter_parameters`

When `true`, Rails will precompile [`config.filter_parameters`](#config-filter-parameters)
using [`ActiveSupport::ParameterFilter.precompile_filters`][].

The default value is `true`.

[`ActiveSupport::ParameterFilter.precompile_filters`]: https://api.rubyonrails.org/classes/ActiveSupport/ParameterFilter.html#method-c-precompile_filters

#### `config.public_file_server.enabled`

Configures whether Rails should serve static files from the public directory.
Defaults to `true`.

When serving static files using a web server or reverse proxy
(such as Nginx or Caddy) which sits in front of your Rails application,
set this value to `false`.

#### `config.railties_order`

An array which defines the order in which railties and engines are loaded. The
default value is `[:all]`.

You can customize it as:

```ruby
config.railties_order = [Blog::Engine, :main_app, :all]
```

#### `config.rake_eager_load`

When `true`, the application is eager loaded when running Rake tasks.

Defaults to `false`.

#### `config.relative_url_root`

Configures the relative root path when [deploying to a subdirectory](
configuring.html#deploying-to-a-subdirectory). The default
is `ENV['RAILS_RELATIVE_URL_ROOT']`.

#### `config.reload_classes_only_on_change`

A boolea flag controlling whether classes are reloaded only when tracked
files change. By default, this value is `true`, and Rails tracks all
files within the autoload paths.

If `config.enable_reloading` is `false`, this option is ignored.

#### `config.require_master_key`

When enabled, the app will not boot if a master key hasn't been made
available through `ENV["RAILS_MASTER_KEY"]` or the `config/master.key` file.

The default value is `false`.

#### `config.revision`

Used to set a value that uniquely identifies the current application
version — for example, a git hash. The value must be a string.

When omitted, Rails first checks `ENV["REVISION"]`, then tries reading a
`REVISION` file in the application root. If both are absent it attempts to
get the current commit from the local git repository. Finally, if no value
is found, the value is set to `nil`.

```ruby
config.revision = ENV["GIT_SHA"]
```

The revision can be accessed using `Rails.app.revision` and used for
deployment tracking or error reporting.

#### `config.sandbox_by_default`

When `true`, the Rails console starts in sandbox mode by default.
The `--no-sandbox` flag must be specified to start the console
without sandbox mode. This helps prevent accidental writes to
production databases.

Defaults to `false`.

#### `config.secret_key_base`

The fallback for specifying the input secret for an application's key generator.
It is recommended to leave this unset, and instead to specify a `secret_key_base`
in `config/credentials.yml.enc`.

See the [`secret_key_base` API documentation](
https://api.rubyonrails.org/classes/Rails/Application.html#method-i-secret_key_base)
for more information and alternative configuration methods.

#### `config.server_timing`

When `true`, the [`ServerTiming` middleware](rails_on_rack.html#actiondispatch-servertiming)
is added to the middleware stack.

The default value is `false`, but the stock `config/environments/development.rb`
file sets it to `true`.

#### `config.session_options`

Returns the additional options passed to
[`config.session_store`](#config-session-store).

Use this method to read the options only. [`config.session_store`](#config-session-store)
should be used to assign the options along with the session store.

```ruby
config.session_store :cookie_store, key: "_your_app_session"
config.session_options # => {key: "_your_app_session"}
```

#### `config.session_store`

Specifies the class used to store the session. Allowed values are:

* `:cache_store`
* `:cookie_store`
* `:mem_cache_store`
* a custom store
* `:disabled`

Additional options can be specified when assigning the session store:

```ruby
config.session_store :cookie_store, key: "_your_app_session"
```

If a custom store is specified as a symbol, it will be resolved to
the `ActionDispatch::Session` namespace:

```ruby
# use ActionDispatch::Session::MyCustomStore as the session store
config.session_store :my_custom_store
```

The default is a cookie store with the application name as the key.

#### `config.silence_healthcheck_path`

Specifies the path of the health-check endpoint that should be silenced in the
logs. [`Rails::Rack::SilenceRequest`][] implements the silencing.

This prevents health-check requests from clogging the production logs.

The default value is `nil`, but the stock `config/environments/production.rb`
sets it to `"/up"`.

[`Rails::Rack::SilenceRequest`]: https://api.rubyonrails.org/classes/Rails/Rack/SilenceRequest.html

#### `config.ssl_options`

Configures options for the [`ActionDispatch::SSL`][] middleware.

The default value is `{ hsts: { subdomains: true } }`.

[`ActionDispatch::SSL`]: https://api.rubyonrails.org/classes/ActionDispatch/SSL.html

#### `config.time_zone`

Sets the default time zone for the application and enables
time zone awareness for Active Record.

#### `config.x`

Used to add custom nested configuration options to the Rails configuration object.

```ruby
config.x.payment_processing.schedule = :daily
Rails.app.config.x.payment_processing.schedule # => :daily
```

See [Custom Configuration](#custom-configuration) for details.

#### `config.yjit`

Enables [YJIT](https://docs.ruby-lang.org/en/master/jit/yjit_md.html) when
running Ruby 3.3 or newer.

When deploying to a memory constrained environment, it may be
advisable to set this to `false`.

```ruby
config.yjit = true              # Enable YJIT with default settings
config.yjit = { stats: true }   # Enable YJIT with custom options
config.yjit = false             # Disable YJIT
```

The default value is `!Rails.env.local?`.

#### `Regexp.timeout`

Rails sets this value to `1.0` by default.

See Ruby's documentation for [
`Regexp.timeout=`](https://docs.ruby-lang.org/en/master/Regexp.html#method-c-timeout-3D).

### Configuring Assets

#### `config.assets.paths`

Registers an array of source paths for the [Asset Pipeline](asset_pipeline.html).

#### `config.assets.prefix`

Defines the URL path prefix where assets are served from.

Defaults to `/assets`.

#### `config.assets.manifest_path`

Sets the path to the [asset pipeline's manifest file](asset_pipeline.html#referencing-assets).

The default is a file named `.manifest.json` in the
[`config.assets.prefix`](#config-assets-prefix) directory within the
public folder.

#### `config.assets.excluded_paths`

Registers an array of paths to exclude from the
[asset pipeline's load paths](asset_pipeline.html#load-paths).

#### `config.assets.compilers`

This option defines _compilers_ to process certain types files in the asset
pipeline. The default value is:

```ruby
[
  ["text/css", Propshaft::Compiler::CssAssetUrls],
  ["text/css", Propshaft::Compiler::SourceMappingUrls],
  ["text/javascript", Propshaft::Compiler::JsAssetUrls],
  ["text/javascript", Propshaft::Compiler::SourceMappingUrls]
]
```

#### `config.assets.sweep_cache`

When set to `true`, the cached load path is cleared before
each request if the containing files have changed. This is used
in the development environment to ensure the map caches are
reset when asset files are changed.

The default value is `Rails.env.development?`.

#### `config.assets.server`

A boolean value that defines whether or not to include the
`Propshaft::Server` Rack middleware in the app's
[middleware stack](rails_on_rack.html#action-dispatch-middleware-stack).

The middleware is used to serve and hot reload assets in
development.

The default setting is `Rails.env.development? || Rails.env.test?`.

#### `config.assets.relative_url_root`

Sets the URL root for asset paths when [deploying to a subdirectory](
configuring.html#deploying-to-a-subdirectory).

The default value is
[`config.relative_url_root`](#config-relative-url-root).

#### `config.assets.output_path`

Sets the output path where the assets are written after processing
through the asset pipeline.

The default value is [`config.assets.prefix`](#config-assets-prefix)
folder located within the `public/` folder.

#### `config.assets.file_watcher`

Sets the file watcher used to monitor changes in the asset files.
Defaults to [`config.file_watcher`](#config-file-watcher).

#### `config.assets.version`

Registers an optional string that is incorporated in the generation
of the hash used to stamp the asset's filename. This can be changed
to force all files to be recompiled.

The default value of `"1.0"` is set in `config/initializers/assets.rb`.

#### `config.assets.logger`

Registers a logger conforming to the interface of `Log4r` or
the default Ruby `Logger` class.

Defaults to [`config.logger`](#config-logger).

Setting `config.assets.logger` to `false` will turn off logs
for served assets.

#### `config.assets.quiet`

Disables logging of assets requests. The default value is `false`,
but the stock `config/environments/development.rb` file sets
it to `true`.

### Configuring Generators

The behavior of Rails generators can be customized using
`config.generators`:

```ruby
config.generators do |g|
  g.orm :active_record
  g.test_framework :test_unit
end
```

The full set of methods that can be used in this block are:

| Method                          | Description                                     |
| ------------------------------- | ----------------------------------------------- |
| `force_plural`                  | Allows pluralized model names. Defaults to `false`. |
| `helper`                        | Defines whether or not to generate helpers. Defaults to `true`. |
| `integration_tool`              | Defines the integration tool used to generate integration tests. Defaults to `:test_unit`. |
| `system_tests`                  | Defines the integration tool used to generate system tests. Defaults to `:test_unit`. |
| `orm`                           | Defines the orm to use. Defaults to `false` which will use Active Record. |
| `resource_controller`           | Sets the generator for controllers when using `bin/rails generate resource`. Defaults to `:controller`. |
| `resource_route`                | Controls whether a resource route definition should be generated. Defaults to `true`. |
| `scaffold_controller`           | Sets the generator for _scaffolded_ controllers when using `bin/rails generate scaffold`. Defaults to `:scaffold_controller`. |
| `test_framework`                | Defines the test framework to use. Defaults to `false` which will use `minitest`. |
| `template_engine`               | Controls templating engine used. Defaults to `:erb`. |
| `apply_rubocop_autocorrect_after_generate!` | Applies RuboCop's autocorrect feature after Rails generators are run. |

### Configuring i18n

All configuration options in this section are delegated to the [`I18n`][]
library. See the [Internationalization guide](i18n.html) for more details.

[`I18n`]: https://github.com/ruby-i18n/i18n

#### `config.i18n.available_locales`

Defines the permitted available locales for the app. Defaults to
all locale keys found in locale files, usually only `:en` on a
new application.

#### `config.i18n.default_locale`

Sets the default locale of an application used for i18n.

The default is `:en`.

#### `config.i18n.enforce_available_locales`

When `true`, an `I18n::InvalidLocale` error is raised when a locale
that isn't declared in the `available_locales` list is encountered.

The default is `true`.

Keeping this option enabled is recommended as a security measure to
prevent malicious users from setting an invalid locale via user input.

#### `config.i18n.load_path`

Sets the path to the files containing localized strings.

Defaults to `config/locales/**/*.{yml,rb}`.

#### `config.i18n.raise_on_missing_translations`

Determines whether an error should be raised when localized text for a
key is missing.

When `true`, views and controllers raise `I18n::MissingTranslationData`.
If set to `:strict`, models will also raise the error.

The default setting is `false`, meaning no error will be raised.

#### `config.i18n.fallbacks`

Sets fallback behavior for missing translations.

Setting this option to `true` will fallback to the default locale:

```ruby
config.i18n.fallbacks = true
```

You can also supply an array of locales to use a fallback options:

```ruby
config.i18n.fallbacks = [:tr, :en]
```

Or, different fallbacks can be set for specific locales.

For example, the below example demonstrates how to use `:tr` as a
fallback for `:az` and  both `:de` and `:en` as fallbacks for `:da`:

```ruby
config.i18n.fallbacks = { az: :tr, da: [:de, :en] }
# or
config.i18n.fallbacks.map = { az: :tr, da: [:de, :en] }
```

### Configuring Active Model

#### `config.active_model.i18n_customize_full_message`

Controls whether the [`Error#full_message`][ActiveModel::Error#full_message]
format can be overridden in an i18n locale file. Defaults to `false`.

When set to `true`, `full_message` will look for a format at the attribute
and model level of the locale files.

The default format is `"%{attribute_name} %{error_message}"`.

The following example demonstrates how to override the format for
all `Person` attributes, and set a specific format for the `age`
attribute.

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

#### `config.active_record.logger`

Registers a logger conforming to the interface of `Log4r` or
the default Ruby `Logger` class, which is then passed on to any new
database connections made.

Retrieve this logger by calling `logger` on either an Active Record model
class or an Active Record model instance.

Defaults to [`config.logger`](#config-logger).

Set it to `nil` to disable logging for Active Record.

#### `config.active_record.primary_key_prefix_type`

Configures the name for the primary key columns in your database.

By default, Rails names the primary key column `id`. Alternatively, one of
the below values may be specified:

* `:table_name`: A `Customer` model will look for `customerid` as the primary
key column.

* `:table_name_with_underscore`: A `Customer` model will look for `customer_id`
as the primary key column.

#### `config.active_record.table_name_prefix`

Sets a global string to prepend to all table names.

For example, setting this to `northwest_` means that a `Customer` model
will be backed by a table named `northwest_customers`.

The default is an empty string.

#### `config.active_record.table_name_suffix`

Sets a global string to append to all table names.

For example, setting this to `_northwest` means that a `Customer` model
will be backed by a table named `customers_northwest`.

The default is an empty string.

#### `config.active_record.schema_migrations_table_name`

Sets the name of the schema migrations table.

The default value is `nil`, which sets it to `"schema_migrations"`.

#### `config.active_record.internal_metadata_table_name`

Sets the name of the internal metadata table.

The default value is `nil`, which sets it to `"ar_internal_metadata"`.

#### `config.active_record.protected_environments`

Registers an array of Rails environments where database Rake tasks
performing destructive operations should be prohibited.

The default value is `nil`, which sets it to `[ "production" ]`.

#### `config.active_record.pluralize_table_names`

Defines the naming convention for the tables that back Active Record models.

The default is `true`, which means that a `Customer` model will be backed by
the `customers` table.

When set to `false`, a `Customer` class will be backed by a table
named `customer`.

WARNING: Some Rails generators and installers (notably `active_storage:install`
and `action_text:install`) create tables with pluralized names regardless of this
setting. If you set `pluralize_table_names` to `false`, you will need to
manually rename those tables after installation to maintain consistency.
These installers use fixed table names in their migrations for
compatibility reasons.

#### `config.active_record.default_timezone`

Sets the default timezone when reading dates and times from the database.
The default is `:utc`.

Alternatively, you can set it to `:local`, which will use `Time.local` instead.

#### `config.active_record.schema_format`

Defines the format for the file representing the database schema. This can be
set to `:ruby` (the default), or `:sql`.

`:ruby` defines the schema using a Rails DSL similar to database migrations. It
is database agnostic.

`:sql` dumps the schema to a set of SQL statements. These may potentially be
database-dependent.

This can be overridden for specific databases by setting `schema_format` in
the database configuration (`config/database.yml`). See the
[database configuration](#database-configuration) section for more details.

#### `config.active_record.error_on_ignored_order`

A boolean that toggles whether an error should be raised if the order
of a query is ignored during a batch query.

The options are `true` (raise error) or `false` (warn). The default
is `false`.

#### `config.active_record.timestamped_migrations`

A boolean controlling whether migrations are serialized with timestamps
(`true`) or with serial integers (`false`).

The default is `true`, which is recommended to prevent conflicts when there
are multiple developers working on the same application.

#### `config.active_record.automatically_invert_plural_associations`

A boolean that defines whether Active Record will automatically look for inverse
relations with a pluralized name. This is used to infer
[bi-directional associations](association_basics.html#bi-directional-associations)

The default value is `false`.

Consider the below example:

```ruby
class Post < ApplicationRecord
  has_many :comments
end

class Comment < ApplicationRecord
  belongs_to :post
end
```

When `automatically_invert_plural_associations` is `false`, Active Record
will not automatically infer `:comments` as the inverse association of
`belongs_to :post`. It would expect the inverse association to be the
singular `:comment`.

In the opposite case, when `automatically_invert_plural_associations` is `true`,
`:comments` will be inferred as the inverse association of `belongs_to :post`.

In most cases, inferring the inverse associations (by setting this option to `true`)
is beneficial as it prevents internal inconsistencies and optimizes SQL queries.
There may, however, be some compatibility issues with legacy code.

The global setting for this option may be overridden in specific models:

```ruby#2
class Comment < ApplicationRecord
  self.automatically_invert_plural_associations = false

  belongs_to :post
end
```

#### `config.active_record.validate_migration_timestamps`

A boolean value controlling the validation of timestamps in database
migration files.

When enabled, an error will be raised if the timestamp prefix for a migration
is more than a day ahead of the current time. The default value is `true`.

This prevents forward-dating of migration files, which can impact migration
generation and other migration commands.

This option requires that
[`config.active_record.timestamped_migrations`](#config-active-record-timestamped-migrations)
is set to `true`.

#### `config.active_record.db_warnings_action`

Controls the action taken when an SQL query produces a warning.

The available options are:

| Value               | Behavior                              |
|---------------------|---------------------------------------|
| `:ignore` (default) | Database warnings will be ignored.    |
| `:log`              | Database warnings will be logged using `ActiveRecord.logger` at the `:warn` level |
| `:raise`            | Database warnings will be raised as `ActiveRecord::SQLWarning`. |
| `:report`           | Database warnings will be reported to subscribers of Rails' error reporter. |

Alternatively, you can supply a custom proc which accepts a `SQLWarning`
error object:

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

Specifies a list of warning codes and messages that will be
ignored regardless of the configured `db_warnings_action`.

The list of warnings to suppress can be defined using Strings or
Regular Expressions:

```ruby
config.active_record.db_warnings_action = :raise
# The following warnings will not be raised
config.active_record.db_warnings_ignore = [
  /Invalid utf8mb4 character string/,
  "An exact warning message",
  "1062", # MySQL Error 1062: Duplicate entry
]
```

The default is `nil`, which means that all warnings will be
reported.

#### `config.active_record.migration_strategy`

Use this option to customize the strategy class used
to execute database modifications in a migration.

The default class (`ActiveRecord::Migration::DefaultStrategy`)
delegates all method calls to the connection adapter.

Custom strategies must inherit from
`ActiveRecord::Migration::ExecutionStrategy`. Alternatively,
`ActiveRecord::Migration::DefaultStrategy` can be subclasses to preserve
the default behavior for methods that aren't implemented:

```ruby
class CustomMigrationStrategy < ActiveRecord::Migration::DefaultStrategy
  def drop_table(*)
    raise "Dropping tables is not supported!"
  end
end

config.active_record.migration_strategy = CustomMigrationStrategy
```

You can also configure migration strategies for specific adapters
by setting the `migration_strategy` class on the adapter itself.

This is useful when customizing migration behavior for a specific
database type.

For example, the below snippet demonstrates a custom migration strategy
for PostgreSQL only:

```ruby
class CustomPostgresStrategy < ActiveRecord::Migration::DefaultStrategy
  def drop_table(*)
    # Custom logic specific to PostgreSQL
  end
end

ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.migration_strategy = CustomPostgresStrategy
```

#### `config.active_record.migration_error`

Specifies the behavior when database migrations are pending.

It can be set to `:page_load`, which will raise an `ActiveRecord::PendingMigrationError`
on all requests. Alternatively, it can be left unset, which means that
nothing happens when migrations are pending.

By default, this option is unset, but the stock
`config/environments/development.rb` sets it to `:page_load`.

#### `config.active_record.schema_versions_formatter`

Used to customize the formatter class used by the schema dumper to
format or order schema versions. This option is only relevant when your
[`config.active_record.schema_format`](#config-active-record-schema-format)
is `:sql`.

You may wish to customize the order of the versions in the SQL statement
to prevent merge conflicts when a large number of people are working on the
same project.

Create a custom formatter to accomplish this:

```ruby
class CustomSchemaVersionsFormatter
  def initialize(connection)
    @connection = connection
  end

  def format(versions)
    # Special sorting of versions to reduce the likelihood of merge conflicts.
    sorted_versions = versions.sort { |a, b| b.to_s.reverse <=> a.to_s.reverse }

    sql = +"INSERT INTO schema_migrations (version) VALUES\n"
    sql << sorted_versions.map { |v| "(#{@connection.quote(v)})" }.join(",\n")
    sql << ";"
    sql
  end
end

config.active_record.schema_versions_formatter = CustomSchemaVersionsFormatter
```

WARNING: Do not use this class to transform or modify the version
strings in any way. Doing that would create an inconsistency between
the version strings in every database migration file, and the contents
of the schema versions table.

#### `config.active_record.lock_optimistically`

A boolean that toggles whether Active Record uses optimistic locking.

It's set to `true` by default.

#### `config.active_record.cache_timestamp_format`

Defines the format of the timestamp value in the cache
key. It can be set to any key in the `Time::DATE_FORMATS` hash.

The default is `:usec`.

#### `config.active_record.record_timestamps`

A boolean value which controls whether `create` or `update` operations
on a model updates the timestamp columns in the database table.

The default value is `true`.

#### `config.active_record.partial_inserts`

A boolean value that toggles whether or not partial writes (inserts only set
attributes that are different from the default) are used when creating
new records.

The default value is `false`.

#### `config.active_record.partial_updates`

A boolean value controlling whether or not partial writes (updates only set
attributes that are dirty) are used when updating existing records.

When using partial updates, ensure you also optimistic locking
([`config.active_record.lock_optimistically`](#config-active-record-lock-optimistically))
since concurrent updates may write attributes based on a stale read state.

The default value is `true`.

#### `config.active_record.maintain_test_schema`

A boolean value which controls whether Active Record should keep your
test database schema up-to-date with `db/schema.rb` (or `db/structure.sql`)
when running the test suite.

The default is `true`.

#### `config.active_record.dump_schema_after_migration`

A flag controlling whether or not the database schema should be dumped
to a file (`db/schema.rb` or `db/structure.sql`) when database migrations
are run.

The default value is `true`, but the stock `config/environments/production.rb`
file sets it to `false`.

#### `config.active_record.dump_schema_migrations`

A boolean flag controlling whether Ruby schema dumps [include the
migration versions](active_record_migrations.html#recording-migration-versions-in-schema-dumps)
recorded in the `schema_migrations` table.

The default value is `false`.

The value be overridden for specific databases by setting
`:dump_schema_migrations` in the database configuration.

#### `config.active_record.dump_schema_migrations_sort_by`

When
[`config.active_record.dump_schema_migrations`](#config-active-record-dump-schema-migrations-sort-by)
is enabled, migration versions are sorted using their reversed strings
by default. This helps avoid merge conflicts.

This option allows you to customize the sort order. The value is passed to
`sort_by!` as a proc, meaning you can define a symbol representing a method
on the `String` object:

```ruby
# Linear order.
config.active_record.dump_schema_migrations_sort_by = :itself
```

Alternatively, a custom proc that accepts a single string argument
may be used:

```ruby
# Hash-based order.
require "digest/md5"

config.active_record.dump_schema_migrations_sort_by = ->(version) {
  Digest::MD5.hexdigest(version)
}
```

NOTE: The order of the versions does not affect any functionality,
as the `schema_migrations` table acts as a set.

#### `config.active_record.dump_schemas`

Customize the database schemas that will be dumped when running the
`db:schema:dump` task.

The options are:

* `:schema_search_path` (the default): dumps any schemas listed in the
  `schema_search_path` key in the database configuration. If this option
  is omitted, the behavior will be the same as setting `:all`.
* `:all`: dumps all schemas regardless of the `schema_search_path`.
* A string of comma separated schemas.

#### `config.active_record.before_committed_on_all_records`

A boolean value denoting whether `before_committed!` callbacks
are triggered on all enrolled records in a transaction.

The default value is `true`.

#### `config.active_record.belongs_to_required_by_default`

A boolean value indicating whether a record should fail validation if
a `belongs_to` association is absent.

The default value is `true`.

#### `config.active_record.belongs_to_required_validates_foreign_key`

A boolean value controlling the validation behavior when a model with a
`belongs_to` association is saved.

When `true`, Rails will always execute an extra query to check that the
parent record for a `belongs_to` association exists.

The default behavior (`false`) is to only conduct the presence check when
the value of the foreign key changes. The optimizes the SQL queries executed
when updating a record.

#### `config.active_record.marshalling_format_version`

Define the format to use when an Active Record object is serialized with
Marshal.

Currently, the only supported value is `7.1`.

#### `config.active_record.action_on_strict_loading_violation`

Defines the error handling behavior when [`strict_loading`][] is
set on an association.

The default value is `:raise`, which will raise an exception.
Alternatively, you can log the violation by setting
this option to `:log`.

[`strict_loading`]: active_record_querying.html#strict-loading

#### `config.active_record.strict_loading_by_default`

A flag which sets the global default for [`strict_loading`][].

Defaults to `false`.

#### `config.active_record.strict_loading_mode`

Configures the mode in which strict loading is reported.

The default is `:all`. Alternatively, it can be set to `:n_plus_one_only`
which will only report when loading associations that will lead to an
_N + 1 query_.

#### `config.active_record.index_nested_attribute_errors`

When using nested attributes with [`accepts_nested_attributes_for`][] for
a `has_many` relationship, this configuration option controls how error
messages are keyed.

Consider the below models:

```ruby
class Post < ApplicationRecord
  has_many :comments
  accepts_nested_attributes_for :comments
end

class Comment < ApplicationRecord
  validates :body, presence: true
end
```

When this option is `true`, the index of the invalid object
will be included in the key, and each instance of the invalid object
will have its own key-value entry:

```ruby
post = Post.new(comments_attributes: [{}, {}])
post.validate

post.errors.messages
# => {"comments[0].body": ["can't be blank"], "comments[1].body": ["can't be blank"]}
```

The default is `false`, which excludes the index, and consolidates all instances
of that specific error under a single key:

```ruby
post = Post.new(comments_attributes: [{}, {}])
post.validate

post.errors.messages
# => {"comments.body": ["can't be blank"]}
```

[`accepts_nested_attributes_for`]: https://api.rubyonrails.org/classes/ActiveRecord/NestedAttributes/ClassMethods.html#method-i-accepts_nested_attributes_for

#### `config.active_record.use_schema_cache_dump`

When enabled, schema cache information is available in the `db/schema_cache.yml`
file (generated by `bin/rails db:schema:cache:dump`). This eliminates the need
for a database query to fetch this information.

The default value is `true`.

#### `config.active_record.cache_versioning`

A boolean that configures whether or not to include the `cache_version` when
generating a model's `cache_key`.

The default value is `true`, which excludes the `cache_version` from
the `cache_key`.

```ruby
# config.active_record.cache_versioning = true
Post.first.cache_key
=> "posts/1"

# config.active_record.cache_versioning = false
Post.first.cache_key
=> "posts/1-20260424135308973764"
```

#### `config.active_record.collection_cache_versioning`

Configures whether the `cache_version` is included when calculating
an Active Record relation's `cache_key`.

The default value is `true`, which excludes the `cache_version` from
the `cache_key`.

```ruby
# config.active_record.collection_cache_versioning = true
Post.first.comments.cache_key
=> "comments/query-d4bff9f79484a29ce9e746f5c1eb918c"

# config.active_record.collection_cache_versioning = false
Post.first.comments.cache_key
=> "comments/query-d4bff9f79484a29ce9e746f5c1eb918c-3-20260331154438654584"
```

#### `config.active_record.has_many_inversing`

When enabled (the default), the inverse object for a
`belongs_to` association, which is defined with `has_many`,
can be traversed to in memory.

```ruby
class Post < ApplicationRecord
  has_many :comments
end

class Comment < ApplicationRecord
  belongs_to :post, inverse_of: :comments
end

# config.active_record.has_many_inversing = true
comment = Comment.new
post = comment.build_post
comment.post.comments
# => [#<Comment:0x000000011d81c608 id: nil, created_at: nil, updated_at: nil>]

# config.active_record.has_many_inversing = false
comment = Comment.new
post = comment.build_post
comment.post.comments
# => []
```

TIP: The `inverse_of` option above can be omitted and automatically
inferred by enabling
[`config.active_record.automatically_invert_plural_associations`](#config-active-record-automatically-invert-plural-associations).

#### `config.active_record.automatic_scope_inversing`

Defines whether the inverse association for `has_many` associations that have
a scope should be automatically inferred.

The default value is `true`, which means that the `has_many :comments`
relation below will automatically be inferred as the inverse of
`belongs_to :post`.

```ruby
class Post < ApplicationRecord
  has_many :comments, -> { visible }
end

class Comment < ApplicationRecord
  belongs_to :post
end
```

WARNING: The automatic detection still won't work if the inverse
association has a scope. In the above example a scope on the
`post` association will prevent Rails from finding the
inverse for the `comments` association.

#### `config.active_record.destroy_association_async_job`

Configures the name of the job class used to destroy associated records
in the background.

The default value is `ActiveRecord::DestroyAssociationAsyncJob`.

#### `config.active_record.destroy_association_async_batch_size`

When an association with the option `dependent: :destroy_async`, is
destroyed, this option controls the maximum number of records that will
be destroyed in a single job.

A lower batch size will enqueue more background jobs, but each one will
execute faster; whereas a higher batch size will enqueue fewer jobs where
each one takes longer to complete.

The default value is `nil`, meaning all dependent records for an association
will be destroyed in a single background job.

#### `config.active_record.disable_prepared_statements`

A boolean which, when set to `true` disables SQL _prepared statements_.

#### `config.active_record.queues.destroy`

Sets the Active Job queue in which to enqueue jobs to destroy records.

The default value is `nil`, which sends jobs to the
[default Active Job queue](#config-active-job-default-queue-name).

#### `config.active_record.enumerate_columns_in_select_statements`

A boolean flag controlling whether column names will always be included in
`SELECT` statements — avoiding wildcard queries like `SELECT * FROM ...`.

For example, enabling this option may prevent prepared statement cache errors
when adding columns to a PostgreSQL database.

The default value is `false`, meaning wildcard queries will be used
where appropriate.

#### `config.active_record.verify_foreign_keys_for_fixtures`

A boolean value which, when enabled, validates all foreign key constraints
after fixtures are loaded in tests. Supported by PostgreSQL and SQLite only.

The default value is `true`.

#### `config.active_record.raise_on_assign_to_attr_readonly`

A boolean value which controls whether an exception
is raised when an
[`attr_readonly`](https://api.rubyonrails.org/classes/ActiveRecord/ReadonlyAttributes/ClassMethods.html#method-i-attr_readonly)
attribute is assigned.

The default value is `true`.

#### `config.active_record.run_commit_callbacks_on_first_saved_instances_in_transaction`

When multiple Active Record model instances change the same record
within a transaction, Rails runs `after_commit` or `after_rollback`
callbacks for only one of them. This option specifies how Rails chooses
which instance receives the callbacks.

When `true`, transactional callbacks are run on the first instance
to save, even though its instance state may be stale.

When `false` (the default), transactional callbacks are run on the
instances with the freshest instance state. Those instances are chosen
as follows:

* In general, run transactional callbacks on the last instance to
  save a given record within the transaction.

* There are two exceptions:
  * If the record is created within the transaction, then updated by another
    instance, `after_create_commit`  callbacks will be run on the second
    instance. This is instead of the `after_update_commit` callbacks  that
    would naively be run based on that instance’s state.
  * If the record is destroyed within the transaction, then `after_destroy_commit`
    callbacks will be fired on the last destroyed instance, even if a stale
    instance subsequently performed an update (which will have affected 0 rows).

#### `config.active_record.default_column_serializer`

The serializer implementation to use if one isn't explicitly defined for a given
column.

The default value is `nil`.

`YAML` has been used historically, but it's not a very efficient format
and can be the source of security vulnerabilities if not carefully employed.

#### `config.active_record.run_after_transaction_callbacks_in_order_defined`

A boolean flag controlling the order in which _after transaction_ callbacks
are fired.

When `true` (the default), `after_commit` callbacks are executed in the order
they are defined in a model. When `false`, they are executed in reverse
order.

All other callbacks are always executed in the order they are
defined in a model (unless you use `prepend: true`).

#### `config.active_record.query_log_tags_enabled`

A boolean toggling adapter-level query comments.

Defaults to `false`, but is set to `true` in the stock
`config/environments/development.rb` file.

When enabled, database prepared statements will be automatically disabled.
If prepared statements are desired in conjunction with `query_log_tags`
you must explicitly enable them:

```ruby
config.active_record.disable_prepared_statements = false
```

NOTE: High cardinality comments can cause degraded performance
as the database may not be able to rely on a query plan cache. When forcing
prepared statements with query log tags, high cardinality values should
be avoided — for example: `:request_id` or `admin_id`. Even basic
`controller#action` tags can cause high cardinality on basic queries
such as a `current_user` lookup since it will happen across many endpoints.

#### `config.active_record.query_log_tags`

Registers an array containig the key-value tags to be inserted in a
SQL comment.

The default value is `[ :application, :controller, :action, :job ]`.

All available tags are:

* `:application`
* `:controller`,
* `:namespaced_controller`
* `:action`
* `:job`
* `:source_location`.

WARNING: Calculating the `:source_location` of a query can be slow, so
consider its impact when using it in a production environment.

#### `config.active_record.query_log_tags_format`

A symbol specifying the formatter to use for query log tags. The default
value is `:sqlcommenter`, alternatively you can also use `:legacy`.

#### `config.active_record.cache_query_log_tags`

A boolean specifying whether to enable caching of query log tags. For
applications that have a large number of queries, caching query log tags
can provide a performance benefit when the context does not change during
the lifetime of the request or job execution.

Defaults to `false`.

#### `config.active_record.query_log_tags_prepend_comment`

A boolean that defines whether to prepend query log tags comment to the query.

By default, comments are appended at the end of the query. Certain databases such
as MySQL may truncate the query text. This is the case for slow query logs and
the results of querying some InnoDB internal tables where the length of the query
is more than 1024 bytes.

Prepend comments by enabling this option to always retain the log tags comments in
queries.

Defaults to `false`.

#### `config.active_record.schema_cache_ignored_tables`

WARNING: Deprecated in favor of
[`config.active_record.schema_ignored_tables`](#config-active-record-schema-ignored-tables),
and will be removed in a future Rails version. It is now an alias for that
option, so setting it also excludes the tables from the schema file.

#### `config.active_record.schema_ignored_tables`

Registers a list of tables to ignore when generating the schema
cache and the schema file. It accepts an array of strings, representing the
table names, or regular expressions.

#### `config.active_record.verbose_query_logs`

When enabled, the source locations of methods that call database queries
will be logged below the relevant queries.

The default value is `true` in development and `false` in all other
environments.

#### `config.active_record.sqlite3_adapter_strict_strings_by_default`

Specifies whether the `SQLite3Adapter` should be used in a _strict strings_ mode.
The use of a strict strings mode disables double-quoted string literals.

SQLite has some quirks around double-quoted string literals.
It first tries to consider double-quoted strings as identifier names, but
if they don't exist it then considers them as string literals. As such, typos
can silently go unnoticed.

For example, it is possible to create an index for a non existing column.
See [SQLite documentation](https://www.sqlite.org/quirks.html#double_quoted_string_literals_are_accepted) for more details.

The default value is `true`.

#### `config.active_record.postgresql_adapter_decode_bytea`

A boolean flag which controls whether the PostgreSQL adapter decodes
bytea columns.

```ruby
ActiveRecord::Base.connection
     .select_value("select '\\x48656c6c6f'::bytea").encoding #=> Encoding::BINARY
```

The default value is `true`.

#### `config.active_record.postgresql_adapter_decode_dates`

A boolean flag which controls whether the PostgreSQL adapter decodes
date columns.

```ruby
ActiveRecord::Base.connection
     .select_value("select '2024-01-01'::date").class #=> Date
```

The default value is `true`.

#### `config.active_record.postgresql_adapter_decode_money`

A boolean flag which controls whether the PostgreSQL adapter decodes
date columns.

```ruby
ActiveRecord::Base.connection
     .select_value("select '12.34'::money").class #=> BigDecimal
```

The default value is `true`.

#### `config.active_record.async_query_executor`

Configures the pooling of asynchronous queries.

The default value is `nil`, which means `load_async` is disabled and
instead directly executes queries in the foreground.

Set the value as `:global_thread_pool` or `:multi_thread_pool` to perform
queries asynchronously.

* `:global_thread_pool` will use a single pool for all databases the
  application connects to. This is the preferred configuration
  for applications with a single database, or applications which
  only ever query one database shard at a time.

* `:multi_thread_pool` will use one pool per database, and each pool size
  can be configured individually in `database.yml` through the
  `max_threads` and `min_threads` properties. This can be useful to
  applications regularly querying multiple databases at a time, and
  that need to more precisely define the max concurrency.

#### `config.active_record.global_executor_concurrency`

Defines how many asynchronous queries can be executed concurrently when used
in conjunction with:

```ruby
config.active_record.async_query_executor = :global_thread_pool
```

The default is `4`.

The size of the database connection pool (set using the `max_connections`
option in `config/database.yml`) must be large enough to accommodate both
the foreground threads (web server or job worker threads) and the background
threads specified using this option.

For each process, Rails will create one global query executor that uses this
many threads to process async queries. Thus, the pool size should be at
least `thread_count + global_executor_concurrency + 1`.

For example, if your web server has a maximum of 3 threads,
and `global_executor_concurrency` is set to 4, then your pool size
should be at least 8.

#### `config.active_record.yaml_column_permitted_classes`

Registers an array of additional permitted classes to `safe_load()` on
`ActiveRecord::Coders::YAMLColumn`.

The default is `[Symbol]`.

#### `config.active_record.use_yaml_unsafe_load`

A boolean which allows applications to opt into using `unsafe_load`
on `ActiveRecord::Coders::YAMLColumn`.

Defaults to `false`.

#### `config.active_record.raise_int_wider_than_64bit`

A boolean value that determines whether to raise an exception when
the PostgreSQL adapter is provided an integer that is wider than a signed
64-bit representation.

Defaults to `true`.

#### `config.active_record.generate_secure_token_on`

Sets the point in an object's lifecycle when the value for `has_secure_token`
declarations is generated.

The default is `:initialize`, or alternatively it can be set to `:create`.

```ruby
class User < ApplicationRecord
  has_secure_token
end

# config.active_record.generate_secure_token_on = :initialize
record = User.new
record.token # => "fwZcXX6SkJBJRogzMdciS7wf"

# config.active_record.generate_secure_token_on = :create
record = User.new
record.token # => nil
record.save!
record.token # => "fwZcXX6SkJBJRogzMdciS7wf"
```

#### `config.active_record.permanent_connection_checkout`

`ActiveRecord::Base.connection` checks out a database connection from the
pool and keeps it leased until the end of the request or job. This behavior
can be undesirable in environments that use many more threads or fibers than
there are available connections.

This configuration can be used to find and eliminate code that
calls `ActiveRecord::Base.connection` and migrate it to
`ActiveRecord::Base.with_connection` instead.

The accepted values are:

| Value                 | Behavior                                        |
| --------------------- | ----------------------------------------------- |
| `:disallowed`         | Raises an error                                 |
| `:deprecated`         | Emits a deprecation warning                     |
| `true`                | Allows usage of `ActiveRecord::Base.connection` |

The default value is `true`.

#### `config.active_record.database_cli`

Sets the CLI tool used to access the database via `bin/rails dbconsole`.

By default, the standard tool for the database will be used
(`psql` for PostgreSQL and `mysql` for MySQL).

To customize this, define a hash mapping the tool to the database system.

```ruby
config.active_record.database_cli = {
  postgresql: "pgcli",
  mysql: %w[ mycli mysql ] # An array can be used to define fallbacks
}
```

#### `config.active_record.use_legacy_signed_id_verifier`

Controls how signed IDs are generated and verified using legacy options.

Accepted options are:

* `:generate_and_verify` (default) - Generate and verify signed IDs using the
  following legacy options:

  ```ruby
  { digest: "SHA256", serializer: JSON, url_safe: true }
  ```

* `:verify` - Generate and verify signed IDs using options from
  [`Rails.application.message_verifiers`][], but fall back to verifying with the same
  options as `:generate_and_verify`.

* false - Generate and verify signed IDs using options from
  [`Rails.application.message_verifiers`][] only.

This setting provides a smooth transition to a unified configuration for
all message verifiers. Having a unified configuration makes it more straightforward
to rotate secrets and upgrade signing algorithms.

WARNING: Setting this to false may cause old signed IDs to become unreadable
if `Rails.application.message_verifiers` is not properly configured.
Use [`MessageVerifiers#rotate`][ActiveSupport::MessageVerifiers#rotate] or
[`MessageVerifiers#prepend`][ActiveSupport::MessageVerifiers#prepend] to
configure `Rails.application.message_verifiers` with the appropriate options,
such as `:digest` and `:url_safe`.

[`Rails.application.message_verifiers`]: https://api.rubyonrails.org/classes/Rails/Application.html#method-i-message_verifiers
[ActiveSupport::MessageVerifiers#rotate]: https://api.rubyonrails.org/classes/ActiveSupport/MessageVerifiers.html#method-i-rotate
[ActiveSupport::MessageVerifiers#prepend]: https://api.rubyonrails.org/classes/ActiveSupport/MessageVerifiers.html#method-i-prepend

#### `ActiveRecord::ConnectionAdapters::Mysql2Adapter.emulate_booleans` and `ActiveRecord::ConnectionAdapters::TrilogyAdapter.emulate_booleans`

A flag controlling whether the Active Record MySQL adapter will consider all
`tinyint(1)` columns as booleans. Defaults to `true`.

#### `ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.create_unlogged_tables`

A boolean setting which defines whether database tables created by PostgreSQL
should be _unlogged_. This can speed up performance but adds a risk of data
loss if the database crashes.

Disabling this option in a production environment is highly recommended.

Defaults to `false` in all environments.

Enable this in the `test` environment using:

```ruby
# config/environments/test.rb

ActiveSupport.on_load(:active_record_postgresqladapter) do
  self.create_unlogged_tables = true
end
```

#### `ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.datetime_type`

Configures the native type used by Active Record's PostgreSQL adapter
when `datetime` is called in a migration or schema.

It accepts a symbol corresponding to a value in
`ActiveRecord::ConnectionAdapters::PostgreSQLAdapter::NATIVE_DATABASE_TYPES`.

The default is `:timestamp`, meaning `t.datetime` in a migration will
create a "timestamp without time zone" column.

If you wish to customize this to use a "timestamp with a time zone",
set `:timestamptz`.

Run `bin/rails db:migrate` to rebuild your `schema.rb` if you change
this option.

#### `ActiveRecord::SchemaDumper.ignore_tables`

Accepts an array of tables to **exclude** in any generated schema file.

WARNING: This configuration is deprecated in favor of
[`config.active_record.schema_ignored_tables`](#config-active-record-schema-ignored-tables),
and will be removed in a future Rails version. It is now an alias for that
option, so setting it also excludes the tables from the schema cache.

#### `ActiveRecord::SchemaDumper.fk_ignore_pattern`

Customizes the regular expression used to decide whether a foreign key's
name should be dumped to `db/schema.rb`.

By default, foreign key names starting with `fk_rails_` are not exported to the
database schema dump.

The default value is `/^fk_rails_[0-9a-f]{10}$/`.

#### `config.active_record.encryption.support_unencrypted_data`

When `true`, unencrypted data can be read normally. When `false`,
it will raise errors.

The default is `false`.

#### `config.active_record.encryption.extend_queries`

A boolean flag which, when enabled, sets that queries referencing deterministically
encrypted attributes will be modified to include additional values if needed.
Those additional values will be the clean version of the value
(when `config.active_record.encryption.support_unencrypted_data` is `true`)
and values encrypted with previous encryption schemes, if any
(as provided with the `previous:` option).

The default is `false`.

#### `config.active_record.encryption.encrypt_fixtures`

A boolean value controlling whether encryptable attributes in fixtures will
be automatically encrypted when loaded.

The default is `false`.

#### `config.active_record.encryption.store_key_references`

A boolean value defining whether a reference to the encryption key
is stored in the headers of the encrypted message. This makes for
faster decryption when multiple keys are in use.

The default is `false`.

#### `config.active_record.encryption.add_to_filter_parameters`

A boolean value which, when enabled, automatically adds encrypted attribute names
to [`config.filter_parameters`](#config-filter-parameters).

The default is `true`.

#### `config.active_record.encryption.excluded_from_filter_parameters`

Registers a list of params that won't be filtered out when
[`config.active_record.encryption.add_to_filter_parameters`](#config-active-record-encryption-add-to-filter-parameters)
is true.

The default is an empty array.

#### `config.active_record.encryption.validate_column_size`

A boolean value denoting whether to add a validation based on the column
size. This is recommended to prevent storing huge values using
highly compressible payloads.

The default is `true`.

#### `config.active_record.encryption.primary_key`

The key or lists of keys used to derive root data encryption keys.
The way they are used depends on the key provider configured.

The recommended technique is to set it in the credentials file
under the key: `active_record_encryption.primary_key`.

#### `config.active_record.encryption.deterministic_key`

The key or list of keys used for deterministic encryption.

The recommended technique is to set it in the credentials file
under the key: `active_record_encryption.deterministic_key`.

#### `config.active_record.encryption.key_derivation_salt`

The salt used when deriving keys.

The recommended technique is to set it in the credentials file
under the key: `active_record_encryption.key_derivation_salt`.

#### `config.active_record.encryption.forced_encoding_for_deterministic_encryption`

Sets the default encoding for attributes encrypted deterministically.
The default is `Encoding::UTF_8`.

Forced encoding can be disabled by setting this option to `nil`.

#### `config.active_record.encryption.hash_digest_class`

Sets the digest algorithm used by Active Record Encryption.

The default value is `OpenSSL::Digest::SHA256`.

#### `config.active_record.encryption.support_sha1_for_non_deterministic_encryption`

A boolean flag which, when enabled, allows the decryption of existing data that
was encrypted using a SHA-1 digest.

The default value is `false` — meaning only the digest configured in
`config.active_record.encryption.hash_digest_class` will be supported.

#### `config.active_record.encryption.compressor`

Sets the compressor used to compress encrypted payloads. The default is `Zlib`.

This option can be set to any class responding to `deflate` and `inflate`.

#### `config.active_record.protocol_adapters`

When using a URL to configure the database connection, this option
provides a mapping from the protocol to the underlying
database adapter.

For example, the environment can specify `DATABASE_URL=mysql://localhost/database`
and Rails will map `mysql` to the `mysql2` adapter.

These mappings may be overridden as:

```ruby
config.active_record.protocol_adapters.mysql = "trilogy"
```

If no mapping is found, the protocol is used as the adapter name.

#### `config.active_record.deprecated_associations_options`

Accepts a hash which controls behavior when a
[deprecated association](association_basics.html#deprecated) is accessed.

The hash must contain the keys `:mode` and/or `:backtrace`:

```ruby
config.active_record.deprecated_associations_options = { mode: :notify, backtrace: true }
```

* In `:warn` mode, accessing the deprecated association is reported by the
  Active Record logger. This is the default mode.

* In `:raise` mode, usage raises an `ActiveRecord::DeprecatedAssociationError`
  with a similar message and a clean backtrace in the exception object.

* In `:notify` mode, a `deprecated_association.active_record` Active Support
  notification is published. Details about its payload can be found in the
  [Active Support Instrumentation guide](active_support_instrumentation.html).

Backtraces are disabled by default. If `:backtrace` is true, warnings include a
clean backtrace in the message, and notifications have a `:backtrace` key in the
payload with an array of clean `Thread::Backtrace::Location` objects. Exceptions
always have a clean stack trace.

Clean backtraces are computed using the Active Record backtrace cleaner.

#### `config.active_record.raise_on_missing_required_finder_order_columns`

Raises an error when order dependent finder methods (for example, `#first` or `#second`)
are called without `order` values on the relation, where the model does not have any
order columns (`implicit_order_column`, `query_constraints`,
or `primary_key`) to fall back on.

The default value is `true`.

#### `config.active_record.shuffle_unordered_selects`

A boolean flag determining whether to shuffle the rows of every `SELECT` statement
without an `ORDER BY` clause generated by Active Record.

Since the order of such a query is not specified, the database is free to
return the rows in any order. The order can change when an index is added,
when the data grows, or based on the query planner's algorithm.

Enabling this option makes the lack of order explicit. This way, errors caused
code and tests that accidentally depend on the order of the rows can be found
immediately.

The order is fully random and drawn again on every execution, so
a query cannot accidentally settle into an order that an assertion
keeps passing against.

There are two caveats to be aware of:

The first is that Active Record has to recognise the query based on the
underlying Arel. A precompiled SQL query will not be modified.
Examples of instances where Active Record receives precompiled
SQL include: manually written fully-formed SQL strings, association loading,
and queries using a precompiled statement from
`ActiveRecord::StatementCache` such as `find` and `find_by`.

The second is that rows are shuffled **after** the database has returned them,
so this option cannot change *which* rows come back. Queries ending in `LIMIT 1`
are unaffected (such as calls to `find`, `find_by`, `take`, `pick`, `exists?`,
`has_one`, and `belongs_to`).

A query with an `ORDER BY` is never shuffled even when that ordering is
not a total order, so ties on a non-unique column stay hidden. The logged
SQL statement will be the SQL that was sent, so replaying it manually will
not reproduce the order received by the application.

The default value is `false`, as this is a development aid intended for the
test or development environments.

### Configuring Action Controller

#### `config.action_controller.asset_host`

Sets the host used to serve assets.

Configure this option when a CDN is used to deliver assets rather than
the application server itself.

WARNING: Only use this option if Action Mailer uses a different host
for assets, otherwise use [`config.asset_host`](#config-asset-host).

#### `config.action_controller.perform_caching`

Configures whether the [caching features of Action Controller](caching_with_rails.html)
should be enabled.

The default value is `true`, but it's set to `false` in the
stock `config/environments/development.rb` file.

#### `config.action_controller.default_static_extension`

Sets the extension used for cached pages. Defaults to `.html`.

#### `config.action_controller.include_all_helpers`

Configures whether all view helpers are available across all templates.

When disabled, helpers are scoped to the views for their associated
controller. For example, methods in a `UsersHelper` would only be available
in views rendered in the `UsersController`.

The default setting is `nil`, which has the same behavior as `true` — meaning
that all helpers are available across all views in the application.

#### `config.action_controller.logger`

Registers a logger conforming to the interface of `Log4r` or
the default Ruby `Logger` class.

Defaults to [`config.logger`](#config-logger).

Set it to `nil` to disable logging from Action Controller.

#### `config.action_controller.request_forgery_protection_token`

Sets the token parameter name for
[`RequestForgeryProtection`](https://api.rubyonrails.org/classes/ActionController/RequestForgeryProtection.html).

The default value is `:authenticity_token`.

#### `config.action_controller.allow_forgery_protection`

A boolean determining whether CSRF protection is enabled.

By default, this is `false` in the test environment and `true` in
all other environments.

#### `config.action_controller.forgery_protection_origin_check`

A boolean that determines whether the HTTP `Origin` header should
be checked against the site's origin as an additional CSRF defense.

The default value is `true`.

#### `config.action_controller.per_form_csrf_tokens`

A boolean that controls whether CSRF tokens are only valid for the
method or action they were generated for.

The default value is `true`.

#### `config.action_controller.forgery_protection_verification_strategy`

Sets the verification strategy for CSRF protection. Available strategies are:

* `:header_only`: Uses the `Sec-Fetch-Site` header sent by modern browsers
  to verify that requests originate from the same site. Requests without a
  valid header are rejected.

* `:header_or_legacy_token`: A hybrid approach that checks the
  `Sec-Fetch-Site` header first. If the header indicates
  `same-origin` or `same-site`, the request is allowed. When the
  header is missing or has the value "none", it falls back to
  checking the authenticity token. This supports older browsers while
  logging when fallback occurs.

The default value is `:header_only`.

#### `config.action_controller.default_protect_from_forgery`

Determines whether forgery protection is automatically enabled
on [`ActionController::Base`][].

The default value is `true`.

[`ActionController::Base`]: https://api.rubyonrails.org/classes/ActionController/Base.html

#### `config.action_controller.default_protect_from_forgery_with`

Configures the default strategy used when calling
[`protect_from_forgery`][] without the `:with` option.

The default value is `:exception`.

[`protect_from_forgery`]: https://api.rubyonrails.org/classes/ActionController/RequestForgeryProtection/ClassMethods.html#method-i-protect_from_forgery

#### `config.action_controller.relative_url_root`

Set this value when deploying your Rails app to [a subdirectory](#deploying-to-a-subdirectory).

The default is [`config.relative_url_root`](#config-relative-url-root).

#### `config.action_controller.permit_all_parameters`

When `true`, all the parameters for mass assignment will be permitted.

The default value is `false`.

#### `config.action_controller.action_on_unpermitted_parameters`

Defines the behavior when the controller receives parameters which
have not been explicitly permitted.

The accepted values are:

* `false`: No action is taken.
* `:log`: Emits an `ActiveSupport::Notifications.instrument` event on
  the ` unpermitted_parameters.action_controller` topic and writes a log
  at the DEBUG level.
* `:raise`: Raises an `ActionController::UnpermittedParameters` exception.

The default value is `nil` — which means this option will be set to
`:log` in `test` and `development` environments, and `false` in any other
environment.

#### `config.action_controller.always_permitted_parameters`

Registers an array of parameters that are permitted by default.

The default value is `['controller', 'action']`.

#### `config.action_controller.enable_fragment_cache_logging`

A boolean flag controlling the verbosity of log lines for fragment cache
reads and writes.

The default value is `false`, resulting in logs similar to:

```
Rendered messages/_message.html.erb in 1.2 ms [cache hit]
Rendered recordings/threads/_thread.html.erb in 1.5 ms [cache miss]
```

Alternatively, setting this to `true` will log more detailed information:

```
Read fragment views/v1/2914079/v1/2914079/recordings/70182313-20160225015037000000/d0bdf2974e1ef6d31685c3b392ad0b74 (0.6ms)
Rendered messages/_message.html.erb in 1.2 ms [cache hit]
Write fragment views/v1/2914079/v1/2914079/recordings/70182313-20160225015037000000/3b4e249ac9d168c617e32e84b99218b5 (1.1ms)
Rendered recordings/threads/_thread.html.erb in 1.5 ms [cache miss]
```

#### `config.action_controller.raise_on_missing_callback_actions`

Raises an `AbstractController::ActionNotFound` when the action
specified in a callback's `:only` or `:except` options is missing in
the controller.

The default value is `true` in `development` and `test` environments,
and `false` in other environments.

#### `config.action_controller.raise_on_open_redirects`

This option prevents an application from unintentionally
redirecting to an external host (also known as an _open redirect_).

When this configuration is set to `true`, an
`ActionController::Redirecting::UnsafeRedirectError` will be raised when a URL
with an external host is passed to [redirect_to][]. If an open redirect should
be allowed, then `allow_other_host: true` needs to be added to the method call.

The default value is `false`.

WARNING: This option is deprecated and will be removed in a future Rails
version. Use
[`config.action_controller.action_on_open_redirect`](#config-action-controller-action-on-open-redirect)
instead.

[redirect_to]: https://api.rubyonrails.org/classes/ActionController/Redirecting.html#method-i-redirect_to

#### `config.action_controller.action_on_open_redirect`

Defines the behavior when the application redirects to an external host
(also known as an _open redirect_). The available values are:

| Value            | Behavior                   |
| ---------------- | -------------------------- |
| `:log`           | Logs a warning             |
| `:notify`        | Publishes an `open_redirect.action_controller` notification event |
| `:raise`         | Raises an `ActionController::Redirecting::UnsafeRedirectError` |

If [`raise_on_open_redirects`](#config-action-controller-raise-on-open-redirects)
is set to `true`, it will take precedence
over this configuration for backward compatibility, effectively forcing `:raise`
behavior.

The default value is `:raise`.

#### `config.action_controller.action_on_path_relative_redirect`

This option defines the behavior when the application redirects
to a relative path (a path without a leading `/`).

Rails inserts the path into the app's host when rendering the
redirect. For example, if the app is served at `example.com`:

```ruby
redirect_to "/home" # Redirects to https://example.com/home
```

Redirecting to a relative path can lead to a vulnerability as it would
redirect to a different host:

```ruby
redirect_to "home" # Redirects to https://example.comhome

redirect_to "@otherdomain.com" # Redirects to https://example.com@otherdomain.com
```

This option helps detect such unsafe redirects. The available values
are:

| Value            | Behavior                   |
| ---------------- | -------------------------- |
| `:log`           | Logs a warning             |
| `:notify`        | Publishes an `unsafe_redirect.action_controller` notification event |
| `:raise`         | Raises an `ActionController::Redirecting::UnsafeRedirectError` |

#### `config.action_controller.log_query_tags_around_actions`

Determines whether the controller context for query tags will be automatically
updated via an `around_filter`.

The default value is `true`.

#### `config.action_controller.wrap_parameters_by_default`

A boolean value controlling whether [parameter wrapping][params_wrapper]
is enabled by default for JSON requests.

The default value is `true`.

NOTE: The default value for this option in Rails versions older than 7.0
was `false`. However, apps contained a stock initializer file which set the
value to `true`.

[params_wrapper]: https://api.rubyonrails.org/classes/ActionController/ParamsWrapper.html

#### `config.action_controller.allowed_redirect_hosts`

Registers an array of allowed hosts for redirects.

`redirect_to` will allow redirects to them without raising an
`UnsafeRedirectError` error.

#### `config.action_controller.escape_json_responses`

A boolean flag which configures the JSON renderer to escape HTML entities
and unicode characters that are invalid in JavaScript.

This default value is `false`.

This option exists mainly for backwards compatibility.
When escaping is required, use the `:escape` option when calling
`render json:` in specific controller actions.

#### `config.action_controller.rescue_from_event_backtrace`

Configures the `event_backtrace` attribute in the payload of
`rescue_from_handled.action_controller` notifications, and
`action_controller.rescue_from_handled` events.

The accepted values are:

* `:array`: Stores the backtrace as an array of strings.
* `nil`: Stores the backtrace as the first string of the backtrace,
  stripping the `Rails.root` from the controller path.

### Configuring Action Dispatch

#### `config.action_dispatch.cookies_serializer`

Specifies the serializer to use for cookies. It accepts the same values as
[`config.active_support.message_serializer`](#config-active-support-message-serializer),
plus `:hybrid` which is an alias for `:json_allow_marshal`.

The default value is `:json`.

#### `config.action_dispatch.debug_exception_log_level`

Sets the log level used by the [`ActionDispatch::DebugExceptions`][]
middleware when logging uncaught exceptions during requests.

The default value is `:error`.

[`ActionDispatch::DebugExceptions`]: https://api.rubyonrails.org/classes/ActionDispatch/DebugExceptions.html

#### `config.action_dispatch.default_headers`

A hash containing HTTP headers that are set by default in each response.

The default is:

```ruby
{
  "X-Frame-Options" => "SAMEORIGIN",
  "X-Content-Type-Options" => "nosniff",
  "X-Permitted-Cross-Domain-Policies" => "none",
  "Referrer-Policy" => "strict-origin-when-cross-origin"
}
```

#### `config.action_dispatch.default_charset`

Specifies the default character set for all renders.

Defaults to `nil`.

#### `config.action_dispatch.tld_length`

Sets the TLD (top-level domain) length for the application.

Defaults to `1`.

#### `config.action_dispatch.domain_extractor`

Sets the object used by Action Dispatch to parse host names into
domain and subdomain components. The object must respond to
`domain_from(host, tld_length)` and
`subdomains_from(host, tld_length)`.

The default is `ActionDispatch::Http::URL::DomainExtractor`, which
provides the standard domain parsing logic.

Alternatively, a custom extractor which implements specialized domain
parsing behavior can be defined:

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

A boolean which determines whether to ignore `Accept` HTTP headers
in a request.

Defaults to `false`.

#### `config.action_dispatch.strict_accept_header`

A boolean controlling whether an `Accept` header containing
`*/*` forces an HTML response.

When enabled, Rails honors more specific types instead — for example,
`Accept: application/json, */*` returns JSON instead of HTML.

The default value is `true`.

#### `config.action_dispatch.x_sendfile_header`

Sets a custom _send file_ header specific to certain web servers. This header may
be used by a web server or reverse proxy that sits in front of the Rails application
to intercept the response and stream a static file directly to the client, bypassing
the Rails process and enhancing performance.

For example, it can be set to `X-Sendfile` for Apache.

The default value is `nil`.

#### `config.action_dispatch.http_auth_salt`

Sets the salt for HTTP Authentication.

The default is `'http authentication'`.

#### `config.action_dispatch.signed_cookie_salt`

Sets the salt value used when signing cookies.

Defaults to `'signed cookie'`.

#### `config.action_dispatch.encrypted_cookie_salt`

Sets the salt for encrypting cookies.

The default is `'encrypted cookie'`.

#### `config.action_dispatch.encrypted_signed_cookie_salt`

Sets the salt value used when generating signed and encrypted cookies.

The default is `'signed encrypted cookie'`.

#### `config.action_dispatch.authenticated_encrypted_cookie_salt`

Sets the salt for authenticated encrypted cookies.

The default is `'authenticated encrypted cookie'`.

#### `config.action_dispatch.encrypted_cookie_cipher`

Configures the cipher to use when encrypting cookies.

The default is `"aes-256-gcm"`. Any valid OpenSSL cipher
(`OpenSSL::Cipher.ciphers`) may be used.

#### `config.action_dispatch.signed_cookie_digest`

Configures the digest algorithm to use for signing cookies.

The default is `"SHA1"`.

#### `config.action_dispatch.cookies_rotations`

Allows rotating secrets, ciphers, and digests for encrypted and
signed cookies.

#### `config.action_dispatch.use_authenticated_cookie_encryption`

A boolean controlling whether signed and encrypted cookies use the
`AES-256-GCM` cipher (`true`) or the older `AES-256-CBC` cipher (`false`).

The default value is `true`.

#### `config.action_dispatch.use_cookies_with_metadata`

Signed and encrypted cookies are generated using the
[`ActiveSupport::MessageVerifier`][] and [`ActiveSupport::MessageEncryptor`][]
classes. Messages generated using these classes may contain additional metadata
such as `purpose` and `expiry` for added security.

This option controls whether this metadata is included when securing cookies.

The default value is `true`.

[`ActiveSupport::MessageVerifier`]: https://api.rubyonrails.org/classes/ActiveSupport/MessageVerifier.html
[`ActiveSupport::MessageEncryptor`]: https://api.rubyonrails.org/classes/ActiveSupport/MessageEncryptor.html

#### `config.action_dispatch.perform_deep_munge`

Configures whether `deep_munge` should be called on the parameters.
See the [Security Guide](security.html#unsafe-query-generation) for more
information.

It defaults to `true`.

#### `config.action_dispatch.rescue_responses`

Configures a mapping between exceptions and HTTP response status codes. This way,
the status code returned can be customized for specific exceptions.

The default configuration (shown below) can be retrieved
using `ActionDispatch::ExceptionWrapper.rescue_responses`:

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
  "ActionController::InvalidAuthenticityToken" => :unprocessable_content,
  "ActionController::InvalidCrossOriginRequest" => :unprocessable_content,
  "ActionDispatch::Http::Parameters::ParseError" => :bad_request,
  "ActionController::BadRequest" => :bad_request,
  "ActionController::ParameterMissing" => :bad_request,
  "ActionController::TooManyRequests" => :too_many_requests,
  "Rack::QueryParser::ParameterTypeError" => :bad_request,
  "Rack::QueryParser::InvalidParameterError" => :bad_request,
  "ActiveRecord::RecordNotFound" => :not_found,
  "ActiveRecord::StaleObjectError" => :conflict,
  "ActiveRecord::RecordInvalid" => :unprocessable_content,
  "ActiveRecord::RecordNotSaved" => :unprocessable_content
}
```

Add a mapping using:

```ruby
# Use #[]= or #merge! so the default values aren't overwritten
config.action_dispatch.rescue_responses["MyAuthenticationError"] = :unauthorized
```

Rails will fallback to a `500 Internal Server Error` if an exception isn't
mapped to a specific response code.

#### `config.action_dispatch.wrapper_exceptions`

This configuration option defines an array containing names of
exception classes which will be _unwrapped_ before being reported
by the [`ActiveSupport::ErrorReporter`][]. The unwrapped exception will
also be used to determine the HTTP response status code.

Consider the following example:

```ruby
def index
  begin
    raise FirstException
  rescue FirstException
    raise SecondException
  end
end
```

NOTE: The above controller action is unlikely to appear in production
as shown, but the underlying concept applies.

The above snippet demonstrates a scenario where one exception is rescued to
raise another exception. The second exception is considered a
_wrapper exception_. Calling `cause` on the `SecondException` object
will return the `FirstException` object.

With `config.action_dispatch.wrapper_exceptions` unchanged, `SecondException`
will be reported by [`ActiveSupport::ErrorReporter`][] and used to
determine the appropriate HTTP response code.

If `SecondException` is added to this configuration option:

```ruby
config.action_dispatch.wrapper_exceptions += [ "SecondException" ]
```

then, `cause` will be called on the `SecondException` object which will
return a `FirstException`. This is the error that will be reported, and its
corresponding HTTP response code rendered.

The default value is shown below, and can be obtained using
`ActionDispatch::ExceptionWrapper.wrapper_exceptions`:

```ruby
[ "ActionView::Template::Error" ]
```

NOTE: When adding exception classes to this array, ensure you insert the
string representation of the class name, not the constant itself.

[`ActiveSupport::ErrorReporter`]: https://api.rubyonrails.org/classes/ActiveSupport/ErrorReporter.html

#### `config.action_dispatch.silent_exceptions`

When the application backtrace for an exception is empty, Rails falls back
to showing the framework-level backtrace.

This option registers an array of exceptions where Rails should not fall back
to the framework-level backtrace. This can be used to silence noisy
backtraces for exceptions raised at the framework or plugin level.

The default value is shown below, and can be retrieved using
`ActionDispatch::ExceptionWrapper.silent_exceptions`:

```ruby
[
  "ActionController::RoutingError",
  "ActionDispatch::Http::MimeNegotiation::InvalidType"
]
```

Insert a custom exception using:

```ruby
config.action_dispatch.silent_exceptions += [ "CustomException" ]
```

NOTE: Ensure you insert the string representation of the class name
into this array, not the constant itself.

#### `config.action_dispatch.rescue_templates`

Configures a mapping between exceptions and the template rendered when
they are raised.

`ActionDispatch::ExceptionWrapper.rescue_templates` retrieves the default
value:

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

Only templates built into Rails can be used in this configuration hash.
The files are located at `actionpack/lib/action_dispatch/middleware/templates/rescues`.
You cannot specify a template located within your application.

When an exception doesn't have an explicitly configured template, Rails falls
back to the `diagnostics` template.

#### `config.action_dispatch.cookies_same_site_protection`

Sets the default value of the [`SameSite` attribute](https://web.dev/articles/samesite-cookies-explained)
when creating cookies. When set to `nil`, the `SameSite` attribute is not
added.

The `SameSite` attribute may be set dynamically using a Proc:

```ruby
config.action_dispatch.cookies_same_site_protection = ->(request) do
  case request.user_agent
  when "TestAgent"
    nil
  when /App/
    :strict
  else
    :lax
  end
end
```

The default value is `:lax`.

#### `config.action_dispatch.ssl_default_redirect_status`

Configures the default HTTP status code used when redirecting non-GET/HEAD
requests from HTTP to HTTPS in the [`ActionDispatch::SSL`][] middleware.

The default value is `308`.

[`ActionDispatch::SSL`]: https://api.rubyonrails.org/classes/ActionDispatch/SSL.html

#### `config.action_dispatch.log_rescued_responses`

Configures whether unhandled exceptions registered in
[`config.action_dispatch.rescue_responses`](#config-action-dispatch-rescue-responses)
should be logged.

The default value is `true`.

#### `config.action_dispatch.show_exceptions`

The [`ActionDispatch::ShowExceptions`][] Rack middleware invokes a
Rack application ([`config.exceptions_app`](#config-exceptions-app)) to
render error pages for raised exceptions.

This option sets the strategy for which exceptions are handled through the
exceptions app.

The value can be set to:

* `:all`
  All exceptions are rescued and handled using the exceptions app.

* `:rescuable`
  Exceptions defined in [`config.action_dispatch.rescue_responses`](#config-action-dispatch-rescue-responses)
  will be handled using the exceptions app. All other exceptions will
  not be handled within Rails.

* `:none`
  No exceptions will be handled within Rails.

The default value is `:all`.

[`ActionDispatch::ShowExceptions`]: https://api.rubyonrails.org/classes/ActionDispatch/ShowExceptions.html

#### `config.action_dispatch.strict_freshness`

When an HTTP request contains both `If-Modified-Since` and `If-None-Match`,
this option configures how cache freshness should be evaluated.

The default value is `true` — which means  that `If-None-Match`, when present,
is preferred over `If-Modified-Since` to determine freshness.

When `false`, both `If-None-Match` and `If-Modified-Since` are considered
equally.

#### `config.action_dispatch.always_write_cookie`

Rails will only write the `Set-Cookie` header to the response if:

* The request is made over SSL.
* The request is _not_ made over SSL, but the cookie is marked as _insecure_.
* The request is made to a [Tor Onion service](https://en.wikipedia.org/wiki/Tor_(network)).

Enabling this option will override the above logic and **always**
write the `Set-Cookie` header to the response.

It is set to `true` in `development`, and `false` in all other environments.

#### `config.action_dispatch.verbose_redirect_logs`

A boolean flag which specifies whether source locations of redirects
should be logged below relevant log lines.

By default, it is `true` in `development` and `false` in all other
environments.

### Configuring Action View

#### `config.action_view.cache_template_loading`

A boolean flag controlling whether templates are cached in memory
after they are loaded for the first time.

When enabled, the first time a template is required, it
is read from disk and then cached for future use. Conversely,
when `false`, the template is loaded from disk for every request.

Defaults to [`!config.enable_reloading`](#config-enable-reloading).

#### `config.action_view.field_error_proc`

This option assigns a proc used to generate custom HTML for displaying
Active Model errors in a Rails form. It is evaluated within the context
of an `ActionView::Base` object.

The proc is yielded two arguments:

* `html_tag`: The complete HTML tag for the field containing the error.
  For example, if a password field contained an error, the `html_tag`
  value might look like:

  ```ruby
  "<input class='input' type='password' name='user[password]' id='user_password'>"
  ```

* `instance`: An instance of the _field_ from the
  [form builder](form_helpers.html#customizing-form-builders) that contains the
  error. It is usually an instance of a `ActionView::Helpers::Tags::Base` subclass.
  For a password field, it will be an instance of
  `ActionView::Helpers::Tags::PasswordField`.

You can render a partial within this proc as:

```ruby
ActionView::Base.field_error_proc = proc do |html_tag, instance|
  render "application/form_errors",
    html_tag: html_tag, instance: instance
end
```

Or alternatively, render the required markup inline:

```ruby
ActionView::Base.field_error_proc = Proc.new { |html_tag, instance|
  unless html_tag =~ /^<label/
    content_tag :span, class: "error" do
      safe_join([
        html_tag,
        tag.span { instance.error_message.to_sentence }
      ])
    end
  else
    html_tag
  end
}
```

#### `config.action_view.default_form_builder`

Configures the default [form builder](form_helpers.html#customizing-form-builders)
for generating Rails forms.

The default value is
[`ActionView::Helpers::FormBuilder`](https://api.rubyonrails.org/classes/ActionView/Helpers/FormBuilder.html).

Specify the class as a string to load it after initialization, which means it
will be hot-reloaded in `development`.

#### `config.action_view.logger`

Registers a logger conforming to the interface of `Log4r` or
the default Ruby `Logger` class.

Defaults to [`config.logger`](#config-logger).

Set this option to `false` to disable logging in Action View.

#### `config.action_view.erb_trim_mode`

Controls whether ERB's trimming syntax should be enabled.

The default value is `'-'`, which trims tail spaces and
newline characters when using `<%= -%>` or `<%= =%>`.

All other values will disable trimming.

#### `config.action_view.erb_implementation`

Sets the ERB implementation to use. The available options are:

* `:erubi`: All templates are compiled using [Erubi](https://github.com/jeremyevans/erubi).

* `:herb`: HTML templates are compiled using [Herb](https://github.com/marcoroth/herb),
  and all other formats fallback to Erubi.

Using Herb, structural problems such as an unclosed tag are reported at
compile time along with template location.

This option also accepts a custom class that provides a custom ERB
implementation:

```ruby
config.action_view.erb_implementation = MyCustomERBImplementation
```

The default is `:herb`.

#### `config.action_view.escape_ignore_list`

Registers an array containing MIME types that should not be escaped
by the ERB handler during rendering.

The default value is `nil`, which means that only `"text/plain"` will
not be escaped.

The list can be customized as:

```ruby
config.action_view.escape_ignore_list = [ "text/csv", "text/plain" ]
```

NOTE: `"text/plain"` is a fallback which will be overwritten when you assign
this option, so ensure you include it in your custom array:

#### `config.action_view.strip_trailing_newlines`

When enabled, trailing newlines will be stripped from rendered output.

Defaults to `false`.

#### `config.action_view.frozen_string_literal`

When enabled, ERB templates will be compiled with the
`# frozen_string_literal: true` magic comment, which freezes all
string literals and saves memory allocations.

The default value is `nil`. Set it to `true` to enable it for all views.

#### `config.action_view.embed_authenticity_token_in_remote_forms`

When using [Rails UJS](https://guides.rubyonrails.org/v6.1/working_with_javascript_in_rails.html#unobtrusive-javascript),
forms may be submitted using JavaScript by specifying `data-remote="true"` on the
HTML `form` element, or `local: false` on the
[`form_with`](https://api.rubyonrails.org/classes/ActionView/Helpers/FormHelper.html#method-i-form_with)
helper.

This option controls whether remote forms contain an `authenticity_token`
for CSRF protection. The default value is `nil`, which will insert
an authenticity token in remote Rails forms.

Set it to `false` to exclude the `authenticity_token`, which might
be useful for fragment caching.

WARNING: Rails UJS is deprecated an excluded from modern Rails versions
(starting from v7.0). This option is retained for backwards compatibility.
Rails UJS's functionality has been replaced with
[Turbo](working_with_javascript_in_rails.html#turbo)

#### `config.action_view.prefix_partial_path_with_controller_namespace`

Determines the lookup strategy for partials used in templates rendered
from namespaced controllers.

For example, consider a controller named `Admin::ArticlesController`
which renders this partial in one of its templates:

```erb
<%# app/controllers/admin/articles/index.html.erb %>

<%= render @article %>
```

The default behavior (`true`), will render the partial
`/admin/articles/_article.erb`.

Setting the value to `false` will render `/articles/_article.erb`,
which is the same behavior as rendering from a non-namespaced
controller such as `ArticlesController`.

#### `config.action_view.automatically_disable_submit_tag`

A boolean which determines whether `submit_tag` should automatically
disable the input when clicked.

The default value is `true`.

#### `config.action_view.debug_missing_translation`

When `true`, missing translations will render an error in place of
the string.

This defaults to `true`.

#### `config.action_view.form_with_generates_remote_forms`

A boolean flag that determines whether `form_with` generates forms
with `data-remote="true"` set on the HTML element.

The default value is `false`.

WARNING: Remote forms are designed for use with
[Rails UJS](https://guides.rubyonrails.org/v6.1/working_with_javascript_in_rails.html#unobtrusive-javascript),
which is deprecated and excluded from modern versions of Rails (starting
with v7.0). This option is retained for backwards compatilibity. Rails
UJS's functionality has been replaced with
[Turbo](working_with_javascript_in_rails.html#turbo).

#### `config.action_view.form_with_generates_ids`

A boolean controlling whether `form_with` generates an HTML `id` attribute
for all input fields.

The default value is `true`.

#### `config.action_view.default_enforce_utf8`

Determines whether forms are generated with a hidden tag that forces
older versions of Internet Explorer to submit forms encoded in `UTF-8`.

The default value is `false`.

#### `config.action_view.image_loading`

Specifies a default value for the `loading` attribute of `<img>` tags
rendered by the [`image_tag`][] helper.

For example, when set to `"lazy"`, `<img>` tags rendered by
`image_tag` will include
[`loading="lazy"`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/loading#lazy).

This value can be overridden per image:

```erb
<%= image_tag("profile.jpg", loading: :eager) %>
```

The default is `nil`.

[`image_tag`]: https://api.rubyonrails.org/classes/ActionView/Helpers/AssetTagHelper.html#method-i-image_tag

#### `config.action_view.image_decoding`

Specifies a default value for the
[`decoding`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/decoding)
attribute of `<img>` tags rendered by the [`image_tag`][] helper.

Defaults to `nil`.

#### `config.action_view.annotate_rendered_view_with_filenames`

When enabled, rendered views will be annotated with comments denoting
the source template files. This can be observed in the browser's web
inspector:

```erb
<!-- BEGIN app/views/layouts/application.html.erb-->
<!DOCTYPE html>
<html>
  <%# ... %>
  <!-- BEGIN app/views/posts/new.html.erb-->

  <%# ... %>

  <!-- END app/views/posts/new.html.erb-->
  <%# ... %>

</html>
<!-- END app/views/layouts/application.html.erb-->
```

The default value is `false`, but the stock `config/environments/development.rb`
file sets it to `true`.

#### `config.action_view.preload_links_header`

Determines whether `javascript_include_tag` and `stylesheet_link_tag`
will generate a `Link` HTTP header to preload assets.

The default value is `true`.

#### `config.action_view.button_to_generates_button_tag`

When `true` (the default), [`button_to`][] will always generate a `<button>`
inside a `<form>`.

When `false`, the rendered output depends on how the content is passed:

```erb
<%= button_to "Content", "/" %>

<%# renders %>

<form class="button_to" method="post" action="/">
  <input type="submit" value="Content">
  <input type="hidden" name="authenticity_token" value="...">
</form>
```

```erb
<%= button_to "/" do %>
  Content
<% end %>

<%# renders %>

<form class="button_to" method="post" action="/">
  <button type="submit">
    Content
  </button>
  <input type="hidden" name="authenticity_token" value="...">
</form>
```

[`button_to`]: https://api.rubyonrails.org/classes/ActionView/Helpers/UrlHelper.html#method-i-button_to

#### `config.action_view.apply_stylesheet_media_default`

A boolean flag which, when `true` adds the
[`media=screen` HTML attribute](https://developer.mozilla.org/en-US/docs/Web/API/HTMLLinkElement/media)
when calling `stylesheet_link_tag`.

The default value is `false`.

#### `config.action_view.prepend_content_exfiltration_prevention`

When enabled, Rails form helpers (including `button_to`) will prepend
the form with browser-safe (but technically invalid) HTML that guarantees
their contents cannot be captured by any preceding unclosed tags.

```html#1
<!-- '"` --><!-- </textarea></xmp> --></option></form>
<form action="/posts" accept-charset="UTF-8" method="post">
  <!-- ... -->
</form>
```

The default value is `false`.

#### `config.action_view.sanitizer_vendor`

Configures the HTML sanitizer used by Action View.

The default value is `Rails::HTML5::Sanitizer`.

NOTE: Rails will fall back to `Rails::HTML4::Sanitizer` when running on
JRuby platforms, as `Rails::HTML5::Sanitizer` is not supported.

#### `config.action_view.remove_hidden_field_autocomplete`

When enabled, all hidden input fields generated by Rails helpers **will not**
include the HTML attribute `autocomplete="off"`.

The default value is `true`.

#### `config.action_view.render_tracker`

Registers the tracker used to compute a template's dependency tree.

When the digestor builds a template's dependency tree (to compute the cache
keys used by `cache` blocks and `stale?` checks), it asks the dependency
tracker which other templates are rendered by a given template.

The default is `:ruby`, which uses the Prism parser to compute a template's
dependencies.

The legacy option is `:regex`, which scans the template source using a
regex to determine its dependencies.

Other templating languages may register custom trackers to compute their
dependencies, which can then be assigned to this option.

```ruby
ActiveSupport.on_load(:action_view) do
  ActionView::Template.register_template_handler :mtl, MyTemplateLanguage::Handler
  ActionView::DependencyTracker.register_tracker :mtl, MyTemplateLanguage::DependencyTracker
end

Rails.app.config.action_view.render_tracker = :mtl
```

### Configuring Action Mailbox

#### `config.action_mailbox.logger`

Registers a logger conforming to the interface of `Log4r` or
the default Ruby `Logger` class.

Defaults to [`config.logger`](#config-logger).

Setting this option to `false` will turn off logs for
Action Mailbox.

#### `config.action_mailbox.incinerate_after`

An [`ActiveSupport::Duration`](https://api.rubyonrails.org/classes/ActiveSupport/Duration.html)
object that determines the time period after which processed
[`ActionMailbox::InboundEmail`][] records will be destroyed.

The default is `30.days`.

```ruby
# Incinerate inbound emails 14 days after processing.
config.action_mailbox.incinerate_after = 14.days
```

[`ActionMailbox::InboundEmail`]: https://api.rubyonrails.org/classes/ActionMailbox/InboundEmail.html

#### `config.action_mailbox.queues.incineration`

Configures the Active Job queue used for incineration jobs.

The default is `nil` — which will send incineration jobs to the
[default Active Job queue](#config-active-job-default-queue-name).

#### `config.action_mailbox.queues.routing`

Configures the Active Job queue for routing jobs.

The default is `nil` — which will send routing jobs to the
[default Active Job queue](#config-active-job-default-queue-name).

#### `config.action_mailbox.storage_service`

Configures the Active Storage service used to upload emails.

The default is `nil` — which will use the
[default Active Storage Service](#config-active-storage-service).

### Configuring Action Mailer

#### `config.action_mailer.asset_host`

Sets the host for the assets. This is useful when a CDN is used to
host assets rather than the application server itself.

The default is [`config.asset_host`](#config-asset-host).
Only use this option when Action Controller requires a different asset host
from Action Mailer.

#### `config.action_mailer.logger`

Registers a logger conforming to the interface of `Log4r` or
the default Ruby `Logger` class.

Defaults to [`config.logger`](#config-logger).

Setting this option to `false` will turn off logs
for Action Mailer.

#### `config.action_mailer.delivery_method`

Configures the delivery method for Action Mailer emails.

The following options are available:

* `:smtp` (default) - Sends email using SMTP. Configure it with
  [`config.action_mailer.smtp_settings`][].
* `:sendmail` - Sends email using [sendmail](https://en.wikipedia.org/wiki/Sendmail).
  Configure it with [`config.action_mailer.sendmail_settings`][].
* `:file` - Saves emails to files. Configure it with
  [`config.action_mailer.file_settings`][].
* `:test` - Stores emails in memory in the `ActionMailer::Base.deliveries` array.

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

Configures the `:smtp` delivery method. It accepts a hash of options,
which may include the following:

* `:address`: The SMTP server's host name or IP address. The default is `"localhost"`.
* `:port`: The port number for the SMTP server. The default is `25`.
* `:domain`: The domain to use for the `HELO` handshake.
* `:user_name`: The username to authenticate with the mail server.
* `:password`: The password to authenticate with the mail server.
* `:authentication`: The authentication type, which must be one
  of: `:plain`, `:login`, or `:cram_md5`.
* `:enable_starttls`: Force `STARTTLS` when connecting to the SMTP server and fail
  if unsupported. Defaults to `false`.
* `:enable_starttls_auto`: Enables `STARTTLS` when the SMTP server supports it.
  It defaults to `true`.
* `:openssl_verify_mode`: The strategy used to verify certificates. This is
  useful if you need to validate a self-signed or a wildcard certificate.
  The option accepts an OpenSSL verify constant (`OpenSSL::SSL::VERIFY_NONE` or `
  OpenSSL::SSL::VERIFY_PEER`).
* `:ssl/:tls`: Enables the SMTP connection to use
  SMTP/TLS (SMTPS: SMTP over direct TLS connection).
* `:open_timeout`: Number of seconds to wait while attempting to open a connection.
* `:read_timeout`: Number of seconds to wait until timing-out a read(2) call.

Additionally, any [configuration option accepted by `Mail::SMTP`](https://github.com/mikel/mail/blob/master/lib/mail/network/delivery_methods/smtp.rb)
may also be specified in this hash.

#### `config.action_mailer.smtp_timeout`

This option sets the values for both `:open_timeout` and `:read_timeout`
in the [`mail`][] gem. The default is `5`.

Prior to v2.8.0, the [`mail`][] gem did not configure any default timeouts
for its SMTP requests.

[`mail`]: https://github.com/mikel/mail

#### `config.action_mailer.sendmail_settings`

Configures the `:sendmail` delivery method.

It accepts a hash of options, which can include:

* `:location` - The location of the sendmail executable. Defaults to `/usr/sbin/sendmail`.
* `:arguments` - The command line arguments. Defaults to `%w[ -i ]`.

#### `config.action_mailer.file_settings`

Configures the `:file` delivery method.

It accepts a hash of options, which can include:

* `:location` - The location where files are saved. Defaults to `"#{Rails.root}/tmp/mails"`.
* `:extension` - The file extension. The default is an empty string.

#### `config.action_mailer.raise_delivery_errors`

A boolean flag which, when enabled, will raise an error when
email delivery cannot be completed.

It defaults to `true`.

#### `config.action_mailer.perform_deliveries`

Specifies whether mail will actually be delivered.

It is `true` by default but can be set to `false` for testing.

#### `config.action_mailer.default_options`

Configures a set of default options for emails generated by Action Mailer.

Use this option to globally set fields such as `from` or `reply_to`.

The default settings are:

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

Registers an array of [observers](action_mailer_basics.html#observing-emails)
which will be notified after a email is delivered.

The elements in the array must be a class, string, or symbol. Strings and
symbols will be camelized and constantized.

```ruby
config.action_mailer.observers = [ "MailObserver" ]
```

[`Mail::Message`]: https://api.rubyonrails.org/classes/Mail/Message.html

#### `config.action_mailer.interceptors`

Registers an array of [interceptors](action_mailer_basics.html#intercepting-emails)
which will be called before mail is delivered. This allows you to make
modifications to the email before it hits the delivery agents.

The elements in the array must be a class, string, or symbol.
Strings and symbols will be camelized and constantized.

```ruby
config.action_mailer.interceptors = [ "MailInterceptor" ]
```

#### `config.action_mailer.preview_interceptors`

Registers an array of interceptors which will be called before
a mail is previewed. The elements in the array must be a class, string,
or symbol. Strings and symbols will be camelized and constantized.

The interceptor classes must implement the `previewing_email(message)`
method, which will receive a [`Mail::Message`][] object.

```ruby
config.action_mailer.preview_interceptors = [ "MyPreviewMailInterceptor" ]
```

#### `config.action_mailer.preview_paths`

Specifies the file paths which will be searched for
[mailer previews](action_mailer_basics.html#previewing-emails).

```ruby
config.action_mailer.preview_paths << "#{Rails.root}/lib/mailer_previews"
```

#### `config.action_mailer.show_previews`

A boolean flag controlling whether mailers can be previewed.

When unset (the default), it will be `true` in the `development`
environment only.

#### `config.action_mailer.perform_caching`

A boolean flag controlling whether mailer templates should perform
fragment caching.

The default is `nil`, which will enable caching. The stock
`config/environments/development.rb` file sets it to `false`.

#### `config.action_mailer.deliver_later_queue_name`

Specifies the default Active Job queue to use for the
[default mail delivery job](#config-action-mailer-delivery-job).

The default value is `nil`, which sends delivery jobs to the
[default Active Job queue](#config-active-job-default-queue-name).

Mailer classes can override this to use a different queue.

Note that this option only applies when using the default delivery job.
Custom job classes will specify their own queue.

Ensure that your Active Job adapter is configured to process
the specified queue, otherwise delivery jobs may be silently ignored.

#### `config.action_mailer.delivery_job`

Sets the delivery job class used to send mail.

The default value is `"ActionMailer::MailDeliveryJob"`.

#### `config.action_mailer.raise_on_missing_callback_actions`

Mirrors [`config.action_controller.raise_on_missing_callback_actions`](#config-action-controller-raise-on-missing-callback-actions),
but applies to mailers.

The default is `false`.

### Configuring Active Support

#### `config.active_support.bare`

A boolean flag that controls whether `active_support/all` is loaded when
booting Rails.

The default is `nil`, which loads `active_support/all`. Set it to `true`
to disable this.

#### `config.active_support.test_order`

Sets the order in which test cases are executed.

The default value is `:random`. Alternatively, it can be set to
`:sorted`.

#### `config.active_support.escape_html_entities_in_json`

A boolean which controls whether HTML entities are escaped during
JSON serialization.

Defaults to `true`.

#### `config.active_support.use_standard_json_time_format`

A boolean which, when enabled, serializes dates to ISO 8601 format
in JSON.

The default is `true`.

#### `config.active_support.time_precision`

Sets the precision of JSON encoded time values. Defaults to `3`.

#### `config.active_support.hash_digest_class`

Registers the digest class used to generate non-sensitive digests, such as
the `ETag` header.

The default value is `OpenSSL::Digest::SHA256`.

#### `config.active_support.key_generator_hash_digest_class`

Configures the digest class to use to derive secrets from the
secret key base, such as for encrypted cookies.

The default value is `OpenSSL::Digest::SHA256`.

#### `config.active_support.use_authenticated_message_encryption`

A boolean flag which controls whether `AES-256-GCM` is used as the default
cipher (`true`) for encrypting messages, instead of the legacy `AES-256-CBC`.

The default value is `true`.

#### `config.active_support.message_serializer`

Specifies the default serializer used by [`ActiveSupport::MessageEncryptor`][]
and [`ActiveSupport::MessageVerifier`][].

The accepted values are shown in the table below. All provided serializers
have a fallback mechanism to simplify migration.

| Serializer | Serialize and deserialize | Fallback deserialize |
| ---------- | ------------------------- | -------------------- |
| `:marshal` | `Marshal` | `ActiveSupport::JSON`, `ActiveSupport::MessagePack` |
| `:json` | `ActiveSupport::JSON` | `ActiveSupport::MessagePack` |
| `:json_allow_marshal` | `ActiveSupport::JSON` | `ActiveSupport::MessagePack`, `Marshal` |
| `:message_pack` | `ActiveSupport::MessagePack` | `ActiveSupport::JSON` |
| `:message_pack_allow_marshal` | `ActiveSupport::MessagePack` | `ActiveSupport::JSON`, `Marshal` |

The default value is `:json_allow_marshal`.

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

You can also define a custom serializer object that responds to `dump` and
`load`:

```ruby
config.active_support.message_serializer = YAML
```

[`ActiveSupport::MessageEncryptor`]: https://api.rubyonrails.org/classes/ActiveSupport/MessageEncryptor.html
[`ActiveSupport::MessageVerifier`]: https://api.rubyonrails.org/classes/ActiveSupport/MessageVerifier.html
[`message_serializer_fallback.active_support`]: active_support_instrumentation.html#message-serializer-fallback-active-support

#### `config.active_support.use_message_serializer_for_metadata`

A boolean flag which, when `true`, enables a performance optimization
in [`ActiveSupport::MessageEncryptor`][] and
[`ActiveSupport::MessageVerifier`][] by serializing message data and metadata
together.

This changes the message format, so messages serialized this
way cannot be read by Rails versions older than v7.1. However, messages that
use the old format can still be read, regardless of whether this optimization is
enabled.

The default value is `true`.

#### `config.active_support.cache_format_version`

Specifies the serialization format to use for the cache.

The accepted values are:

* `7.0`: serializes cache entries more efficiently.
* `7.1`: further improves efficiency, and allows expired and version-mismatched
  cache entries to be detected without deserializing their values. It also
  includes an optimization for bare string values such as view fragments.

All formats are backward and forward compatible, meaning cache entries written
in one format can be read when using another format. This behavior makes it
easy to migrate between formats without invalidating the entire cache.

The default value is `7.1`.

#### `config.active_support.deprecation`

Configures the strategy used by [`Deprecation::Behavior`][] to report deprecation
warnings. This can be a single value, array, or an object that responds to `call`.

The available options are:

| Value           | Behavior       |
| --------------- | -------------- |
| `:raise` | Raises [`ActiveSupport::DeprecationException`][] |
| `:stderr` | Log all deprecation warnings to `$stderr`. |
| `:log` | Log all deprecation warnings to `Rails.logger` |
| `:notify` | Use [`ActiveSupport::Notifications`][] to notify `deprecation.rails`. |
| `:report` | Use [`ActiveSupport::ErrorReporter`][] to report deprecations. |
| `:silence` | Do nothing |

In the stock `config/environments/development.rb` file, this option is set
to `:log`, and in `config/environments/test.rb`, it is `:stderr`.
In `production`, it is omitted in favor of
[`config.active_support.report_deprecations`](#config-active-support-report-deprecations).

When unset, it defaults to `:stderr`.

[`Deprecation::Behavior`]: https://api.rubyonrails.org/classes/ActiveSupport/Deprecation/Behavior.html
[`ActiveSupport::DeprecationException`]: https://api.rubyonrails.org/classes/ActiveSupport/DeprecationException.html
[`ActiveSupport::Notifications`]: https://api.rubyonrails.org/classes/ActiveSupport/Notifications.html
[`ActiveSupport::ErrorReporter`]: https://api.rubyonrails.org/classes/ActiveSupport/ErrorReporter.html

#### `config.active_support.disallowed_deprecation_warnings`

Defines the criteria used to identify deprecation messages which should
be disallowed. This option can be an array containing strings, symbols, or
regular expressions. These are compared against the text of the
generated deprecation warning.

All deprecations can be disallowed by setting this option to `:all`.

Warnings matched by this option will be handled using the strategy set by
[`config.active_support.disallowed_deprecation`](#config-active-support-disallowed-deprecation)

#### `config.active_support.disallowed_deprecation`

Configures how [`Deprecation::Behavior`][] handles
[disallowed deprecation warnings](#config-active-support-disallowed-deprecation-warnings).
It can be set to a single value, array, or an object that responds to `call`.

The available options are:

| Value           | Behavior       |
| --------------- | -------------- |
| `:raise` | Raises [`ActiveSupport::DeprecationException`][] |
| `:stderr` | Log all deprecation warnings to `$stderr`. |
| `:log` | Log all deprecation warnings to `Rails.logger` |
| `:notify` | Use [`ActiveSupport::Notifications`][] to notify `deprecation.rails`. |
| `:report` | Use [`ActiveSupport::ErrorReporter`][] to report deprecations. |
| `:silence` | Do nothing |

When unset, it defaults to `:raise`.

This option is intended for the `development` and `test` environments.
In production, favor
[`config.active_support.report_deprecations`](#config-active-support-report-deprecations).

#### `config.active_support.report_deprecations`

A boolean flag which controls whether Rails should report deprecation warnings.

When `false`, all deprecation warnings including disallowed deprecations from
your application, its gems, and from Rails will be silenced.

However, it may not prevent all deprecation warnings emitted from
[`ActiveSupport::Deprecation`](https://api.rubyonrails.org/classes/ActiveSupport/Deprecation.html).

The default value is `nil`, which means deprecations will be reported. The stock
`config/environments/production.rb` file sets it to `false`.

#### `config.active_support.isolation_level`

Configures the isolation boundary for Rails' internal state. The default
is `:thread`.

When using a fiber-based server or job processor such as
[Falcon](https://socketry.github.io/falcon/), change this option to `:fiber`.

#### `config.active_support.executor_around_test_case`

A boolean which, when enabled, wraps all test cases around
[`Rails.app.executor.wrap`](https://api.rubyonrails.org/classes/ActiveSupport/ExecutionWrapper.html#method-c-wrap).

This makes the behavior of test cases closer to an actual request or job.
Several features usually disabled in tests, such as the Active Record query cache
and asynchronous queries, will be enabled when wrapped in an executor.

The default value is `true`.

#### `Rails.logger.silencer`

Logs below a specific level can be silenced within the scope of a block
using [`Rails.logger.silence`](https://api.rubyonrails.org/classes/ActiveSupport/LoggerSilence.html#method-i-silence). This option is a boolean flag that toggles whether
the silencer is enabled.

The default is `true`.

#### `ActiveSupport::Cache::Store.logger`

Registers a logger conforming to the interface of `Log4r` or
the default Ruby `Logger` class. It is used within cache store
operations.

#### `ActiveSupport.utc_to_local_returns_utc_offset_times`

A boolean value which controls whether
[`ActiveSupport::TimeZone.utc_to_local`][] returns a time with
a UTC offset (`true`) or a UTC time incorporating that offset (`false`).

The default value is `true`.

[`ActiveSupport::TimeZone.utc_to_local`]: https://api.rubyonrails.org/classes/ActiveSupport/TimeZone.html#method-i-utc_to_local

#### `config.active_support.raise_on_invalid_cache_expiration_time`

A boolean flag which, when enabled, raises an `ArgumentError` when
`Rails.cache` [`fetch`][ActiveSupport::Cache::Store#fetch] or
[`write`][ActiveSupport::Cache::Store#write]
is supplied an invalid `expires_at` or `expires_in` time.

When disabled, the exception will be reported as `handled`
and logged instead.

The default value is `true`.

[ActiveSupport::Cache::Store#fetch]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-fetch
[ActiveSupport::Cache::Store#write]: https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-write

#### `ActiveSupport.raise_on_invalid_time_zone_parse`

A boolean flag which controls whether [`ActiveSupport::TimeZone#parse`][]
raises an `ArgumentError` when passed a string that fulfils one of the
below conditions:

* contains no recognizable date information, such as `"foobar"`
* appears to be a date but is out-of-range, such as `"9000"`,
  which would be interpreted as _month 90_.

When `false`, out-of-range strings will still raise an `ArgumentError`, but
strings that contain no recognizable date information will return `nil`.

The default value is `true`.

[`ActiveSupport::TimeZone#parse`]: https://api.rubyonrails.org/classes/ActiveSupport/TimeZone.html#method-i-parse

#### `config.active_support.event_reporter_context_store`

Registers a custom context store for the [Event Reporter](https://api.rubyonrails.org/classes/ActiveSupport/EventReporter.html).
The context store is used to manage metadata that will be attached to every
event emitted by the reporter.

By default, the Event Reporter uses `ActiveSupport::EventContext` which
stores context in fiber-local storage.

A custom context may be used, as long it as it implements the context
store interface as shown below:

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

The default value is `nil`, which means that `ActiveSupport::EventContext`
store is used.

#### `config.active_support.escape_js_separators_in_json`

Specifies whether `LINE SEPARATOR (U+2028)` and `PARAGRAPH SEPARATOR (U+2029)`
characters are escaped when generating JSON.

Historically, these characters were not valid inside JavaScript literal
strings but that changed in ECMAScript 2019.

As such it's no longer a concern in modern browsers: https://caniuse.com/mdn-javascript_builtins_json_json_superset.

The default value is `false`.

### Configuring Active Job

#### `config.active_job.queue_adapter`

Sets the adapter for the queuing backend. The default adapter is `:async`.

The list of built-in adapters can be found in the
[API documentation](https://api.rubyonrails.org/classes/ActiveJob/QueueAdapters.html).

Rails installs the [Solid Queue](https://github.com/rails/solid_queue/) gem
by default, and the stock `config/environments/production.rb` file configures
it as the queue adapter.

#### `config.active_job.default_queue_name`

Sets the default queue name. When unset, it is `"default"`.

```ruby
config.active_job.default_queue_name = :medium_priority
```

#### `config.active_job.queue_name_prefix`

Sets a prefix which will be appended to the queue name within jobs. It is blank
by default.

The following configuration would queue the given job on the
`production_high_priority` queue when run in production:

```ruby
# config/environments/production.rb

config.active_job.queue_name_prefix = Rails.env
```

```ruby
class GuestsCleanupJob < ActiveJob::Base
  queue_as :high_priority
  #....
end
```

#### `config.active_job.queue_name_delimiter`

When [`config.active_job.queue_name_prefix`](#config-active-job-queue-name-prefix)
is set, this option is used to join the prefix with the queue name.

The default value is `"_"`.

The following configuration would queue the job on the
`video_server.low_priority` queue:

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

Registers a logger conforming to the interface of `Log4r` or
the default Ruby `Logger` class.

Defaults to [`config.logger`](#config-logger).

Setting this option to `false` will turn off logs for
Active Job.

#### `config.active_job.custom_serializers`

Registers an array of
[custom argument serializers](active_job_basics.html#add-custom-types-by-defining-serializers).

Defaults to `[]`.

#### `config.active_job.enqueue_after_transaction_commit`

A boolean controlling whether jobs enqueued inside an Active Record
transaction are deferred until after the transaction commits.

When `false`, jobs are enqueued immediately.

Individual jobs can override the global setting:

```ruby
class NotificationJob < ApplicationJob
  self.enqueue_after_transaction_commit = false
end
```

The default value is `true`.

#### `config.active_job.log_arguments`

A boolean flag which, when enabled, logs the arguments passed to a job.

Defaults to `true`.

#### `config.active_job.verbose_enqueue_logs`

A boolean flag that determines whether the source locations
of methods that enqueue background jobs are logged below relevant enqueue
log lines.

The default value is `false`, but the stock `config/environments/development.rb`
sets it to `true`.

#### `config.active_job.retry_jitter`

Sets the amount of _jitter_ (random variation) applied to the delay time
calculated when retrying failed jobs.

The default value is `0.15`.

#### `config.active_job.log_query_tags_around_perform`

A boolean which determines whether the job context for query tags
will be automatically updated via an `around_perform`.

The default value is `true`.

### Configuring Action Cable

#### `config.action_cable.url`

Configures the URL for the Action Cable server, specified as a string.

Use this option when running standalone Action Cable servers.

#### `config.action_cable.mount_path`

Sets the path where Action Cable will be mounted in the main server
process. The default is `"/cable"`.

Seting this to `nil` will not mount Action Cable as part of
your Rails server.

You can find more detailed configuration options in the
[Action Cable Overview](action_cable_overview.html#configuration).

#### `config.action_cable.precompile_assets`

Determines whether the Action Cable assets should be added
to the asset pipeline precompilation.

It has no effect when Sprockets is not used.

The default value is `true`.

#### `config.action_cable.connection_class`

Specifies a custom class to use for each Action Cable connection. This
value must be a proc which returns the class.

The default value is `nil`, which means `ApplicationCable::Connection`
is used.

#### `config.action_cable.worker_pool_size`

Sets the size of Action Cable's thread pool which is used to
process WebSocket messages.

The default value is unset, meaning it falls back to `4`.

#### `config.action_cable.allow_same_origin_as_host`

A boolean which determines whether an origin matching the
cable server itself will be permitted.

The default value is `true`.

Set to `false` to disable automatic access for `same-origin` requests, and
strictly allow only the [configured origins](#config-action-cable-allowed-request-origins).

#### `config.action_cable.allowed_request_origins`

Configures the request origins which will be accepted by the cable server.
The value can be a string, regular expression, or an array containing either
of those types.

The default value in `development` is `/https?:\/\/localhost:\d+/`. It is
unset in all other environments.

#### `config.action_cable.disable_request_forgery_protection`

When set to `true`, request forgery protection is disabled for Action Cable
requests, meaning requests from all origins will be accepted.

The default value is `false`.

#### `config.action_cable.logger`

Registers a logger for Action Cable, conforming to the interface of
`Log4r` or the default Ruby `Logger` class.

Defaults to [`config.logger`](#config-logger).

Set this to `nil` to disable logging for Action Cable.

#### `config.action_cable.log_tags`

Similar to [`config.log_tags`](#config-log-tags), but specifically
for Action Cable.

It is unset by default.

#### `config.action_cable.filter_parameters`

Similar to [`config.filter_parameters`](#config-filter-parameters), but
specifically for Action Cable.

The default value is unset — meaning `config.filter_parameters` will
be used as a fall back.

#### `config.action_cable.health_check_path`

The path on the Action Cable server used for health-check requests. This
is `nil` by default which disables health-checks for Action Cable server.

The health-check endpoint for your main Rails application is unaffected.
This option may be useful when running standalone Action Cable servers.

#### `config.action_cable.health_check_application`

The Rack application used by the Action Cable server to respond
to health-check requests. The default is the `show` action of the
[`Rails::HealthController`][].

[`Rails::HealthController`]: https://api.rubyonrails.org/classes/Rails/HealthController.html

### Configuring Active Storage

#### `config.active_storage.service`

Configures the service used to upload files managed by Active Storage. The
value is a symbol which references a key in `config/storage.yml`.

```ruby
# config/environments/development.rb

config.active_storage.service = :local
```

```yml
# config/storage.yml

test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
```

In the above example, both `test` and `local` services use the `Disk`
service to upload files. This is implemented by
[`ActiveStorage::Service::DiskService`][]. Rails also provides services for
Google Cloud Storage (`GCS`), Amazon S3 (`S3`), and `Mirror` which allows the
use of multiple services.

Custom services can be implemented by conforming to the interface defined
by [`ActiveStorage::Service`][].

See the [Active Storage guide](active_storage_overview.html) for
further details.

[`ActiveStorage::Service::DiskService`]: https://api.rubyonrails.org/classes/ActiveStorage/Service/DiskService.html
[`ActiveStorage::Service`]: https://api.rubyonrails.org/classes/ActiveStorage/Service.html

#### `config.active_storage.variant_processor`

Registers the processor used to build [variants](active_storage_overview.html#image-variants)
of uploaded blobs.

The accepted values are:

* `:mini_magick`: Uses the [`MiniMagick`](https://github.com/minimagick/minimagick) gem.
* `:vips`: Uses the [`ruby-vips`](https://github.com/libvips/ruby-vips) gem.
* `:disabled`: Turns off variant processing.

A custom class which implements the interface defined
by [`ActiveStorage::Transformers::Transformer`][] may also be specified.

```ruby
config.active_storage.variant_processor = CustomTransformer
```

The default value is `:vips`.

NOTE: The built-in image analyzers accept a blob only when
`variant_processor` is `:vips` or `:mini_magick`. When using a custom
processor add a custom analyzer to
[`config.active_storage.analyzers`](#config-active-storage-analyzers) as well.

[`ActiveStorage::Transformers::Transformer`]: https://api.rubyonrails.org/classes/ActiveStorage/Transformers/Transformer.html

#### `config.active_storage.analyzers`

Registers an array of analyzers available for Active Storage blobs.

The default is:

```ruby
[
  ActiveStorage::Analyzer::ImageAnalyzer::Vips,
  ActiveStorage::Analyzer::ImageAnalyzer::ImageMagick,
  ActiveStorage::Analyzer::VideoAnalyzer,
  ActiveStorage::Analyzer::AudioAnalyzer
]
```

The image analyzers can extract width and height of an image blob.

The video analyzer can extract width, height, duration, angle,
aspect ratio, and detect the presence of video or audio channels of a
video blob.

The audio analyzer can extract the duration and bit rate of an audio blob.

Disable analyzers by setting this to an empty array:

```ruby
config.active_storage.analyzers = []
```

#### `config.active_storage.analyze`

Controls when attachment analysis (image/video/audio metadata extraction)
is performed:

* `:immediately`: Analyze before validation, making metadata available
  for validations (for example: image dimensions, video duration).
* `:later`: Analyze after upload from local IO or via background job for
  direct uploads.
* `:lazily`: Skip automatic analysis and analyze on-demand.

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

Attachments with `process: :immediately` variants implicitly use
immediate analysis to ensure metadata is available before processing.

NOTE: Direct uploads bypass the server so the file isn't locally available
for analysis. In this case, `:immediately` falls back to `:later`, analyzing
via background job after upload completes. Metadata isn't available for validation.

The default value is `:immediately`.

#### `config.active_storage.previewers`

Registers an array of image previewers available for Active Storage blobs.

The default is:

```ruby
[
  ActiveStorage::Previewer::PopplerPDFPreviewer,
  ActiveStorage::Previewer::MuPDFPreviewer,
  ActiveStorage::Previewer::VideoPreviewer
]
```

`PopplerPDFPreviewer` and `MuPDFPreviewer` can generate a thumbnail from
the first page of a PDF blob. `VideoPreviewer` generates a thumbnail from a
_relevant_ frame of a video blob.

#### `config.active_storage.paths`

A hash of options defining the paths to the external binaries used
by previewers and analyzers such as `ffmpeg`.

The default is `{}`, meaning the commands will be looked for in the default path.

Any of these options may be included:

* `:ffprobe` - The location of the ffprobe executable.
* `:mutool` - The location of the mutool executable.
* `:ffmpeg` - The location of the ffmpeg executable.

```ruby
config.active_storage.paths[:ffprobe] = "/usr/local/bin/ffprobe"
```

#### `config.active_storage.variable_content_types`

Registers an array of MIME types that Active Storage can
transform using the variant processor.

The default value is:

```ruby
[
  "image/png", "image/gif", "image/jpeg", "image/tiff", "image/bmp",
  "image/vnd.adobe.photoshop", "image/vnd.microsoft.icon",
  "image/webp", "image/avif", "image/heic", "image/heif"
]
```

#### `config.active_storage.web_image_content_types`

Registers an array of MIME types which are regarded as web image content
types. Variants of these types can be processed without being converted to the
fallback PNG format.

For example, if you want to use `AVIF` variants in your application you can add
`image/avif` to this array.

The default value is:

```ruby
[
  "image/png",
  "image/gif",
  "image/webp"
]
```

#### `config.active_storage.content_types_to_serve_as_binary`

Registers an array of MIME types that Active Storage will always serve as an
attachment, rather than inline.

The default value is:

```ruby
[
  "text/html",
  "image/svg+xml",
  "application/postscript",
  "application/x-shockwave-flash",
  "text/xml",
  "application/xml",
  "application/xhtml+xml",
  "application/mathml+xml",
  "text/cache-manifest"
]
```

#### `config.active_storage.content_types_allowed_inline`

Registers an array of MIME types that Active Storage will serve as inline.

The default value is:

```ruby
[
  "image/webp", "image/avif", "image/png",
  "image/gif", "image/jpeg", "image/tiff", "image/bmp",
  "image/vnd.adobe.photoshop", "image/vnd.microsoft.icon",
  "application/pdf"
]
```

#### `config.active_storage.queues.analysis`

Sets the Active Job queue to use for analysis jobs.

When `nil`, analysis jobs are sent to the
[default Active Job queue](#config-active-job-default-queue-name).

The default value is `nil`.

#### `config.active_storage.queues.mirror`

Sets the Active Job queue to use for direct upload mirroring jobs.

When `nil`, mirroring jobs are sent to the
[default Active Job queue](#config-active-job-default-queue-name).

The default is `nil`.

#### `config.active_storage.queues.preview_image`

Sets the Active Job queue to use for preprocessing
previews of images.

When `nil`, preprocessing jobs are sent to the
[default Active Job queue](#config-active-job-default-queue-name).

The default is `nil`.

#### `config.active_storage.queues.purge`

Sets the Active Job queue to use for purge jobs.

When `nil`, purge jobs are sent to the
[default Active Job queue](#config-active-job-default-queue-name).

The default is `nil`.

#### `config.active_storage.queues.transform`

Sets the Active Job queue to use for preprocessing
variants.

When `nil`, preprocessing jobs are sent to the
[default Active Job queue](#config-active-job-default-queue-name).

The default is `nil`.

#### `config.active_storage.logger`

Registers a logger for Active Storage, conforming to the interface of `Log4r` or
the default Ruby `Logger` class.

Defaults to [`config.logger`](#config-logger).

Setting this option to `false` will turn off logs for Active Storage.

#### `config.active_storage.service_urls_expire_in`

Configures the default expiry of URLs generated by:

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

Configures the default expiry of URLs in the Rails application
generated by Active Storage. The default is `nil`.

#### `config.active_storage.touch_attachment_records`

A boolean flag which, when enabled, will `touch` the parent record
when an attachment is updated.

The default is `true`.

#### `config.active_storage.s3_public_uploads_via_acl`

A boolean flag which determines whether the [Amazon S3 service][]
sets the `public-read` ACL on uploads for services
configured with `public: true`.

The default and recommended [S3 object ownership](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)
setting is **Bucket owner enforced**. Buckets configured in this way
reject all requests specifying an ACL. Access should be controlled using a
bucket policy instead.

Set this to `false` to skip the ACL for buckets that don't accept one.

The default is `false`.

[Amazon S3 service]: https://api.rubyonrails.org/classes/ActiveStorage/Service/S3Service.html

#### `config.active_storage.routes_prefix`

Sets the route prefix for the routes served by Active Storage.

Accepts any value supported by `scope`, such as a string path prefix or a hash of
routing options.

The default is `"/rails/active_storage"`.

The below example demonstrates how Active Storage routes can be served
from a different subdomain.

```ruby
config.active_storage.routes_prefix = { path: "/files", subdomain: "assets" }
```

#### `config.active_storage.track_variants`

A boolean which determines whether variants are recorded in the database.

The default value is `true`.

#### `config.active_storage.draw_routes`

A boolean used to toggle whether Active Storage routes are generated.

The default is `true`.

#### `config.active_storage.draw_direct_upload_route`

A boolean used to toggle generation of the direct upload route, without
affecting the other Active Storage routes.

It has no effect when [`config.active_storage.draw_routes`](#config-active-storage-draw-routes)
is `false`.

When set to `false`, Action Text's `rich_textarea` renders without a
`data-direct-upload-url` unless one is passed explicitly, and a Trix editor
without that attribute hides its attach button and ignores dropped or pasted
files.

The default value is `true`.

#### `config.active_storage.resolve_model_to_route`

Sets the delivery mechanism for Active Storage files.

Allowed values are:

* `:rails_storage_redirect`: Redirect to signed, short-lived service URLs.
* `:rails_storage_proxy`: Proxy files by downloading them.

The default is `:rails_storage_redirect`.

#### `config.active_storage.video_preview_arguments`

Sets the arguments passed to `ffmpeg` when generating video previews:

The default value is

```ruby
"-vf 'select=eq(n\\,0)+eq(key\\,1)+gt(scene\\,0.015),loop=loop=-1:size=2,trim=start_frame=1' -frames:v 1 -f image2"
```

`-vf 'select=eq(n\\,0)+eq(key\\,1)+gt(scene\\,0.015)"` select the first video frame,
plus keyframes, plus frames that meet the scene change threshold.

`loop=loop=-1:size=2,trim=start_frame=1'` uses the first video frame as a
fallback when no other frames meet the criteria by looping the first
(one or) two selected frames, then dropping the first looped frame.

#### `config.active_storage.video_preview_input_arguments`

Configures the arguments passed to `ffmpeg` before `-i` when
generating video preview images.

`ffmpeg`'s flags are position dependent, so use this option to define
arguments that apply to the input such as `-codec_whitelist` and
`-protocol_whitelist`.

The default value is `""`.

See [Media Processing of File Uploads](security.html#media-processing-of-file-uploads)
in the Security Guide.

#### `config.active_storage.ffprobe_arguments`

Defines the arguments passed to `ffprobe` before the file path when analyzing
videos and audio. Applies to both `ActiveStorage::Analyzer::VideoAnalyzer` and
`ActiveStorage::Analyzer::AudioAnalyzer`.

Arguments that make `ffprobe` reject a file will fail that file's analysis.

The default value is `""`.

See [Media Processing of File Uploads](security.html#media-processing-of-file-uploads)
in the Security Guide.

#### `config.active_storage.multiple_file_field_include_hidden`

When an Active Storage [`has_many_attached`][] relationship is modified, the
current collection is _replaced_ by the new value.

This option controls whether the [`file_field`][] helper renders an auxillary
hidden field containing an empty collection of attachments.

It is enabled by default. Submitting a form with the hidden field generated
by this option will remove all currently attached files. To retain the existing
files, either, the hidden field needs to be omitted altogether, or the file
attributes need to be resubmitted with the form.

[`has_many_attached`]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/Model.html#method-i-has_many_attached
[`file_field`]: https://api.rubyonrails.org/classes/ActionView/Helpers/FormBuilder.html#method-i-file_field

#### `config.active_storage.precompile_assets`

Determines whether the Active Storage assets should be precompiled by the
asset pipeline. It has no effect where Sprockets isn't used.

The default value is `true`.

#### `config.active_storage.streaming_max_ranges`

[`ActiveStorage::Streaming`][] allows partial resources to be requested using
[HTTP Range Requests][], but this can be abused for denial of service attacks.

This option defines how many ranges a byte range request may contain.

The default value is `1`, which means a single range of bytes is accepted.
This allows for retries and works for the vast majority of use cases.

[`ActiveStorage::Streaming`]: https://api.rubyonrails.org/classes/ActiveStorage/Streaming.html
[HTTP Range Requests]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Range_requests

### Configuring Action Text

#### `config.action_text.attachment_tag_name`

Accepts a string for the HTML tag used to wrap attachments.

Defaults to `"action-text-attachment"`.

#### `config.action_text.sanitizer_vendor`

Configures the HTML sanitizer used by Action Text.

The default value is `Rails::HTML5::Sanitizer`.

NOTE: Rails will back back to `Rails::HTML4::Sanitizer` when running on
JRuby platforms, as `Rails::HTML5::Sanitizer` is not supported.

Database Configuration
----------------------

The `config/database.yml` file specifies the connection information for all
databases used in the Rails app.

It is keyed by the [_environments_](#environment-specific-configuration-files)
configured within Rails. A `default` block containing options common to all
environments may also be specified:

```yml
default: &default
  adapter: postgresql
  max_connections: 5

development:
  <<: *default
  database: myapp_development

test:
  <<: *default
  database: myapp_test

production:
  <<: *default
  database: myapp_production
```

In the development environment, this will connect to the database
named `myapp_development` using the `postgresql` adapter.

Alternatively, the same information can be formatted as a URL and provided
via an environment variable named `DATABASE_URL`:

```ruby
ENV["DATABASE_URL"] # => "postgresql://localhost/myapp_development?max_connections=5"
```

Or. the URL can also be specified in the `config/database.yml` file:

```yaml
development:
  url: postgresql://localhost/blog_development?max_connections=5
```

The [Connection Attribute Preference](#connection-attribute-preference)
section below explains how Rails handles conflicting information in
`config/database.yml` and `ENV["DATABASE_URL"]`.

The `config/database.yml` file can contain include Ruby code within
ERB tags `<%= %>`. This technique can be used to read environment variables
or calculate settings dynamically.

```yml
# ...

production:
  <<: *default
  user: <%= ENV["DATABASE_USER"] %>
  password: <%= ENV["DATABASE_PW"] %>
  database: myapp_production
```

NOTE: The database adapter for any given URL scheme can be modified using
[`config.active_record.protocol_adapters`](#config-active-record-protocol-adapters).
When connecting to a database using a URL, this option allows you to change the
database adapter without modifying the URL.

When an app connects to multiple databases, define each one under the environment
key:

```yml
# ...

production:
  primary:
    <<: *default
    username: <%= ENV["PRIMARY_DATABASE_USER"] %>
    password: <%= ENV["PRIMARY_DATABASE_PW"] %>
    database: myapp_primary_database
  primary_replica:
    <<: *default
    username: <%= ENV["REPLICA_DATABASE_USER"] %>
    password: <%= ENV["REPLICA_DATABASE_PW"] %>
    database: myapp_primary_database
    replica: true
  animals:
    <<: *default
    username: <%= ENV["ANIMALS_DATABASE_USER"] %>
    password: <%= ENV["ANIMALS_DATABASE_PW"] %>
    database: animals_database
    migrations_paths: db/animals_migrate
```

See the [Multiple Databases guide](active_record_multiple_databases.html)
for further information.

### Connection Attribute Preference

Database connections can be defined in the `config/database.yml` file, or
the `DATABASE_URL` environment variable. This section explains how data from
both these sources are married up.

When only one of the above sources is present for a given environment, it will
be used to connect to the database.

When both sources are present simultaneously for a given environment, Rails
will merge the information based on a number of rules (all below examples
assume that Rails is running in the `development` environment):

1.  The environment variable takes precendence when both sources contain
    conflicting information.

    ```yml
    # config/database.yml

    development:
      adapter: sqlite3
      timeout: 5000
      database: storage/development.sqlite3
    ```

    ```bash
    export DATABASE_URL="sqlite3:storage/app.sqlite3"
    bin/rails runner 'puts ActiveRecord::Base.connection_db_config.configuration_hash'
    # => {adapter: "sqlite3", timeout: 5000, database: "storage/app.sqlite3"}
    ```

2.  Non-conflicting values will be merged.

    ```yml
    # config/database.yml

    development:
      adapter: sqlite3
      timeout: 5000
    ```

    ```bash
    export DATABASE_URL="sqlite3:storage/development.sqlite3?max_connections=5"
    bin/rails runner 'puts ActiveRecord::Base.connection_db_config.configuration_hash'
    # => {adapter: "sqlite3", timeout: 5000, database: "storage/development.sqlite3", max_connections: "5"}
    ```

3.  When a `url` key is specified in the `config/database.yml`, it will completely
    override the `DATABASE_URL` environment variable:

    ```yml
    # config/database.yml

    development:
      url: "sqlite3:storage/development.sqlite3"
    ```

    ```bash
    export DATABASE_URL="sqlite3:storage/app.sqlite3?max_connections=5"
    bin/rails runner 'puts ActiveRecord::Base.connection_db_config.configuration_hash'
    # => {adapter: "sqlite3", database: "storage/development.sqlite3"}
    ```

In production, using the `DATABASE_URL` environment variable is recommended,
as well as showing its usage explicitly in `config.database.yml`:

```yml
# config/database.yml

# ...

production:
  url: <%= ENV['DATABASE_URL'] %>
```

### Configuring a SQLite3 Database

Rails connects to a SQLite3 database using the
[`sqlite3`](https://github.com/sparklemotion/sqlite3-ruby) gem.

The built-in adapter configures a production-ready connection. See
[`ActiveRecord::ConnectionAdapters::SQLite3Adapter`](https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SQLite3Adapter.html) for details.

This is the default adapter which is configured when creating a new Rails app.
Another adapter may be specified using the
[`--database` option](command_line.html#configure-a-different-database).

Here's an example SQLite database configuration:

```yaml
development:
  adapter: sqlite3
  database: storage/development.sqlite3
  max_connections: 5
  timeout: 5000
```

[SQLite extensions](https://sqlite.org/loadext.html) are supported when using
`sqlite3` gem v2.4.0 or later:

``` yaml#4-6
development:
  adapter: sqlite3
  database: storage/development.sqlite3
  extensions:
    - SQLean::UUID                     # module name responding to `.to_path`
    - .sqlpkg/nalgeon/crypto/crypto.so # or a filesystem path
    - <%= AppExtensions.location %>    # or ruby code returning a path
```

### Configuring a MySQL or MariaDB Database

Rails connects to a MySQL database using the
[`mysql2`](https://github.com/brianmario/mysql2) gem.

Ensure this gem is installed in your Gemfile, and then specify `mysql2` as
the adapter in your database configuration. Here's an example:

```yaml
development:
  adapter: mysql2
  database: myapp_development
  max_connections: 5
  host: 127.0.0.1
```

Ensure you specify a `username` and `password` if your database requires one.

Advisory locks are enabled by default on MySQL and are used to make database
migrations concurrency safe. This can be disabled using:

```yaml#3
production:
  adapter: mysql2
  advisory_locks: false
  # ...
```

### Configuring a PostgreSQL Database

Rails connects to a PostgreSQL database using the
[`pg`](https://github.com/ged/ruby-pg) gem.

Ensure this gem is installed in your Gemfile, and then specify `postgresql` as
the adapter in your database configuration. Here's an example:

```yaml
development:
  adapter: postgresql
  database: myapp_development
  max_connections: 5
```

Ensure you specify a `username` and `password` if your database requires
it.

When running tasks that create or drop databases (`db:create`, `db:drop`,
and `db:purge`), Active Record connects to the `postgres` database by
default. If your PostgreSQL server does not have a `postgres` database,
as with some managed services, set `maintenance_database` to a database
that you can connect to:

```yaml#3
production:
  adapter: postgresql
  maintenance_database: defaultdb
```

It may also be specified in the `DATABASE_URL` environment variable:

```
postgres://.../myapp_production?maintenance_database=defaultdb
```

Advisory locks are enabled by default on PostgreSQL and are used to
make database migrations concurrency safe. This can be disabled using:

```yaml
production:
  adapter: postgresql
  advisory_locks: false
```

Active Record automatically maintains a cache of prepared statements when using
PostgreSQL. By default, the limit is set to `1000` statements. Change it using:

```yaml#3
production:
  adapter: postgresql
  statement_limit: 200
```

Or disable prepared statements completely:

```yaml#3
production:
  adapter: postgresql
  prepared_statements: false
```

Prepared statements can speed up query execution and planning, but will
use more memory on the database server.

### Configuring the Database on JRuby

Connecting to the database on the JRuby platform requires the
[`activerecord-jdbc-adapter`](https://github.com/jruby/activerecord-jdbc-adapter) gem.
The Readme contains the details for each supported database.

### Configuring Metadata Storage

Rails stores information about the environment and schema in a table
named `ar_internal_metadata`.

This can be disabled for a specific connection by setting
`use_metadata_table`:

```yaml#3
production:
  adapter: postgresql
  use_metadata_table: false
```

This may be useful when connecting to a shared database where the Rails
app cannot create new tables.

### Configuring Retry Behavior

Rails will automatically reconnect to the database server and retry certain queries
if something goes wrong.

The number of retries is set to `1` by default, but can be customized:

```yaml#3
production:
  adapter: mysql2
  connection_retries: 3 # Set to `0` to disable retries
```

Only idempotent queries — which are safe to retry — will be retried.

A `retry_deadline` may also be specified. This is a time period in seconds
after which the query will not be retried.

```yaml#3
production:
  adapter: mysql2
  retry_deadline: 5 # Stop retrying queries after 5 seconds
```

In the above example, a query will not be retried after more than 5 seconds
since the first attempt, even if the maximum retry count hasn't been hit.

This value is `nil` by default, which means that there is no time limit
for retries.

### Configuring Query Cache

Rails maintains a cache for result sets returned by queries. When Rails
encounters the same query again for a given request or job, the cached result
will be used instead of hitting the database again.

The query cache is stored in memory where the least recently used query is
evicted when the cache ceiling is hit. The default cache size is `100`, but
can be customized in the `database.yml`.

```yaml#3
production:
  adapter: mysql2
  query_cache: 200
```

Disable query caching by setting this option to `false`:

```yaml#3
production:
  adapter: mysql2
  query_cache: false
```

Custom Rails Environments
-------------------------

By default Rails ships with three environments: `development`, `test`, and
`production`. Creating additional environments is not recommended — instead,
the preferred approach is to use environment variables to modify app configuration
for _production-like_ environments such as _staging_.

However, it is possible to create custom environments if required. For example, these
are the steps required to create a `staging` environment.

1. Create a configuration file for the environment (`config/environments/staging.rb`)
2. Define the environment specific configuration, using `require_relative` to load
  configurations from other environment files if required.

The environment can now be used:

```bash
$ bin/rails server -e staging
```

NOTE: In a practical setting, you can run a staging server by using environment
variables to configure databases and other external services, while using the
`production` Rails environment. A custom environment causes additional complexity
in environment-specific app logic, and when defining bundler groups in your Gemfile.
As such it is not recommended.

Deploying to a Subdirectory
---------------------------

By default Rails expects that your application is running at the root
of your domain — at: `https://example.com/`. Deploying to a subdirectory
means that the app is accessed via a path prefix: `https://example.com/app`.
In this example, the app has been deployed to the `app` subdirectory,
and this needs to be configured in Rails.

```ruby
config.relative_url_root = "/app"
```

Alternatively, this option can be configured by setting the
`RAILS_RELATIVE_URL_ROOT` environment variable.

Rails will now prepend "/app" when generating links.
