# Background Jobs

We are currently doing the talking to Instagram inline, which can take upwards of 20 seconds.  We may not realize it since we are caching it, but lets integrate ActiveJob so that the user can go about their day while we make the first connection.

The strategy is to make sure all of our tests pass, then change the `require_fresh_user` before filter action to trigger the refresh and redirect the user to the loading page that we made before.

After that, we can start adding other jobs to start syncing more data.

## Installing ActiveJob

ActiveJob is to running background jobs as ActiveRecord is to querying the database.  It provides a common interface that lets us swap out a different runtime as needed.  Lets first add active_job and setup some tests to make sure things get queued up directly.  Then we can experiment with the frontend to experience what the difference it.

Now lets create a new job:

```
$ rails g job UpdateUserFeed
      invoke  rspec
      create    spec/jobs/update_user_feed_job_spec.rb
      create  app/jobs/update_user_feed_job.rb
```

This creates a few files that we will be editing shortly. Finally, lets tell rspec about ActiveJob, by putting the following in the config block of `spec/rails_helper.rb`:

```
  config.include ActiveJob::TestHelper
```

Lets start specing this stuff out.


## Writing Tests

Lets run guard now:

```
$ rake db:migrate && guard
```

We aren't using the user system at the moment, so lets skip those steps.  In `spec/features/forgot_password_spec.rb`:

```
feature "ForgotPasswords", :type => :feature do
  before { skip "No account system yet" }
```


And in `spec/features/registration_spec.rb`:

```
feature "Registrations", :type => :feature do
  before { skip "No account system yet" }
```

And we've removed the `instagram_interactions` routes all together, so we can delete that test:

```
$ git rm spec/requests/instagram_interactions_spec.rb
```

Finally, for both `spec/requests/instagram_media_spec.rb` and `spec/requests/instagram_user_spec.rb` we are expecting a redirect to login, rather than a 200 response, so we can change the 200 to 302 like so:

```
expect(response).to have_http_status(302)
```

## Writing the job code

`spec/jobs/update_user_feed_spec.rb`:

```
require 'rails_helper'

RSpec.describe UpdateUserFeedJob, type: :job do
  let( :user ) { create( :user ) }
  let!( :instagram_auth ) { create( :identity, provider: :instagram, user: user, accesstoken: INSTAGRAM_ACCESS_TOKEN ) }
  let!( :instagram_user ) { create( :instagram_user, username: 'wschenk', user: user ) }
 
  it "should create a job when requesting a synched user" do
    assert_enqueued_with( job: UpdateUserFeedJob ) do
      instagram_user.sync_if_needed
    end

    expect( instagram_user.state ).to eq( 'queued' )
  end
  
  it "should not double enqueue a job" do
    assert_enqueued_with( job: UpdateUserFeedJob ) do
      expect( instagram_user.sync_if_needed ).to be_truthy
    end

    expect( instagram_user.state ).to eq( 'queued' )

    assert_no_enqueued_jobs do
      expect( instagram_user.sync_if_needed ).to be_falsey
    end
  end

  it "should actually sync the users feed" do
    expect( InstagramMedia.count ).to eq( 0 )

    VCR.use_cassette 'instagram/recent_feed_for_user' do
      UpdateUserFeedJob.perform_now( instagram_user.id )
    end
    
    instagram_user.reload

    expect( instagram_user.stale? ).to be_falsey
    expect( instagram_user.state ).to eq( "synced" )
    expect( InstagramMedia.count ).to_not eq(0)
  end
end
```

Running this gives us an error for `sync_if_needed`.  Lets go into `spec/models/instagram_user_spec.rb` and write some logic for that:

```
  context "needing syncing" do
    it "a new object should be stale" do
      iu = create( :instagram_user )
      expect( iu.stale? ).to be_truthy
    end

    it "an object synced more than 12 hours ago should be stale" do
      iu = create( :instagram_user, last_synced: 13.hours.ago )
      expect( iu.stale? ).to be_truthy
    end

    it "an object synced 1 hours ago should not be stale" do
      iu = create( :instagram_user, last_synced: 1.hour.ago )
      expect( iu.stale? ).to be_falsey
    end
  end
```

Lets start making these tests pass!

`app/models/instagram_user.rb`:

```
  def stale?
    self.last_synced.nil? || self.last_synced < 12.hours.ago
  end

  def sync_if_needed
    if stale? && state != "queued"
      update_attribute( :state, "queued" )
      UpdateUserFeedJob.perform_later( self.id )
    end
  end
```

This makes our `InstagramUser` tests green, but our  UpdateUserFeedJob tests fil with `undefined method 'state=' for #<InstagramUser>`!  Lets add that now:

```
$ rails g migration add_state_to_instagram_user
```

And add our attribute:

```
class AddStateToInstagramUser < ActiveRecord::Migration
  def change
    add_column :instagram_users, :state, :string
  end
end
```

Lets restart guard now, and run the migrations:

```
$ rake db:migrate && guard
```

Looks like things are working now!

## Implement the worker

Lets setup the `app/jobs/update_user_feed_job.rb` worker:

```
class UpdateUserFeedJob < ActiveJob::Base
  queue_as :default

  def perform( instagram_user_id )
    instagram_user = InstagramUser.find instagram_user_id

    InstagramMedia.recent_feed_for_user( instagram_user.user )

    instagram_user.state = "synced"
    instagram_user.last_synced = Time.now
    instagram_user.save
  end
end
```

