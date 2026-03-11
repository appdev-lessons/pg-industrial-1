# Photogram Industrial: Devise accounts and photos scaffold

## Getting started

This project includes automated tests, so click on this button to get started:

LTI{Load Photogram Industrial assignment}(https://grades.firstdraft.com/launch)[S9ymPy6WCsn18gLbByVbZQ7k]{vfdtzJb5bLYqYwuqgeRKpc5d}(10)[Photogram Industrial Project]

The current project, `photogram-industrial`, covers everything in this lesson and the next three lessons: _Photogram Industrial Parts 2, 3, and 4_. Keep the project open when you move on to those lessons in order to build on your progress through the series.

<div class="alert alert-danger">

You will need to go all the way through the lesson series and implement everything to get the `grade` tests to pass, which all start out failing. **None of the tests will even run until you add all of the models in the first two parts in this series.**

So, don't panic if you see an error message in Grades about tests not being run prior to adding your User, Photo, Like, Comment, and FollowRequest models in the next lesson.

**This first part does not contain a video guide.** Please follow along closely with the written text.
</div>

Here is a rough target to work towards:

[photogram-industrial.matchthetarget.com](https://photogram-industrial.matchthetarget.com/).

This time around, Photogram will be _industrial grade_ — the kind of code you could charge money for. We'll use database indexes and constraints, advanced association accessors, scopes, validations, view helper methods like `link_to` and `form_with` everywhere, partials to DRY up code judiciously, the Devise gem for authentication and password reset emails, Active Storage for real image uploads, and many other industrial-strength upgrades.

This is like finishing school. We're going to learn how to level up to write a codebase that we can onboard professional developers to.

Launch the codespace for your forked project and get the live preview running with `bin/dev`.

Also, go to the settings of your forked repository on `github.com/YOUR_USERNAME/photogram-industrial` and add your instructors as collaborators ("Settings" tab, then "Manage Access").

We're going to start leaving feedback for you in the form of comments on your pull requests. You're going to start adopting the professional git workflow, where you submit pull requests for your branches, and receive line-by-line comments on your code.

[Here is a cheat sheet for our git workflow.](https://learn.firstdraft.com/lessons/196-git-cli)

We're going to practice the workflow for each feature that we're working on of creating a branch, committing to it, and merging it back to `main`.

To remind you, here is the data model from Photogram:

![](/assets/pg-erd.png)

Importantly, there's the `FollowRequest` table, which keeps track of who's following whom. We have a status column in the `FollowRequest` because this is going to be a permissioned social network. When somebody sends a `FollowRequest`, we're going to start it off as "pending", and the recipient of that request has to update that to "accept" it before the follower can actually see their posts.

## User accounts with Devise

Let's begin in our blank app by adding accounts with Devise. Open your `Gemfile` in the root directory and look for the `gem "devise"` line.

If it's not there, add this gem now. Don't put it in one of the `:development` or `:test` group code blocks, put it outside of these blocks. We want the Devise gem available everywhere, not just when we are in development or test environments.

For example, any gems that you put in the `:development` group are just things we use while developing:

```ruby
group :development do
  gem "annotaterb"
  gem "better_errors"
  gem "binding_of_caller"
  gem "pry-rails"
  gem "rails_db"
  gem "rails-erd"
  gem "rufo"
end

gem "devise"
# Remove whitespace from model attributes
gem "strip_attributes"
```
{: filename="Gemfile" }

These are things like our `better_errors` page for debugging, `annotaterb` to add column information on the models, etc. We don't want these gems to be loaded in the `:production` environment when we deploy our app to users. It saves memory not to have these loaded in production. That's why we have these gem `groups`. It allows us to specify gems that we want to use in production versus development versus all of the time.

With the `gem "devise"` line added _outside_ of any group, we can go to a terminal tab and run the usual commands to install gems and Devise.

(Consider [clearing your terminal](https://learn.firstdraft.com/lessons/31#clear-terminal) before you run any of these commands to clear old output, so you can clearly see any instructions or error messages when the command runs.)

```
bundle install
```

Then:

```
rails generate devise:install
```

If you are asked to overwrite the file when you run these commands, then you can say yes (`Y` or `a` for "yes to all").

The `devise:install` command outputs a list of instructions in the terminal that you should follow. One item on the list is defining a root route in your controller. We don't have any resources yet, but soon `users#feed` will work, so add that:

```ruby{2}
Rails.application.routes.draw do
  root "users#feed"
  # ...
```
{: filename="config/routes.rb" }

Once you make all of the Devise changes, you can make a:

  - `git add -A`,
  - then a `git commit -m "install devise and add root"`,
  - and possibly even a `git push`

in succession at the terminal now to save your work on your GitHub repo fork.

We just made that commit and push on the `main` branch! Oops! We want to get in the habit of branching and merging, which is the proper git workflow.

Before we go on, let's make our first git branch to get into the habit of our new workflow before we add anything else to the app.

Create a branch at the terminal bash prompt by running (replace `<your-initials>` with your initials, e.g. `rb-create-database`):

```
git checkout -b <your-initials>-create-database
```

Now we will be switched to our new feature branch to work on, commit to, push to GitHub, and eventually merge to `main`.

We can begin by generating the `users` table at the terminal:

```
rails g devise user username display_name avatar_image profile_banner bio website private:boolean likes_count:integer comments_count:integer photos_count:integer
```

As a reminder:

 - `g` is short for `generate` in the `rails` command above, like `c` is short for `console`.
 - I dropped `:string` after `username`, `display_name`, `avatar_image`, etc. because `string` is the default datatype.

Why not a commit to get things started:

`git add -A`

then:

`git commit -m "generated users with devise"`

## Users migration file

Before we migrate the `users` table to our database, let's open the migration file and explore some values:

```ruby
# frozen_string_literal: true

class DeviseCreateUsers < ActiveRecord::Migration[8.0]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""
# ...
```
{: filename="db/migrate/<date-time-of-migration>_devise_create_users.rb" }

This is the long migration that Devise wrote on our behalf when we generated the `users`. It's adding email and encrypted password columns for us automatically. If you look down the file, there are also columns that it uses internally for handling the forgotten password flow:

```ruby
# ...
      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at
# ...
```

There's even some optional columns (commented out by default) to make the users `# Trackable` `# Confirmable`, and `# Lockable`, very nice!

All the way at the bottom, we can find the columns that _we_ specified in the generation:

```ruby
# ...
      t.string :username
      t.string :display_name
      t.string :avatar_image
      t.string :profile_banner
      t.string :bio
      t.string :website
      t.boolean :private
      t.integer :likes_count
      t.integer :comments_count
      t.integer :photos_count
# ...
```

Before I run this migration, I want to make a couple of changes. First, I want to set default values on the count columns and the `private` column:

```ruby{8:(29-40),9:(32-43),10:(30-41),11:(22-35)}
# ...
      t.string :username
      t.string :display_name
      t.string :avatar_image
      t.string :profile_banner
      t.string :bio
      t.string :website
      t.boolean :private, default: true
      t.integer :likes_count, default: 0
      t.integer :comments_count, default: 0
      t.integer :photos_count, default: 0
# ...
```

Whenever you generate a model or scaffold, it's a good idea to come into the migration file in `db/migrate/` and think about default values on each column. Usually, for numerical columns, like `likes_count` or `comments_count`, a starting value of 0 makes sense (rather than the default of `nil` if we don't set anything when we instantiate a new instance). For the `private` column, we want new accounts to be private by default (`true`).

Another nice thing we can do is shown on these lines at the bottom of the migration file:

```ruby{2,3}
# ...
    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    # add_index :users, :confirmation_token,   unique: true
    # add_index :users, :unlock_token,         unique: true
  end
end
```

Devise added these `add_index ...` lines. This is a very important concept in database design. The index is how we speed up lookups of records. It's just like the index in a long book! Without it, Rails has to scan through every "page" of the database to find the record. With the index, the database creates a separate record keeping area with the indexes that we specify, so we can look up all of those very quickly.

This is important for primary keys, so primary keys are automatically indexed on the database. No need to add any lines to the migration file. If you have any other columns that you plan to look up records by (usually things like `email` or `username`), then it's a good idea to add indexes to those columns (although you can always add them later if you notice slowness). The `email` index was already added for us, but let's add one for our `username`s:

```ruby{6}
# ...
    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    # add_index :users, :confirmation_token,   unique: true
    # add_index :users, :unlock_token,         unique: true
    add_index :users, :username,             unique: true
  end
end
```

In addition, we (and Devise) have used the `unique: true` option for these columns. This will add a _database constraint_ enforcing uniqueness within the column at the database level, which is a stronger guarantee than an `ActiveRecord` validation.

<aside markdown="1">
An `ActiveRecord` model validation still allows for ["race conditions"](https://en.wikipedia.org/wiki/Race_condition). Hence, a database constraint on uniqueness is important to add here.
</aside>

Devise knows that we want both an index and a uniqueness constraint for `email` since that's what we uniquely identify and look up accounts by. In this app, we decided to have a `username` column that we're probably going to be using similarly. Therefore, _we_ (_not_ Devise) had to add an index and a uniqueness constraint for it.

An advanced optimization that we can make, is to use a case-insensitive column for `username`. That way, when doing lookups, we won't have to worry about `RaGhU` not matching `raghu`, or normalizing by downcasing or upcasing before every lookup; the database will take care of it for us. [You can read more here.](https://mikecoutermarsh.com/storing-email-in-postgres-rails-use-citext/)

We can enable this feature by adding a line at the very top of the `change` method in the migration file, then we can change the `email` and `username` columns to use it:

```ruby{5,7:(9-14),12:(9-14)}
# ...
class DeviseCreateUsers < ActiveRecord::Migration[8.0]
  def change
    create_table :users do |t|
      enable_extension("citext")
      ## Database authenticatable
      t.citext :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      # ...

      t.citext :username, null: false
#...
```

Note that we also added `null: false` to `username` to prevent blank usernames at the database level.

This is an example of a database-specific feature. Previously, we used a lightweight database called **SQLite** that did not have `citext` column support. **PostgreSQL**, the professional database that we are using now, has many other excellent features (JSON datatype, range datatype, ordering by geographic distance, full-text search), and Rails provides first-class support for many of them; [see this Rails Guide for a rundown](https://guides.rubyonrails.org/active_record_postgresql.html).

<aside markdown="1">
Pretty much every device in the world has SQLite installed on it, including hardware devices. Probably if you have any smart device in your home, even a light bulb, it has SQLite installed on it. This database is universal, and a nice way for us to get started, but we're upgrading now to PostgreSQL. This is a very powerful database and we're going to use it now in development mode, as well as production, so that we have access to all of the features.
</aside>

When you're satisfied with your migration, `rake db:migrate` and `git commit`, perhaps using our shortcut. The message should be something succinct that describes the incremental change, like "generated users" or "edited and migrated devise users".

## Active Storage

Before we go further, let's set up Active Storage. Active Storage is a built-in Rails framework for uploading files and attaching them to Active Record models. Unlike CarrierWave (which stores filenames as strings in your database columns), Active Storage uses its own set of tables to track file attachments.

Run the following at the terminal to install Active Storage:

```
rails active_storage:install
```

This will create a migration that adds three tables: `active_storage_blobs`, `active_storage_attachments`, and `active_storage_variant_records`. These tables work together to manage file uploads.

Go ahead and migrate:

```
rake db:migrate
```

And commit:

```
git add -A
git commit -m "installed active storage"
```

We'll use Active Storage later when we add `has_one_attached` declarations to our models for handling image uploads. For now, we just need the tables in place.

## Photos resource

Now it's time to start generating the rest of our data model. Typically, I generate my `users` first because I'm going to have a lot of associations between `users` and everything else in almost every application.

Let's revisit the ERD:

![](/assets/pg-erd.png)

The question you have to answer now is: for each of these tables, do you want to generate a `scaffold` or do you just want to generate a `model`? How do we figure that out?

My usual rule of thumb:

 - If I will need routes and controller/actions for users to be able to CRUD records in the table, then I probably want to generate `scaffold`. (At least some of the generated routes/actions/views will go unused. I need to remember to go back and at least disable the routes, and eventually delete the entire RCAVs, at some point; or I risk introducing security holes.)
 - If the model will only be used on the backend, e.g. by other models, then I probably want to generate `model`. For example, a `Month` model where I will create all twelve records once in `db/seeds.rb` does not require routes, `MonthsController`, `app/views/months/`, etc.

In this case, since users will be CRUDing all of the remaining resources, we'll `scaffold` them all.

Let's generate the photos resource first:

```
rails g scaffold photo image comments_count:integer likes_count:integer caption:text owner:references pinned:boolean
```

Notice that I used `owner:references` as the foreign key column name and datatype, instead of what you might have been expecting, `owner_id:integer`. (An alias for `owner:references` is `owner:belongs_to`. They mean the same thing.) We've also added a `pinned:boolean` column so that users can pin important photos to the top of their profile.

Go take a look at the generated migration file. First, be sure to add some default values to the `_count` columns and the `pinned` column:

```ruby{5:(32-43),6:(29-40),8,9:(20-48)}
class CreatePhotos < ActiveRecord::Migration[8.0]
  def change
    create_table :photos do |t|
      t.string :image
      t.integer :comments_count, default: 0
      t.integer :likes_count, default: 0
      t.text :caption
      t.references :owner, null: false, foreign_key: true
      t.boolean :pinned, default: false, null: false

      t.timestamps
    end
  end
end
```
{: filename="db/migrate/<date-time-of-migration>_create_photos.rb" }

If we ran this migration as-is,

 - Even though it says `t.references` instead of the usual `t.integer`, the datatype would be `integer` (or whatever the default datatype is for primary keys for the database you are using; [I commonly use UUIDs these days](https://pawelurbanek.com/uuid-order-rails)).
 - The column name would be `owner_id` rather than `owner`, since `t.references` knows the convention we want to follow.
 - A database constraint would be added preventing the column from being blank. If you want to allow this foreign key column to be blank, which is sometimes the case, then you should delete the `null: false` option.
 - We _don't_ need to add an `index: true` option to the `owner_id` column, because Rails adds this lookup index by default. That's good, because we'll often look up photos by their `owner_id`, or filter the photo table by `owner_id`.

Go ahead and try to `rake db:migrate` now.

Uhoh! Can you spot the helpful error message?

```
... relation "owners" does not exist
```

That's because we departed from conventional naming here! Our table that we are associating with `photos` is `users`, but we are associating it as `owner`.

If you head over to `app/models/photo.rb`, you'll notice that a `belongs_to :owner` association accessor was automatically added:

```ruby{2}
class Photo < ApplicationRecord
  belongs_to :owner
end
```
{: filename="app/models/photo.rb" }

That association isn't quite right, is it? Because the other model name is `User`, not `Owner`; we just chose to use a more descriptive foreign key column name than `user_id`.

So, the generator tried to be helpful, but couldn't know that we went off-convention with our foreign key column name. Update the association accessor to be correct:

```ruby{2:(20-39)}
class Photo < ApplicationRecord
  belongs_to :owner, class_name: "User"

  validates :caption, presence: true
  validates :image, presence: true
end
```
{: filename="app/models/photo.rb" }

And while we're at it with association accessors, we should add the `has_many` side to the `User` model (noting all the nice Devise additions):

```ruby{7}
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  has_many :own_photos, class_name: "Photo", foreign_key: "owner_id"
end
```
{: filename="app/models/user.rb" }

Similarly, we need to update the migration file to point the foreign key to the correct table:

```ruby{4:(41-73)}
class CreatePhotos < ActiveRecord::Migration[8.0]
  # ...
      t.text :caption
      t.references :owner, null: false, foreign_key: { to_table: :users }
  # ...
```
{: filename="db/migrate/<date-time-of-migration>_create_photos.rb" }

The `to_table:` key in the hash allows us to supply the table name that `owner` should point to.

When you're satisfied, `rake db:migrate`. Then commit with a `git add -A; git commit -m "Generated photos"` at the terminal. And you could even push the changes up to your repo with a `git push`.

<div class="alert alert-danger">

You will need to go all the way through the lesson series and implement everything to get the `grade` tests to pass, which all start out failing. **None of the tests will even run until you add all of the models in the first two parts in this series.**

So, don't panic if you see an error message in Grades about tests not being run prior to adding your User, Photo, Like, Comment, and FollowRequest models in the next lesson.
</div>

- Approximately how long (in minutes) did this lesson take you to complete?
{: .free_text_number #time_taken title="Time taken" points="1" answer="any" }

<!--

# List of project specs for AI assistant

require "rails_helper"

describe "/[USERNAME]/discover" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}/discover"

    expect(page.status_code).to be(200)
  end

  it "shows photos liked by people the current user follows", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev")
    owner = User.create(username: "owner", email: "owner@example.com", password: "appdev", private: false)
    photo = create_photo(owner: owner, caption: "owner caption")
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    Like.create(fan_id: leader.id, photo_id: photo.id)

    visit "/#{user.username}/discover"

    expect(page).to have_content(photo.caption)
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "/[USERNAME]/feed" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}/feed"

    expect(page.status_code).to be(200)
  end

  it "shows their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader, caption: "leader caption")
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    expect(page).to have_content(photo.caption)
    expect(page).to have_css("img")
  end

  it "allows them to like their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    click_on "0 likes"

    expect(page).to have_css("i.fa-solid.fa-heart")
  end

  it "allows them to un-like their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    Like.create(fan_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    click_on "1 like"

    expect(page).to have_css("i.fa-regular.fa-heart")
  end

  it "allows the user to add a comment on their leader's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")

    visit "/#{user.username}/feed"

    fill_in "comment[body]", with: "New comment"
    click_button "Create Comment"

    expect(page).to have_content("New comment")
  end

  it "allows the user to delete their comment", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    comment = Comment.create(body: "New comment", author_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    within("#comment_#{comment.id}") do
      click_on "Delete"
    end

    expect(page).not_to have_content("New comment")
  end

  it "allows the user to edit their comment", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    leader = User.create(username: "leader", email: "leader@example.com", password: "appdev", private: false)
    photo = create_photo(owner: leader)
    FollowRequest.create(sender_id: user.id, recipient_id: leader.id, status: "accepted")
    comment = Comment.create(body: "New comment", author_id: user.id, photo_id: photo.id)

    visit "/#{user.username}/feed"

    within("#comment_#{comment.id}") do
      click_on "Edit"
    end

    fill_in "comment[body]", with: "Edited comment"
    click_button "Update Comment"

    expect(page).to have_content("Edited comment")
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "/" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page.status_code).to be(200)
  end

  it "has a bootstrap navbar", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_tag("nav", with: { class: "navbar" })
  end

  it "has a Settings link for the signed in user", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_link("Settings", href: "/users/edit")
  end

  it "does not have a sign in link if the user is already signed in", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to_not have_link("Sign in", href: "/users/sign_in")
  end

  it "has a link, 'Feed', that navigates to the 'Feed' page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Feed"

    expect(page).to have_current_path("/#{user.username}/feed")
  end

  it "has a link, 'Discover', that navigates to the 'Discover' page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Discover"

    expect(page).to have_current_path("/#{user.username}/discover")
  end

  it "has a link, 'Go to profile', that navigates to the profile page", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    click_on "Go to profile"

    expect(page).to have_current_path("/#{user.username}")
  end

  it "has an 'Add photo' button", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/"

    expect(page).to have_button("Add photo")
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

require "rails_helper"

describe "/photos/new" do
  it "has a form to add a new photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/photos/new"

    expect(page).to have_form("/photos", :post)
  end

  it "does not allow the user to add a new photo without a caption", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/photos/new"

    all("input[type='file']").last.attach_file("#{Rails.root}/spec/support/test_image.jpeg")
    all("input[type='submit']").last.click

    expect(page).to have_content("Caption can't be blank")
  end

  it "allows the user to add a new photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/photos/new"

    all("input[type='file']").last.attach_file("#{Rails.root}/spec/support/test_image.jpeg")
    all("textarea").last.fill_in(with: "caption")
    all("input[type='submit']").last.click

    expect(page).to have_content("Photo was successfully created")
  end

  it "redirects to the photo details page after creating a new photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/photos/new"

    all("input[type='file']").last.attach_file("#{Rails.root}/spec/support/test_image.jpeg")
    all("textarea").last.fill_in(with: "caption")
    all("input[type='submit']").last.click

    expect(page).to have_current_path("/photos/#{Photo.last.id}")
  end
end

describe "/photos/[ID]" do
  it "displays the photo and caption", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")

    visit "/photos/#{photo.id}"

    expect(page).to have_css("img")
    expect(page).to have_content(photo.caption)
  end

  it "allows the user to edit the photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")

    visit "/photos/#{photo.id}"

    click_on "Edit"

    all("textarea").last.fill_in(with: "new caption")
    all("input[type='submit']").last.click

    expect(page).to have_content("new caption")
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "/[USERNAME]" do
  it "can be visited", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}"

    expect(page.status_code).to be(200)
  end

  it "has a Posts tab", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}"

    expect(page).to have_button("Posts")
  end

  it "has a Likes tab", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    visit "/#{user.username}"

    expect(page).to have_button("Likes")
  end

  it "displays each of the user's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")

    visit "/#{user.username}"

    expect(page).to have_css("img")
    expect(page).to have_content(photo.caption)
  end

  it "shows the comments on the user's photos", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")
    comment = Comment.create(body: "comment body", author_id: user.id, photo_id: photo.id)

    visit "/#{user.username}"

    expect(page).to have_content(comment.body)
  end

  it "allows the user to delete their photo", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    photo = create_photo(owner: user, caption: "caption")

    visit "/#{user.username}"

    click_on "Delete"

    expect(page).not_to have_content(photo.caption)
  end

  it "shows a list of followers on the user profile", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: other_user.id, recipient_id: user.id, status: "accepted")

    visit "/#{user.username}"

    click_on "followers"

    expect(page).to have_content(other_user.username)
  end

  it "shows a list of leaders on the user profile", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: user.id, recipient_id: other_user.id, status: "accepted")

    visit "/#{user.username}"

    click_on "following"

    expect(page).to have_content(other_user.username)
  end

  it "shows a 'Following' button for leaders", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: user.id, recipient_id: other_user.id, status: "accepted")

    visit "/#{other_user.username}"

    expect(page).to have_button("Following")
  end

  it "shows pending follow requests for private accounts", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    private_user = User.create(username: "private_user", email: "private_user@example.com", password: "appdev", private: true)

    visit "/#{private_user.username}"

    click_on "Follow"

    expect(page).to have_button("Requested")
  end

  it "allows a user to unfollow another user", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: user.id, recipient_id: other_user.id, status: "accepted")

    visit "/#{other_user.username}"

    click_on "Following"

    expect(page).to have_button("Follow")
  end

  it "allows a user to cancel pending follow request", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    private_user = User.create(username: "private_user", email: "private_user@example.com", password: "appdev", private: true)
    FollowRequest.create(sender_id: user.id, recipient_id: private_user.id, status: "pending")

    visit "/#{private_user.username}"

    click_on "Requested"

    expect(page).to have_button("Follow")
  end

  it "allows a user to accept a follow request", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: other_user.id, recipient_id: user.id, status: "pending")

    visit "/#{user.username}/pending"

    click_on "Accept"

    expect(page).not_to have_content(other_user.username)
  end

  it "allows a user to reject a follow request", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")
    sign_in(user)

    other_user = User.create(username: "other_user", email: "other_user@example.com", password: "appdev")
    FollowRequest.create(sender_id: other_user.id, recipient_id: user.id, status: "pending")

    visit "/#{user.username}/pending"

    click_on "Reject"

    expect(page).not_to have_content(other_user.username)
  end
