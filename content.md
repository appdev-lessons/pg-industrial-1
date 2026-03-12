# Photogram Industrial: Image uploads, devise users, and photos scaffold

## Getting started

This project includes automated tests, so click on this button to get started:

LTI{Load Photogram Industrial assignment}(https://grades.firstdraft.com/launch)[S9ymPy6WCsn18gLbByVbZQ7k]{vfdtzJb5bLYqYwuqgeRKpc5d}(10)[Photogram Industrial Project]

The project, `pg-industrial`, spans over several of the next lessons. Keep the codespace open when you move on to those lessons in order to build on your progress through the series.

<div class="alert alert-danger">

You will need to go all the way through the lesson series and implement everything to get the `grade` tests to pass, which all start out failing. Don't panic if you see an error message in Grades about tests not being run prior to adding your User, Photo, Like, Comment, and FollowRequest models.

**When you finish this lesson `grade` tests will not run, you will gain the points for the project as you move deeper into the lesson series.**
</div>

Here is the target that we will work towards:

[pg-industrial.matchthetarget.com](https://pg-industrial.matchthetarget.com/)

This time around, Photogram will be _industrial grade_ — the kind of code you could charge money for. We'll use database indexes and constraints, advanced association accessors, scopes, validations, view helper methods like `link_to` and `form_with` everywhere, partials to DRY up code judiciously, the Devise gem for authentication and password reset emails, Active Storage for real image uploads via Cloudinary, and many other industrial-strength upgrades.

This is like finishing school. We're going to learn how to level up to write a codebase that we can onboard professional developers to; and which LLMs can make quick sense of based on their professional code training data.

Launch the codespace for your forked project and get the live preview running with `bin/server`. You'll see we're starting from scratch.

## The data model

Before we dive into code, let's look at the full data model we're building towards:

![Photogram ERD](/assets/pg-erd.png)

We have five tables: Users, Photos, Comments, Likes, and FollowRequests (and an additional table `ActiveStorage` related to image uploads). In this first lesson, we'll focus on the **Users** and **Photos** tables.

Importantly, there's the `FollowRequest` table, which keeps track of who's following whom. We have a `status` column in `FollowRequest` because this is going to be a permissioned social network. When somebody sends a follow request, it starts as "pending," and the recipient has to accept it before the follower can see their posts.

## Git workflow

We're going to practice the professional git workflow of creating branches, committing to them, and merging back to `main`. That way, we can leave feedback in the form of comments on your pull requests, with line-by-line comments on your actual code.

Let's create our first branch now in the terminal:

```
git checkout -b create-database
```

We'll work on this branch for the rest of the lesson.

## Adding gems

Our starting point is a bare Rails 8 app. The Gemfile has basic Rails gems, but it's missing several that we need.

Open your `Gemfile` and add the following gems **outside** of any `group` block (we want these available in all environments, not just development or test):

```ruby
gem "devise"                          # User authentication (sign up, sign in, etc.)
gem "strip_attributes"                # Remove whitespace from model attributes
gem "validate_url"                    # URL validation for models
gem "faker"                           # Generate fake data for seeds
gem "cloudinary"                      # Cloud image storage and CDN
gem "ransack"                         # Search and filtering
```
{: filename="Gemfile" }

<aside>
Why outside of any group? Gems in the `:development` group are only loaded while developing, like `better_errors` for debugging. We don't want those in production because they waste memory. But gems like `devise` and `cloudinary` need to work everywhere: development, test, _and_ production. That's why they go outside any group block.
</aside>

Now install them:

```
bundle install
```

Now would be a good time for a commit:

```
git add -A
git commit -m "Added required gems to Gemfile"
```

## Setting up Cloudinary

In previous projects, we might have stored uploaded images locally in the `public/` folder. That works fine in development for a few images, but when you're handling lots of images (like in Photogram), or when you deploy to production via a service like Render, the `public/` folder won't work. We need a cloud storage service, and we'll use Cloudinary.

### Configure environment variables

To prepare, create a file called `.env` in the root of your project with three empty keys:

```
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
```
{: filename=".env" copyable }

Now, let's get those keys.

<aside>

Never commit your `.env` file to git. It contains secret API keys. The `.gitignore` file in the starting point already excludes it, but double-check that `.env` appears in your `.gitignore` if you're not sure.
</aside>

### Retrieve Cloudinary keys

If you don't already have one, go to [Cloudinary](https://cloudinary.com/users/register_free) and sign up for a free account using your existing GitHub account. Once you're logged in, click "Home" on the left menu and go to your "Dashboard." You'll need these three values:

1. **Cloud name**: should be at the top of your dashboard page.
2. **API Key**: to access the API values, click "Go to API Keys" next to your "Cloud name."
3. **API Secret**: shown as hidden value next to the API Key. You will need to enter an email-delivered code to reveal this one.

As you find each value, enter it as the value for the corresponding key in your `.env` file in your codespace.

### Create the Cloudinary initializer

Now we need to tell Rails how to connect to Cloudinary. Create a new file:

```ruby
Cloudinary.config do |config|
  config.cloud_name = ENV.fetch("CLOUDINARY_CLOUD_NAME")
  config.api_key = ENV.fetch("CLOUDINARY_API_KEY")
  config.api_secret = ENV.fetch("CLOUDINARY_API_SECRET")
  config.cdn_subdomain = true
end
```
{: filename="config/initializers/cloudinary.rb" copyable }

We use `ENV.fetch` instead of `ENV[]` because `fetch` will raise a helpful error message if the environment variable is missing, rather than silently returning `nil` and causing confusing errors later.

### Configure storage.yml

Open `config/storage.yml`. You should see a commented-out section for Cloudinary:

```yaml{7-10}
# ...

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

# Uncomment and change the folder name when you set up Cloudinary
# cloudinary:
#   service: Cloudinary
#   folder: appdev_2

# Leave this alone — serves sample_data images from the First Draft AppDev-2 Cloudinary account
cloudinary_sample_data:
  service: Cloudinary
  cloud_name: dzhwwlb9e

# ...
```
{: filename="config/storage.yml" }

 Uncomment it and update the folder name so it looks like this:

```yaml{8-10}
# ...

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

# Uncomment and change the folder name when you set up Cloudinary
cloudinary:
  service: Cloudinary
  folder: photogram

# Leave this alone — serves sample_data images from the First Draft AppDev-2 Cloudinary account
cloudinary_sample_data:
  service: Cloudinary
  cloud_name: dzhwwlb9e

# ...
```
{: filename="config/storage.yml" }

There's also a `cloudinary_sample_data` section in the file. Leave that as-is. It will be used by the sample data task.

### Point Active Storage to Cloudinary

Open `config/environments/development.rb` and find the line that says:

```ruby
config.active_storage.service = :local
```

Change it to:

```ruby{1:(33-43)}
config.active_storage.service = :cloudinary
```
{: filename="config/environments/development.rb" }

This tells Active Storage to use the `cloudinary` service that we just uncommented in the `config/storage.yml` for file uploads, instead of the `local` filesystem (which you may have also noticed under the `local:` section of the `config/storage.yml`).

For good measure, you can also find the `config.active_storage.service` line in the `config/environments/production.rb` file and also change it from `:local` to `:cloudinary`. That will prepare us in case we deploy the app later.

Phew! We're all set up to use Cloudinary for image uploads. Now would be a good time for a commit:

```
git add -A
git commit -m "Configured Cloudinary for image uploads"
```

## Installing Active Storage

Active Storage is a built-in Rails framework for uploading files and attaching them to Active Record models. Unlike the old approach of storing filenames as strings in database columns, Active Storage uses its own set of tables to track file attachments.

Run the following at the terminal to install Active Storage:

```
rails active_storage:install
```

This creates a migration that adds three tables: `active_storage_blobs`, `active_storage_attachments`, and `active_storage_variant_records`. These tables work together to manage file uploads: blobs store metadata about the file, attachments link blobs to your models, and variant records track image transformations.

Go ahead and migrate:

```
rails db:migrate
```

And commit:

```
git add -A
git commit -m "Installed Active Storage"
```

## Installing Devise

Now let's set up Devise, the gem that handles user authentication (sign up, sign in, sign out, password resets, and more).

First, run the Devise installer:

```
rails generate devise:install
```

The installer prints a list of manual setup steps in the terminal. We don't need to do all of them now. One that we should do is defining a root route. We don't have any resources yet, but we know that `users#feed` will be our homepage eventually. Let's add it now:

```ruby{4}
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check
  
  root "users#feed"
end
```
{: filename="config/routes.rb" }

<aside>

We hadn't seen it before, but the `"/up"` route comes out-of-the-box with any modern Rails app. It's a "health check" placed at the top of the `routes.rb` and used by deployment services to periodically check that the app is running.
</aside>

This will cause an error if we visit the root URL right now (since we don't have a `UsersController` yet), but that's fine. We'll build it in a later lesson.

The other thing we should do now is double check this setting in our `development.rb` file:

```rb{}
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
```
{: filename="config/environments/development.rb"}

Development email configuration should already be setup, but check the file and confirm.

We won't worry about flash messages or view templates for now (items 3 and 4). We'll get to those later.

Now would be a good time for a commit:

```
git add -A
git commit -m "Installed Devise and added root route"
```

## Generating the User model with Devise

Instead of using the standard `rails generate model` command, we use Devise's generator. This gives us authentication columns (email, encrypted_password, etc.) for free, plus any custom columns we specify. Use the copy button the left side to make sure you get this generator command without any typos:

```
rails g devise User username display_name avatar_image profile_banner bio website private:boolean likes_count:integer comments_count:integer photos_count:integer
```
{: copyable }

As a reminder: `g` is short for `generate`, and I dropped `:string` after `username`, `display_name`, etc. because `string` is the default datatype.

This command does several things:
- Creates a migration file in `db/migrate/`
- Creates `app/models/user.rb` with Devise modules configured
- Adds `devise_for :users` to `config/routes.rb`, which gives us routes like `/users/sign_in`, `/users/sign_up`, `/users/sign_out`, and more

Let's commit the generated files before we start editing:

```
git add -A
git commit -m "Generated User model with Devise"
```

## Editing the Users migration

Before we migrate, let's open the migration file and make some important improvements. You'll find it in `db/migrate/`. It will be named something like `<timestamp>_devise_create_users.rb`. We'll walk through each change one at a time.

### Case-insensitive text with citext

Add `enable_extension("citext")` on the very first line inside `create_table`, and change the `email` column from `t.string` to `t.citext`:

```ruby{2,4:(9-14)}
    create_table :users do |t|
      enable_extension("citext")
      ## Database authenticatable
      t.citext :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""
      # ...
```
{: filename="db/migrate/<timestamp>_devise_create_users.rb" }

Then, further down in the same block, change the `username` column from `t.string` to `t.citext` as well:

```ruby{4:(9-14)}
      # ...
      # t.datetime :locked_at

      t.citext :username
      t.string :display_name
      # ...
```
{: filename="db/migrate/<timestamp>_devise_create_users.rb" }

This enables PostgreSQL's `citext` (case-insensitive text) extension. Why does this matter? Without `citext`, if someone signs up as `Alice@Example.com` and later tries to sign in with `alice@example.com`, the database would treat those as different values. With `citext`, the database handles case-insensitive comparisons automatically, so there's no need to call `.downcase` before every lookup.

<aside>
This is a PostgreSQL-specific feature. PostgreSQL has many powerful features like this (JSON datatypes, range datatypes, geographic distance ordering, full-text search), and Rails provides first-class support for many of them. [See this Rails Guide for a rundown.](https://guides.rubyonrails.org/active_record_postgresql.html)
</aside>

### Preventing blank usernames

We can also add `null: false` to the `username` column to prevent blank usernames from entering the database:

```ruby{1:(25-37)}
      t.citext :username, null: false
```
{: filename="db/migrate/<timestamp>_devise_create_users.rb" }

This is a **database-level constraint** that prevents a row from being saved with a `NULL` username. It's a stronger guarantee than a Rails validation alone, because it protects against race conditions and any code that might bypass ActiveRecord.

### Default values

We can also set sensible defaults on several columns:

```ruby{3:(25-39),4:(29-40),5:(32-43),6:(30-41)}
      # ...
      t.string :website
      t.boolean :private, default: true
      t.integer :likes_count, default: 0
      t.integer :comments_count, default: 0
      t.integer :photos_count, default: 0

      t.timestamps null: false
      # ...
```
{: filename="db/migrate/<timestamp>_devise_create_users.rb" }

Whenever you generate a model, it's a good habit to think about default values for each column. For counter columns, starting at `0` makes much more sense than `nil`. For the `private` column, we want new accounts to be private by default. Users can opt in to making their profile public later.

### Indexes and uniqueness constraints

At the bottom of the migration, Devise already added indexes for `email` and `reset_password_token`. Add one more for `username`:

```ruby{4}
    # ...
    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    add_index :users, :username,             unique: true
    # ...
  end
end
```
{: filename="db/migrate/<timestamp>_devise_create_users.rb" }

An index is like the index at the back of a book: it lets the database find records quickly without scanning every row. Since we'll frequently look up users by `username` (e.g., for profile URLs like `/alice`), an index here is essential.

The `unique: true` option adds a **database constraint** enforcing uniqueness. This is stronger than an ActiveRecord `validates :uniqueness` alone, which is susceptible to race conditions.

<aside>
An ActiveRecord model validation checks uniqueness by first querying the database to see if a matching record exists, then inserting the new record. But between those two steps, another request could sneak in and insert a duplicate. A database-level uniqueness constraint prevents this entirely. The database itself will reject the duplicate.
</aside>

Now migrate:

```
rails db:migrate
```

And commit:

```
git add -A
git commit -m "Edited and migrated Users table with citext, defaults, and indexes"
```

## Configuring ApplicationRecord

Before we configure the User model, let's add `strip_attributes` to `ApplicationRecord` so that _every_ model in our app benefits from it:

```ruby{4}
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class

  strip_attributes
end
```
{: filename="app/models/application_record.rb" }

We previously added `gem "strip_attributes"` to our `Gemfile`, so we now have the `strip_attributes` method available throughout the codebase. This method automatically removes leading and trailing whitespace from all string attributes before saving. This prevents issues like a user accidentally signing up with `" alice "` as their username. Since we put it in `ApplicationRecord`, every model that inherits from it (which is all of them) gets this behavior for free.

## Configuring the User model

Open `app/models/user.rb`. Devise already generated some code for us. We're going to add Active Storage attachments, an association, validations, and a callback. Let's walk through each change.

### Active Storage attachments

Add the Active Storage `has_one_attached` attachment declarations after the Devise configuration:

```ruby{6,8,16-17}
# == Schema Information
#
# Table name: users
#
#  id                     :bigint           not null, primary key
#  avatar_image           :string
#  ...
#  profile_banner         :string
# ...
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  has_one_attached :avatar_image, dependent: :purge_later
  has_one_attached :profile_banner, dependent: :purge_later
end
```
{: filename="app/models/user.rb" }

These declarations tell Active Storage that a User can have an avatar image and a profile banner attached in the `avatar_image` and `profile_banner` columns that we prepared. The `dependent: :purge_later` option means that when a user is deleted, their attached images will be automatically cleaned up in a background job.

### The association

Next, add the association for photos:

```ruby{4}
  # ...
  has_one_attached :profile_banner, dependent: :purge_later

  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy
end
```
{: filename="app/models/user.rb" }

We're calling the association `own_photos` (not just `photos`) because a user might interact with many photos they don't own (through likes, comments, etc.). The `foreign_key: :owner_id` tells Rails to look for the `owner_id` column on the `photos` table, and `class_name: "Photo"` clarifies which model to use since the association name doesn't match the model name. The `dependent: :destroy` ensures that when a user is deleted, all their photos are deleted too.

### Username validation

Add the username validation:

```ruby{3-8}
  # ...
  has_many :own_photos, foreign_key: :owner_id, class_name: "Photo", dependent: :destroy

  validates :username,
    presence: true,
    uniqueness: true,
    format: {
      with: /\A[\w_\.]+\z/i,
      message: "can only contain letters, numbers, periods, and underscores"
    }
  # ...
```
{: filename="app/models/user.rb" }

We require a username, enforce uniqueness (at the Rails level, on top of our database constraint), and restrict the format to letters, numbers, periods, and underscores, just like Instagram. The regex `\A[\w_\.]+\z` means: from the start of the string (`\A`), one or more word characters, underscores, or periods (`[\w_\.]+`), to the end of the string (`\z`).

### Website validation

Add the website validation:

```ruby{3}
  # ...
    }

  validates :website, url: { allow_blank: true }
  # ...
```
{: filename="app/models/user.rb" }

This uses the `validate_url` gem we installed earlier. If a user provides a website, it must be a valid URL. But it's optional. `allow_blank: true` means they can leave it empty.

### Default avatar callback

Add the callback and method at the end of the class:

```ruby{3,5-12}
  # ...
  validates :website, url: { allow_blank: true }

  before_create :set_default_avatar

  def set_default_avatar
    image = "https://res.cloudinary.com/dzhwwlb9e/image/upload/v1773240782/960px-Default_pfp.svg_dpntzd_ga9htr.png"
    avatar_image.attach(
      io: URI.open(image),
      filename: image.split("/").last,
      content_type: "image/jpg"
    )
  end
end
```
{: filename="app/models/user.rb" }

The `before_create` callback runs just before a new user record is saved for the first time. It downloads a default avatar image from Cloudinary and attaches it to the user. This way, every user starts with a profile picture rather than a broken image link.

Now would be a good time for a commit:

```
git add -A
git commit -m "Configured ApplicationRecord and User model"
```

## Generating the Photos scaffold

Now let's generate the Photos resource. Since users will be creating, viewing, editing, and deleting photos, we want a full scaffold:

```
rails generate scaffold photo image caption:text owner:references pinned:boolean comments_count:integer likes_count:integer
```

Notice that we used `owner:references` instead of `owner_id:integer`. The `references` type does several things for us:
- Creates the column as `owner_id` (following Rails conventions)
- Adds `null: false` by default
- Adds a database index on the column
- Adds a `belongs_to :owner` association in the model
- Adds a foreign key constraint in the migration

Let's commit the generated files before editing:

```
git add -A
git commit -m "Generated Photos scaffold"
```

## Editing the Photos migration

Open the generated migration file in `db/migrate/`. We need to make a few changes. Let's walk through each one.

### Foreign key to the correct table

The generator created `t.references :owner, null: false, foreign_key: true`. But `foreign_key: true` tells the database to look for a table called `owners`, which doesn't exist! Our table is `users`. We fix this by specifying the target table explicitly:

```ruby{4:(41-73)}
      # ...
      t.text :caption
      t.belongs_to :owner, null: false, foreign_key: { to_table: :users }, index: true
      t.boolean :pinned, default: false, null: false
      # ...
```
{: filename="db/migrate/<date-time-of-migration>_create_photos.rb" }

<aside>
`t.belongs_to` and `t.references` are aliases. They do exactly the same thing. I used `belongs_to` here just because it reads nicely.
</aside>

### Default values

Just like with the Users migration, we set sensible defaults:

```ruby{3:(30-43),4:(30-40),5:(32-43)}
      # ...
      t.belongs_to :owner, null: false, foreign_key: { to_table: :users }, index: true
      t.boolean :pinned, default: false, null: false
      t.integer :comments_count, default: 0
      t.integer :likes_count, default: 0

      t.timestamps
      # ...
```
{: filename="db/migrate/<date-time-of-migration>_create_photos.rb" }

New photos start unpinned (`false`) and with zero likes and comments.

Now migrate:

```
rails db:migrate
```

## Configuring the Photo model

Open `app/models/photo.rb`. The generator gave us a `belongs_to :owner`, but it doesn't know that `owner` refers to the `User` model. Let's walk through each change.

### Active Storage for images

Add the Active Storage declaration at the top of the class:

```ruby{2}
class Photo < ApplicationRecord
  has_one_attached :image, dependent: :purge_later

  belongs_to :owner, class_name: "User", counter_cache: true
  # ...
```
{: filename="app/models/photo.rb" }

Just like with the User's avatar, we declare that a Photo has an attached image managed by Active Storage.

### The belongs_to association

Update the generated `belongs_to` to specify the class name and counter cache:

```ruby{3:(24-56)}
  # ...
  has_one_attached :image, dependent: :purge_later

  belongs_to :owner, class_name: "User", counter_cache: true
  # ...
```
{: filename="app/models/photo.rb" }

We specify `class_name: "User"` because the association name `owner` doesn't match the model name `User`. The `counter_cache: true` option is a nice performance optimization — every time a photo is created or destroyed, Rails will automatically increment or decrement the `photos_count` column on the associated User. This means we can display "42 photos" on a user's profile without running a `COUNT(*)` query every time.

### Validations

Add validations for the required fields:

```ruby{3-4}
  # ...
  belongs_to :owner, class_name: "User", counter_cache: true

  validates :caption, presence: true
  validates :image, presence: true
  # ...
```
{: filename="app/models/photo.rb" }

Every photo must have a caption and an image. Simple and essential.

### Scopes

Add three scopes for ordering and filtering:

```ruby{3-5}
  # ...
  validates :image, presence: true

  scope :latest, -> { order(created_at: :desc) }
  scope :pinned, -> { where(pinned: true) }
  scope :unpinned, -> { where(pinned: false) }
end
```
{: filename="app/models/photo.rb" }

Scopes are named queries that you can chain. Instead of writing `Photo.where(pinned: true).order(created_at: :desc)` everywhere, we can write `Photo.pinned.latest`. They make our code more readable and keep query logic in the model where it belongs.

Now would be a good time for a commit:

```
git add -A
git commit -m "Edited Photos migration and configured Photo model"
```

## About the sample data

The starting point includes a pre-written `sample_data` rake task at `lib/tasks/dev.rake`. You don't need to write it; it's already done. Here's what it does at a high level:

- Creates 10 users (Alice through Jack) with emails like `alice@example.com` and the password `appdev`
- Makes some users private (Bob, Carol, Eve, Ivy)
- Attaches specific avatar images from Cloudinary to each user
- Gives Alice a profile banner image
- Creates follow relationships between users (some accepted, some pending)
- Creates 3 photos per user with philosophical captions
- Creates likes and comments from followers
- Uses `User.skip_callback(:create, :before, :set_default_avatar)` to bypass the default avatar callback, since it manually attaches specific avatars for each user

<div class="alert alert-info">

**Important:** `rake sample_data` won't run successfully until the next lesson, when all five models (User, Photo, Like, Comment, FollowRequest) are in place. After completing this lesson, you can still test things by signing up through the browser at `/users/sign_up`, or by creating a user in the Rails console:

```
rails console
User.create(username: "alice", email: "alice@example.com", password: "appdev")
```
</div>

## Verify your progress

At this point, you should have:

1. All gems installed
2. Cloudinary configured with your API credentials
3. Active Storage installed and pointed at Cloudinary
4. Devise installed with `devise_for :users` in your routes
5. A `users` table with citext columns, defaults, and indexes
6. A `photos` table with proper foreign key, defaults, and indexes
7. User and Photo models with associations, validations, and scopes

Try starting your server with `bin/server` and visiting `/users/sign_up`. You should be able to create a new account. If everything is configured correctly, the new user will automatically get a default avatar image uploaded to Cloudinary.

If you can sign up and sign in, you're in great shape. The views won't look like much yet. We'll build those out in later parts.

Now would be a good time for a final commit and push:

```
git add -A
git commit -m "Completed User and Photo models"
git push -u origin HEAD
```

In the next part, we'll generate the remaining models (Likes, Comments, and FollowRequests) and wire up all the associations between them.

---

- Approximately how long (in minutes) did this lesson take you to complete?
{: .free_text_number #time_taken title="Time taken" points="1" answer="any" }

<!--

# List of project specs for AI assistant

require "rails_helper"

RSpec.describe Photo, type: :model do
  describe "has a belongs_to association defined called 'owner' with Class name 'User'", points: 1 do
    it { should belong_to(:owner).class_name("User") }
  end

  describe "has a has_many association defined called 'comments'", points: 1 do
    it { should have_many(:comments) }
  end

  describe "has a has_many association defined called 'likes'", points: 1 do
    it { should have_many(:likes) }
  end

  describe "has a has_many (many-to_many) association defined called 'fans' through 'likes'", points: 1 do
    it { should have_many(:fans).through(:likes) }
  end
end

require "rails_helper"

RSpec.describe User, type: :model do
  describe "has a has_many association defined called 'comments' with Class name 'Comment' and foreign key 'author_id'", points: 1 do
    it { should have_many(:comments).class_name("Comment").with_foreign_key("author_id") }
  end

  describe "has a has_many association defined called 'own_photos' with Class name 'Photo' and foreign key 'owner_id'", points: 1 do
    it { should have_many(:own_photos).class_name("Photo").with_foreign_key("owner_id") }
  end

  describe "has a has_many association defined called 'likes' with Class name 'Like' and foreign key 'fan_id'", points: 1 do
    it { should have_many(:likes).class_name("Like").with_foreign_key("fan_id") }
  end

  describe "has a has_many (many-to_many) association defined called 'liked_photos' through 'likes' and source 'photo'", points: 1 do
    it { should have_many(:liked_photos).through(:likes).source(:photo) }
  end

  describe "has a has_many association defined called 'sent_follow_requests' with Class name 'FollowRequest' and foreign key 'sender_id'", points: 1 do
    it { should have_many(:sent_follow_requests).class_name("FollowRequest").with_foreign_key("sender_id") }
  end

  describe "has a has_many association defined called 'received_follow_requests' with Class name 'FollowRequest' and foreign key 'recipient_id'", points: 1 do
    it { should have_many(:received_follow_requests).class_name("FollowRequest").with_foreign_key("recipient_id") }
  end

  describe "has a has_many association defined called 'accepted_sent_follow_requests' with scope where 'status' is \"accepted\"", points: 1 do
    it { should have_many(:accepted_sent_follow_requests).class_name("FollowRequest").with_foreign_key("sender_id").conditions(status: "accepted") }
  end

  describe "has a has_many association defined called 'accepted_received_follow_requests' with scope where 'status' is \"accepted\"", points: 1 do
    it { should have_many(:accepted_received_follow_requests).class_name("FollowRequest").with_foreign_key("recipient_id").conditions(status: "accepted") }
  end

  describe "has a has_many (many-to_many) association defined called 'followers' through 'accepted_received_follow_requests' and source 'sender'", points: 1 do
    it { should have_many(:followers).through(:accepted_received_follow_requests).source(:sender) }
  end

  describe "has a has_many (many-to_many) association defined called 'leaders' through 'accepted_sent_follow_requests' and source 'recipient'", points: 1 do
    it { should have_many(:leaders).through(:accepted_sent_follow_requests).source(:recipient) }
  end

  describe "has a has_many (many-to_many) association defined called 'feed' through 'leaders' and source 'own_photos'", points: 1 do
    it { should have_many(:feed).through(:leaders).source(:own_photos) }
  end

  describe "has a has_many (many-to_many) association defined called 'discover' through 'leaders' and source 'liked_photos'", points: 1 do
    it { should have_many(:discover).through(:leaders).source(:liked_photos) }
  end
end

-->
