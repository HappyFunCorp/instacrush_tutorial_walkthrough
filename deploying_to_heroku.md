# Deploying to Heroku

OK, so we're in a good place now to try and deploy this thing to heroku.  What we need to do is:

- Install the Heroku Toolbelt
- Create the heroku app
- Setup DNS
- Setup and configure the instragram APP
- Setup the heroku database
- Setup and configure Google Analytics

Let's get going on that now!

## Heroku Toolbelt

https://toolbelt.heroku.com

## Create the Heroku App

I'm going to call this app "happyfancyname".  You'll need to pick your own.

Lets type in the command now:
```
$ heroku create happyfancyname
Creating happyfancyname... done, stack is cedar-14
https://happyfancyname.herokuapp.com/ | https://git.heroku.com/happyfancyname.git
Git remote heroku added
```

And we can check to see how git has been setup:

```
$ git remote -v
heroku	https://git.heroku.com/happyfancyname.git (fetch)
heroku	https://git.heroku.com/happyfancyname.git (push)
origin	git@github.com:HappyFunCorp/instacrush_tutorial.git (fetch)
origin	git@github.com:HappyFunCorp/instacrush_tutorial.git (push)
```

This means when we say `git push heroku master` it's going to push our code to `https://git.heroku.com/happyfancyname.git`.

Lets do that now:

```
$ git push heroku master
```


This is going to start the build process.  We can see which version of ruby heroku thinks we are using, as well as watching the bunder process as it goes installing things.  This will also precompile all of our assets.

Once it's done, let's open it up and see what we get:

```
$ heroku open
```

And this should open up your main landing page! 

## Setup DNS

Next, lets setup DNS.  We DNS simple at happyfuncorp.  Since we're going to have this as a subdomain, we want to setup a CNAME to point to our heroku app.

CNAME fancyname.happyfuncorp.com resolves to happyfancyname.herokuapp.com

Lets tell heroku about this domain:

```
combray:instacrush_tutorial wschenk$ heroku domains:add fancyname.happyfuncorp.com
Adding fancyname.happyfuncorp.com to happyfancyname... done
 !    Configure your app's DNS provider to point to the DNS Target happyfancyname.herokuapp.com
 !    For help, see https://devcenter.heroku.com/articles/custom-domains
```

Lets give it a try:

```
$ open http://fancyname.happyfuncorp.com
```

## Instagram Client App

Lets go to  https://instagram.com/developer/clients/register/ and create a new app.

The callback URL should look like:
http://fancyname.happyfuncorp.com/users/auth/instagram/callback

except you should replace fancyname.happyfuncorp.com with your url.  If you didn't setup a DNS entry, make the domain the herokuapp.com domain.

OK, lets now tell our app about the instagram client and secret (replace with your values):

```
$ heroku config:add INSTAGRAM_APP_ID=37f9f7a2...  INSTAGRAM_APP_SECRET=933ede01...
Setting config vars and restarting happyfancyname... done
```

Now when we go to the site, we should be able to connect with instagram. However, once we press Authorize we'll get an error page on our site.

## Create the database

Lets try and run the database migrations now:

```
$ heroku run rake db:migrate
```

When I do this, I get an error that says:

```
StandardError: An error has occurred, this and all later migrations canceled:

PG::UndefinedColumn: ERROR:  column "instagram_medium_id" referenced in foreign key constraint does not exist
: ALTER TABLE "instagram_interactions" ADD CONSTRAINT "fk_rails_e556aee8de"
FOREIGN KEY ("instagram_medium_id")
  REFERENCES "instagram_media" ("id")
```

This is because, a long way back, we forced `instgram_media` to be plural, and not all of the adaptors know about that.  No worries, we'll change the `create_instagram_interactions.rb` migration to create the foreign key manually:

```
class CreateInstagramInteractions < ActiveRecord::Migration
  def change
    create_table :instagram_interactions do |t|
      t.references :instagram_media, index: true
      t.integer :instagram_user_id
      t.string :comment
      t.boolean :is_like

      t.timestamps null: false
    end
    add_index :instagram_interactions, :instagram_user_id
    add_index :instagram_interactions, :is_like
    # This is for pgsql.  Remove foreign_key: true above and replace with this
    add_foreign_key :instagram_interactions, :instagram_media, column: :instagram_media_id
  end
end
```


_how did we test that?  maybe go into installing postgres locally_


I'm also going to add `ruby '2.2.2'` to the Gemfile, since that's what I'm running locally so it makes sense to keep the versions here and on heroku in sync.  

```
$ git commit -a -m "Updated migration and ruby version for deployment"
[master a11b00d] Updated migration and ruby version for deployment
 2 files changed, 3 insertions(+), 2 deletions(-)
$ git push heroku master
```

Now after this deployes, we can try and run the migrations again:

```
$ heroku run rake db:migrate
```

Now that this worked, lets restart everything.

```
$ heroku restart
```

Lets also add a admin user for active admin.  But add something different than that password!

```
$ hero run console
> AdminUser.create!(email: 'admin@example.com', password: 'password', password_confirmation: 'password')
```

## Testing it out

Open up a log in one window:

```
$ heroku logs --tail
```

And then hit the site and try and connect with instagram.  Lets see what happens.  After I connect, I see the job queued but I don't 

