# Making the URLs friendly

Right now when we look at crushes, or users, we see the internal ID of our server.  That's sort of ugly.  Lets change the URLS using the friendly_id gem to make things cleaner.

First, add the gem inside of `Gemfile`

```
gem 'friendly_id'
```

Then run 

```
$ bundle
```

To install the gem.

## Fixing up crushes
The first thing we are going to do is to edit `crush.rb`, to include the new plugin.

```
class Crush < ActiveRecord::Base
  extend FriendlyId
  friendly_id :slug, use: :slugged
```

And then we need to actually create the slug value.  Inside of the `find_for_user` method, lets add that line now:

```
    c.likes_count = likes
    c.comments_count = interactions - likes
    c.slug = "#{c.instagram_user.username} #{c.crush_user.username}".hash.abs.to_s(36)
```

Lets add the route now, to say that when generating a link to a crush, instead of using the default `:id` parameter we'll use `:slug` instead.  This goes in `routes.rb`

```
  resources :crush, only: [:index, :show], param: :slug do
    collection do
      get 'loading'
    end
  end
```

Finally, we need to tweak the `crush_controller.rb` to do the query based on the correct parameter:

```
  def show
    @crush = Crush.find_by_slug( params[:slug] )
  end
```

## InstagramUser

`instagram_user.rb`: 

```
class InstagramUser < ActiveRecord::Base
  extend FriendlyId
  friendly_id :username, use: :slugged
  
  def slug
    username
  end

```

`routes.rb`:

```
  resources :instagram_users, only: [:index, :show], path: 'instagram/users', param: :username
```

`instagram_users_controller.rb`:

```
  def show
    @user = InstagramUser.findby_username( params[:username] )
...
```