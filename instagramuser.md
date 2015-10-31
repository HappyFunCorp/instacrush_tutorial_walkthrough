## InstagramUser

Lets start with the main user model.  We’ll want to associate it with the current user, add a last synced time, and throw in some other meta data that we’ll probably need:

```
$ rails g scaffold instagram_user user:references last_synced:datetime username:string:index full_name:string profile_picture:string media_count:integer followed_count:integer following_count:integer
```

And create the database and start running tests.  We’ll be flipping back a forth between the `rails console` and editing the spec files.

```
$ rake db:migrate && guard
```

It says we have 2 pending tests to write, one for `InstagramUser` and one for `User`.  Lets start with InstagramUser.

`instagram_user_spec.rb` : lets work on the basic path.

```
require 'rails_helper'

INSTAGRAM_ACCESS_TOKEN="token"

RSpec.describe InstagramUser, type: :model do
  let( :user ) { create( :user ) }
  let( :instagram_auth ) { create( :identity, provider: :instagram, user: user, accesstoken: INSTAGRAM_ACCESS_TOKEN ) }

  it "should sync the users info" do
    expect( instagram_auth ).to_not be_nil
    expect( InstagramUser.count ).to eq( 0 )

    InstagramUser.sync_from_user( user )

    iu = InstagramUser.where( user_id: user.id ).first
    expect( iu ).to_not be_nil
    expect( iu.username ).to eq( "wschenk" )
  end
end

```

Now lets start to implement it:

```
class InstagramUser < ActiveRecord::Base
  def self.sync_from_user( user )
    user_info = user.instagram_client.user

    p user_info
  end
end
```

All we are doing here is to pull down the data and see what gets returned.

When we look at the test results, we get a `VCR::Errors::UnhandledHTTPRequestError` that says:

```
 An HTTP request has been made that VCR does not know how to handle:
    GET https://api.instagram.com/v1/users/self.json?access_token=token
```

There are two problems.  One is that the token we are trying to use is not valid, and the second is that we want to record the response from the instagram API so that we don’t always hit the server when running tests.

We have a token in the database, lets pull that from the console now:

```
[1] instacrush »  Identity.first.accesstoken
=> "509161.38c3f84.....
```

Copy that value into the `INSTAGRAM_ACCESS_TOKEN` constant.  Second, we need to wrap the call to `sync` in a `VCR.use_cassette` block, which will let the request go through the first time and then play it back subsequent times.

```
    VCR.use_cassette( "instagram/user_sync" ) do
      InstagramUser.sync( user )
    end
```


Now when I run my tests I get the error:

```
InstagramUser
#<Hashie::Mash bio="" counts=#<Hashie::Mash followed_by=190 follows=224 media=678> full_name="Will Schenk" id="509161" profile_picture="http://scontent.cdninstagram.com/hphotos-xaf1/t51.2885-19/11374591_955491831180543_1527410442_a.jpg" username="wschenk" website="http://willschenk.com">
  should sync the users info (FAILED - 1)
```

(You should get your instagram user instead of mine.)

### Populating the values

Lets add some more conditions to `instagram_user_spec`.  You’ll need to adjust these values based upon what was returned from instagram.  Since we’re playing back the requests (using `VCR.use_cassette`) these values won’t change again.

```
  it "should sync the users info" do
    expect( instagram_auth ).to_not be_nil
    expect( InstagramUser.count ).to eq( 0 )

    VCR.use_cassette( "instagram/user_sync" ) do
      InstagramUser.sync_from_user( user )
    end

    iu = InstagramUser.where( user_id: user.id ).first

    expect( iu ).to_not be_nil
    expect( iu.username ).to eq( "wschenk" )
    expect( iu.full_name ).to eq( "Will Schenk" )
    expect( iu.profile_picture ).to eq( "http://scontent.cdninstagram.com/hphotos-xaf1/t51.2885-19/11374591_955491831180543_1527410442_a.jpg" )
    expect( iu.media_count ).to eq( 678 )
    expect( iu.followed_count ).to eq( 190 )
    expect( iu.following_count ).to eq( 224 )
  end
```

Lets write code so this will pass.  Obviously you’ll need to change the data in the test to make sure that it reflects what the API returns for your user:

```
  def self.sync_from_user( user )
    user_info = user.instagram_client.user

    u = where( :username => user_info['username']).first_or_create
    u.user_id = user.id
    u.username = user_info['username']
    u.full_name = user_info['full_name']
    u.profile_picture = user_info['profile_picture']
    if user_info['counts']
      u.media_count = user_info['counts']['media']
      u.followed_count = user_info['counts']['followed_by']
      u.following_count = user_info['counts']['follows']
    end
    
    u.save

    u
  end
```

OK, this is nice and easy.  We are using rail’s `first_or_create` method, which looks first in the database to see if the record is already there and will only create it if needed.  You can pass this method a block which will only be called if the record is new, however, we always want our data up-to-date so we will override with the results.

We are looking up `InstagramPost` using the `username` attribute, which is why we put an index on it before.

Let’s get into loading up the rest of the data!

## Check in the code

```
$ git add .
$ git commit -a -m "InstagramUser scaffold and basic spec"
```

(Github link: https://github.com/HappyFunCorp/instacrush_tutorial/commit/24acbcb49aee4dc92b4b67608b8ba9fbd600b804)