At this point, all of our specs should be green.

## Testing out the crush controller

Lets first start by looking at what anonymous users see when they get the page.

- index should require them to login
- loading should require them to login
- showing a crush should display the crush

We are using the `let!` rspec method to force some objects to be instantiated, which in turn reference others defined with `let` and bring them back into place.

```
require 'rails_helper'

RSpec.describe CrushController, :type => :controller do
  let!( :instagram_auth ) { create( :identity, provider: :instagram, user: user, accesstoken: INSTAGRAM_ACCESS_TOKEN ) }
  let!( :crush ) { create( :crush, instagram_user: instagram_user, crush_user: crush_user ) }

  let( :user ) { create( :user ) }
  let( :instagram_user ) { create( :instagram_user, user: user ) }
  let( :crush_user ) { create( :instagram_user, user: user ) }

  context "anonymous user" do
    before( :each ) do
      allow_message_expectations_on_nil
      login_with nil
    end

    it "index should redirect to login with instagram" do
      get :index
      expect( response ).to redirect_to( user_omniauth_authorize_path( :instagram ) )
    end

    it "loading should redirect to login with instagram" do
      get :loading
      expect( response ).to redirect_to( user_omniauth_authorize_path( :instagram ) )
    end

    it "should display a public crush" do
      get :show, slug: crush.slug
      expect( response ).to have_http_status(200)
    end
  end
 end
```

Running this we see that we are unfairly forcing anonymous users to login to see someone elses crush.  Lets change that in `crush_controller.rb` now:

```
class CrushController < ApplicationController
  before_filter :require_instagram_user, except: [:show]
  before_filter :require_fresh_user, except: [:show]
```

All tests should pass!

## Up-to-date users

Now lets tests for logged in users with recently updated information.

The trickiest thing here to so setup all of the variables we need to calculate the crush.  We first need to make sure that there's an interaction in the database, since the system checks to see if anything has changed before redirecting them to the crush, so the data needs to be uptodate.

Then we updated the `:last_synced` attribute to an hour ago, login, and make sure that we have the properly redirects.

```
  let!( :instagram_interaction ) { create( :instagram_interaction, instagram_user: crush_user, instagram_media: post ) }
  let( :post ) { create( :instagram_medium, instagram_user: instagram_user )}

  context "up-to-date user" do
    before( :each ) do
      instagram_user.update_attribute( :last_synced, 1.hour.ago )
      instagram_user.update_attribute( :state, 'synced' )
      user.reload
      allow_message_expectations_on_nil
      login_with user
    end

    it "index should redirect to their crush" do
      get :index
      crush.reload # slug should get updated
      expect( response ).to redirect_to( crush_path( crush.slug ) )
    end

    it "loading should redirect to their crush" do
      get :loading
      crush.reload # slug should get updated
      expect( response ).to redirect_to( crush_path( crush ) )
    end

    it "should display their crush" do
      get :show, slug: crush.slug
      expect( response ).to have_http_status( 200 )
    end
  end
```

Running this, the loading test fails since it simply returns 200.  Lets put in code for the stale users, and start adding the queueing system.  We'll circle back to that soon.

## Full on stale users

```
  context "stale user" do
    before( :each ) do
      allow_message_expectations_on_nil
      login_with user
    end

    it "index should redirect to their crush" do
      assert_enqueued_with( job: UpdateUserFeedJob ) do
        get :index
      end
      expect( response ).to redirect_to( loading_crush_index_path )
    end

    it "loading should remain on loading if it's not loaded yet" do
      get :loading
      expect( response ).to have_http_status( 200 )
    end

    it "loading should redirect to their crush if it's been loaded" do
      get :loading
      expect( response ).to redirect_to( loading_crush_index_path )

      instagram_user.update_attribute( :last_synced, 1.hour.ago )
      instagram_user.update_attribute( :state, "synced" )
      user.reload
      get :loading
      crush.reload
      expect( response ).to redirect_to( crush_path( crush ) )
    end

    it "should display their crush" do
      get :show, slug: crush.slug
      expect( response ).to have_http_status( 200 )
    end
  end      
```

OK, we should get some test failures now, 
-  CrushController up-to-date user loading should redirect to their crush
-  CrushController stale user index should redirect to their crush
-  CrushController stale user loading should remain on loading if it's not loaded yet
-  CrushController stale user loading should redirect to their crush if it's been loaded

## Lets make it green

Lets now change our before filter:

```
  def require_fresh_user
    iu = current_user.instagram_user
    if iu.stale?
      iu.sync_if_needed
      redirect_to loading_crush_index_path, notice: "We're talking with instagram right now"
    end
  end
```

And then in `crush_controller.rb`, lets flesh out that `loading` action:

```
  def loading
    if current_user.instagram_user.state == "synced"
      redirect_to Crush.find_for_user( current_user )
    end      
  end
```

## Marking our progress

We've split out the main filtering into a job, gotten the tests organized.  Right now we don't have a job runner installed, and so rails defaults to running the job inline.  If you fire up your server again you will still see the delay the first time you hit the site.

Lets lock down our progress here and we'll start to expiriment with different job runners next.

```
$ git add .
$ git commit -a -m "Added UpdateUserFeedJob"
```

(Github link: https://github.com/HappyFunCorp/instacrush_tutorial/commit/cf15329b9f0041f81b37a89ec5eabc5d7ac7cf0c)