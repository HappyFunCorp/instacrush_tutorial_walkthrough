# Background Jobs

We are currently doing the talking to Instagram inline, which can take upwards of 20 seconds.  We may not realize it since we are caching it, but lets integrate ActiveJob so that the user can go about their day while we make the first connection.

The strategy is to make sure all of our tests pass, then change the `require_fresh_user` before filter action to trigger the refresh and redirect the user to the loading page that we made before.

After that, we can start adding other jobs to start syncing more data.

## Disable unused tests

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

## Uptodate Users

Now lets tests for logged in users with recently updated information.


