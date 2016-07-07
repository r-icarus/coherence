# Coherence

[![Build Status](https://travis-ci.org/smpallen99/coherence.png?branch=master)](https://travis-ci.org/smpallen99/coherence)  [![License][license-img]][license]

[license-img]: http://img.shields.io/badge/license-MIT-brightgreen.svg
[license]: http://opensource.org/licenses/MIT

> <div style="font-color: red">Alert: Project under active development!</div>
>
> This is really a 0.0.1 version. It has not been fully acceptance tested. For experimental use only!
>
> When I think its ready, I release it on hex!
>
> Thanks for your interest.

Coherence is a full featured, configurable authentication system for Phoenix, with the following modules:

* [Database Authenticatable](#authenticatable): handles hashing and storing an encrypted password in the database.
* [Invitable](#invitable): sends invites to new users with a sign-up link, allowing the user to create their account with their own password.
* [Registerable](#registerable): allows anonymous users to register a users email address and password.
* [Recoverable](#recoverable): provides a link to generate a password reset link with token expiry.
* [Trackable](#trackable): saves login statics like login counts, timestamps, and IP address for each user.
* [Lockable](#lockable): locks an account when a specified number of failed sign-in attempts has been exceeded.
* [Unlockable With Token](#unlockable-with-token): provides a link to send yourself an unlock email.
* [Rememberable](#remember-me): provides persistent login with 'Remember me?' check box on login page.

Coherence provides flexibility by adding namespaced templates and views for only the options specified by the `mix coherence.install` command. This boiler plate code is added to your `web/templates/coherence` and `web/views/coherence` directories.

Once the boilerplate has been generated, you free to customize the source as required.

As well, a `web/coherence_web.ex` is added. Migrations are also generated to add the required database fields.

## Installation

  1. Add coherence to your list of dependencies in `mix.exs`:

        def deps do
          [{:coherence, github: "smpallen99/coherence"}]
        end

  2. Ensure coherence is started before your application:

        def application do
          [applications: [:coherence]]
        end

## Getting Started

First, decide with modules you would like to use for your project. For the following example were are going to use a full install except for the registerable option.

Run the installer

```bash
$ mix coherence.install --full-invitable
```

This will add the coherence configuration to the end of your `config/config.exs` file. Please review this file as there are a couple items you will need to customize like email address and mail api_key.

See [Installer](#installer) for more install options.

You will need to update a few files manually.

```elixir
# web/router.ex

defmodule MyProject.Router do
  use MyProject.Web, :router
  use Coherence.Router         # Add this

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
    plug Coherence.Authentication.Database, db_model: MyProject.User  # Add this
  end

  pipeline :public do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
    plug Coherence.Authentication.Database, db_model: MyProject.User, login: false  # Add this
  end

  scope "/" do
    pipe_through :public
    coherence_routes :public     # Add this
  end

  scope "/" do
    pipe_through :browser
    coherence_routes :private    # Add this
  end
  # ...
end
```

Update your user model like:

```elixir
# web/models/user.ex

defmodule MyProject.User do
  use MyProject.Web, :model
  use Coherence.Schema                                    # Add this

  schema "users" do
    field :name, :string
    field :email, :string
    coherence_schema                                      # Add this

    timestamps
  end

  def changeset(model, params \\ %{}) do
    model
    |> cast(params, [:name, :email] ++ coherence_fields)  # Add this
    |> validate_required([:name, :email])
    |> validate_coherence(params)                         # Add this
  end
end
```

## Option Overview

### Authenticatable

Handles hashing and storing an encrypted password in the database.

Provides `/sessions/new` and `/sessions/delete` routes for logging in and out with
the appropriate templates and view.

The following columns are added the `<timestamp>_add_coherence_to_user.exs` migration:

* :encrypted_password, :string - the encrypted password

### Invitable

Handles sending invites to new users with a sign-up link, allowing the user to create their account with their own password.

Provides `/invitations/new` and `invitations/edit` routes for creating a new invitation and creating a new account from the invite email.

These routes can be configured to require login by using the `coherence_routes :private` macro in your router.exs file.

Invitation token timeout will be added in the future.

The following table is created by the generated `<timestamp>_create_coherence_invitable.exs` migration:

```elixir
create table(:invitations) do
  add :name, :string
  add :email, :string
  add :token, :string
end
```

### Registerable

Allows anonymous users to register a users email address and password.

Provides `/registrations/new` and `/registrations/create` routes for creating a new registration.

Adds a `Register New Account` to the log-in page.

It is recommended that the :confirmable option is used with :registerable to
ensure a valid email address is captured.

### Recoverable

Allows users to reset their password using an expiring token send by email.

Provides `new`, `create`, `edit`, `update` actions for the `/passwords` route.

Adds a "Forgot your password?" link to the log-in form. When clicked, the user provides their email address and if found, sends a reset password instructions email with a reset link.

The expiry timeout can be changed with the `:reset_token_expire_days` config entry.

### Trackable

Saves login statics like login counts, timestamps, and IP address for each user.

Adds the following database field to your User model with the generated migration:

```elixir
add :sign_in_count, :integer, default: 0  # how many times the user has logged in
add :current_sign_in_at, :datetime        # the current login timestamp
add :last_sign_in_at, :datetime           # the timestamp of the previous login
add :current_sign_in_ip, :string          # the current login IP adddress
add :last_sign_in_ip, :string             # the IP address of the previous login
```

### Lockable

Locks an account when a specified number of failed sign-in attempts has been exceeded.

The following defaults can be changed with the following config entries:

* `:unlock_timeout_minutes`
* `:max_failed_login_attempts`

Adds the following database field to your User model with the generated migration:

```elixir
add :failed_attempts, :integer, default: 0
add :unlock_token, :string
add :locked_at, :datetime
```

### Unlockable with Token

Provides a link to send yourself an unlock email. When the user clicks the link, the user is presented a form to enter their email address and password. If the token has not expired and the email and password are valid, a unlock email is sent to the user's email address with an expiring token.

The default expiry time can be changed with the `:unlock_token_expire_minutes` config entry.

### Remember Me

The `rememberable` option provides persistent login when the 'Remember Me?' box is checked during login.

With this feature, you will automatically be logged in from the same browser when your current login session dies using a configurable expiring persistent cookie.

For security, both a token and series number stored in the cookie on initial login. Each new creates a new token, but preserves the series number, providing protection against fraud. As well, both the token and series numbers are hashed before saving them to the database, providing protection if the database is compromised.

The following defaults can be changed with the following config entries:

* :rememberable_cookie_expire_hours (2*24)
* :login_cookie                     ("coherence_login")

The following table is created by the generated `<timestamp>_create_coherence_rememberable.exs` migration:

```elixir
create table(:rememberables) do
  add :series_hash, :string
  add :token_hash, :string
  add :token_created_at, :datetime
  add :user_id, references(:users, on_delete: :delete_all)

  timestamps
end
create index(:rememberables, [:user_id])
create index(:rememberables, [:series_hash])
create index(:rememberables, [:token_hash])
create unique_index(:rememberables, [:user_id, :series_hash, :token_hash])
```

The `--rememberable` install option is not provided in any of the installer group options. You must provide the `--rememberable` option to install the migration and its support.

## Installer

The following examples illustrate various configuration scenarios for the install mix task:

```bash
    # Install with only the `authenticatable` option
    $ mix coherence.install

    # Install all the options except `confirmable` and `invitable`
    $ mix coherence.install --full

    # Install all the options except `invitable`
    $ mix coherence.install --full-confirmable

    # Install all the options except `confirmable`
    $ mix coherence.install --full-invitable

    # Install the `full` options except `lockable` and `trackable`
    $ mix coherence.install --full --no-lockable --no-trackable

    # Remove all the coherence generated boilerplate files
    $ mix coherence.install --clean
```

Run `$ mix help coherence.install` for more information.

## Customization

The `coherence.install` mix task generates a bunch of boiler plate code so you can easily customize the views, templates, and mailer.

Also, checkout the Coherence.Config module for a list of config items you can use to tune the behaviour of Coherence.

### Custom Controllers

By default, controller boilerplate is not generated unless the `--controllers` option is provided to `mix coherence.install`.

The generated controllers are named `MyProject.Coherence.SessionController` as an example. Generated controllers are located in `web/controllers/coherence/`

If the controllers are generated, you will need to change your router to use the new names. For example:

```elixir
  # web/router.ex
  use MyProject.Web, :router
  use Coherence.Router

  # ...

  scope "/", MyProject do   # note the addition of MyProject
    pipe_through :public
    coherence_routes :public
  end

  scope "/", MyProject do   # note the addition of MyProject
    pipe_through :browser
    coherence_routes :private
  end
  # ...
end
```

## License

`coherence` is Copyright (c) 2016 E-MetroTel

The source is released under the MIT License.

Check [LICENSE](LICENSE) for more information.