end

def sign_in(user)
  visit "/users/sign_in"

  fill_in "Email", with: user.email
  fill_in "Password", with: user.password
  click_button "Sign in"
end

def create_photo(owner:, caption: "caption")
  photo = Photo.new(caption: caption, owner_id: owner.id)
  photo.image.attach(io: File.open(Rails.root.join("spec/support/test_image.jpeg")), filename: "test_image.jpeg", content_type: "image/jpeg")
  photo.save!
  photo
end

require "rails_helper"

describe "User authentication" do
  it "displays a banner to sign in when trying to visit the homepage", points: 1 do
    visit "/"

    expect(page).to have_content("You need to sign in or sign up before continuing")
  end

  it "sends the user to the sign in page when trying to visit the homepage", points: 1 do
    visit "/"

    expect(page).to have_current_path("/users/sign_in")
  end

  it "allows new user sign ups", points: 1 do
    visit "/users/sign_up"

    fill_in "Email", with: "alice@example.com"
    fill_in "Password", with: "appdev"
    fill_in "Password confirmation", with: "appdev"
    fill_in "Username", with: "alice"
    click_button "Sign up"

    expect(page).to have_content("Welcome! You have signed up successfully")
  end

  it "allows an existing user to sign in", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")

    visit "/users/sign_in"

    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Sign in"

    expect(page).to have_content("Signed in successfully")
  end

  it "allows a user to sign out", points: 1 do
    user = User.create(username: "alice", email: "alice@example.com", password: "appdev")

    visit "/users/sign_in"

    fill_in "Email", with: user.email
    fill_in "Password", with: user.password
    click_button "Sign in"

    click_on user.username
    click_on "Sign out"

    expect(page).to have_current_path("/users/sign_in")
  end
end

require "rails_helper"

RSpec.describe Comment, type: :model do
  describe "has a belongs_to association defined called 'author' with Class name 'User'", points: 1 do
    it { should belong_to(:author).class_name("User") }
  end

  describe "has a belongs_to association defined called 'photo'", points: 1 do
    it { should belong_to(:photo) }
  end
end

require "rails_helper"

RSpec.describe FollowRequest, type: :model do
  describe "has a belongs_to association defined called 'sender' with Class name 'User'", points: 1 do
    it { should belong_to(:sender).class_name("User") }
  end

  describe "has a belongs_to association defined called 'recipient' with Class name 'User'", points: 1 do
    it { should belong_to(:recipient).class_name("User") }
  end
end

require "rails_helper"

RSpec.describe Like, type: :model do
  describe "has a belongs_to association defined called 'fan' with Class name 'User'", points: 1 do
    it { should belong_to(:fan).class_name("User") }
  end
end

RSpec.describe Like, type: :model do
  describe "has a belongs_to association defined called 'photo'", points: 1 do
    it { should belong_to(:photo) }
  end
end

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
