## InstagramMedia & InstagramInteraction

We’re going to implement these things together, since the interactions and the media are so closely related.  We’re going to scaffold out `InstagramMedia`, but we need to pass in `--force-plural` to make sure that it does go all latin on us and call it `InstagramMedium`.

I’m adding a few references attributes, on `instagram_media` to `instagram_user`, and on `instagram_interaction` to `instagram_media`.  This will take care of setting up the `belongs_to` inside of the active record model as well as setting up the indexes and foreign key relationships in the database.  I’m expecting that I’m going to query on `is_like` so I’m adding an index manually there, just in case.

```
$ rails g scaffold instagram_media instagram_user:references media_id:string media_type:string comments_count:integer likes_count:integer link:string thumbnail_url:string standard_url:string --force-plural

$ rails g scaffold instagram_interaction instagram_media:references instagram_user_id:integer:index comment:string is_like:boolean:index
```

Lets go the test window you have now, kill guard and rerun it with migrations:

```
$ rake db:migrate && guard
```

There are two things that we need to test here.  First this is to sync the users feed, which we’ll run off of the `InstagramUser` object.  The next thing is to pull in the likes for the posts themselves, when we know that there are more interactions 

## First lets try syncing the users feed

Lets add a method `InstagramMedia.recent_feed_for_user` which will pull the data from instagram.  Here’s a basic test for `instagram_media_spec.rb`:

```
require 'rails_helper'

RSpec.describe InstagramMedia, type: :model do
  let( :user ) { create( :user ) }
  let( :instagram_auth ) { create( :identity, provider: :instagram, user: user, accesstoken: INSTAGRAM_ACCESS_TOKEN ) }

  before( :each ) do 
    expect( instagram_auth ).to_not be_nil
  end

  it "should load in media objects for the user" do
    expect( InstagramMedia.count ).to eq( 0 )

    VCR.use_cassette 'instagram/recent_feed_for_user' do
      InstagramMedia.recent_feed_for_user( user )
    end

    expect( InstagramMedia.count ).to_not eq(0)
  end

  it "should add data to the media object" do
    VCR.use_cassette 'instagram/recent_feed_for_user' do
      InstagramMedia.recent_feed_for_user( user )
    end

    media = InstagramMedia.all.first

    expect( media ).to_not be_nil
    expect( media.media_id ).to eq( "1103152031012554955_509161" )
    expect( media.media_type ).to eq( "image" )
    expect( media.comments_count ).to eq( 0 )
    expect( media.likes_count ).to eq( 18 )
  end
```

Now lets mock out something quickly so we can double check the data that we get back:

```
class InstagramMedia < ActiveRecord::Base
  belongs_to :instagram_user

  def self.recent_feed_for_user( user )
    user.instagram_client.user_recent_media.each do |media|
      require 'pp'
      pp media
    end
  end
end
```

The tests fail, but we can look at the data that we can back and start wiring it up.  Since we are using VCR, you’ll notice that the first time this test runs it takes a while because of querying instagram, and now we have all the data locally so it’s much faster.

```
class InstagramMedia < ActiveRecord::Base
  belongs_to :instagram_user

  def self.recent_feed_for_user( user )
    user.instagram_client.user_recent_media.each do |media|
      InstagramMedia.reify( media )
    end
  end

  def self.reify( media )
    p = where( media_id: media['id'] ).first_or_create
    p.media_type = media['type']
    p.link = media['link']
    p.thumbnail_url = media['images']['thumbnail']['url']
    p.standard_url = media['images']['standard_resolution']['url']
    p.comments_count = media['comments']['count']
    p.likes_count = media['likes']['count']
    p.created_at = Time.at( media['created_time'].to_i )

    p.save
  end
end
```

And now our (very basic) tests pass.  Lets add a test to make sure that each post as an associated `InstagramUser` object associated with it:

```
  it "each post should have an associated instagram user" do
    VCR.use_cassette 'instagram/recent_feed_for_user' do
      InstagramMedia.recent_feed_for_user( user )
    end

    InstagramMedia.all.each do |a|
      expect( a.instagram_user ).to_not be_nil
    end
  end

```

And add the associated code to `InstagramMedia`:

```
  def self.recent_feed_for_user( user )
    self_user = InstagramUser.sync_from_user( user )

    user.instagram_client.user_recent_media.each do |media|
      InstagramMedia.reify( media, self_user )
    end
  end

  def self.reify( media, instagram_user )
    p = where( media_id: media['id'] ).first_or_create
    p.instagram_user = instagram_user
...
```

When we rerun this again, we’ll get a VCR error since we’re adding a new API call into the mix.  Lets delete the cassette and run it again:

```
$ rm spec/vcr/instagram/recent_feed_for_user.yml 
```

And we’ve synced the user correctly.

## Setting up some relationships

We don’t really need to test that ActiveRecord associations work in their entirety, but lets puts in some expectations on how we expect to query things.

```
  context "relationships" do
    before( :each ) do
      VCR.use_cassette 'instagram/recent_feed_for_user' do
        InstagramMedia.recent_feed_for_user( user )
      end
    end

    it "should associate an instragram user with the user" do
      expect( user.instagram_user ).to_not be_nil
    end

    it "should associate posts with the instragram user" do
      expect( user.instagram_user.posts.count ).to be > 0
    end

    it "should associate comments with a post" do
      post = InstagramMedia.first
      expect( post ).to_not be_nil
      expect( post.comments.count ).to eq( 0 )
    end

    it "should associate likes with a post" do
      post = InstagramMedia.first
      expect( post ).to_not be_nil
      expect( post.likes.count ).to eq( 0 )
    end

''	    it "should know the users surrounding the user" do
      count = user.instagram_user.nearby_users.count
      create( :instagram_user )
      expect( user.instagram_user.nearby_users.count ).to eq( count )
    end

  end
```

Lets link `InstagramUser` with `User`, like so:
add `has_one :instagram_user` to `user.rb`

Inside of `instagram_user.rb`, link `InstagramMedia` with `InstagramUser` and make a fancy query to get nearby users:

```
has_many :posts, class_name: 'InstagramMedia'

  def nearby_users
    query = InstagramUser.from "instagram_users, instagram_media, instagram_interactions"
    query = query.where "instagram_media.instagram_user_id = #{self.id}"
    query = query.where "instagram_media.id = instagram_interactions.instagram_media_id"
    query = query.where "instagram_interactions.instagram_user_id = instagram_users.id"
  end

```


And set up some of the interactions links in `instagram_media.rb`:


```
  has_many :interactions, class_name: 'InstagramInteraction'

  def comments
    interactions.where( is_like: false )
  end

  def likes
    interactions.where( is_like: true )
  end
```

And now we should be all green again!


## Pull in the interactions

Lets first work on bringing in comment interactions.

```
  context "comments" do
    before( :each ) do
      VCR.use_cassette 'instagram/recent_feed_for_user' do
        InstagramMedia.recent_feed_for_user( user )
      end
    end

    it "should load multiple InstagramUsers" do
      expect( InstagramUser.count ).to be > 1
    end

    it "should have created many instagram interactions" do
      expect( InstagramInteraction.count ).to be > 0
    end
  end

```

Inside of the `InstagramMedia.reify` method, we’ll loop through the comments and add them as found:

```
''	    media['comments']['data'].each do |comment|
      commentor = InstagramUser.from_hash( comment['from'] )
      InstagramInteraction.where( instagram_media_id: p.id, instagram_user_id: commentor.id, is_like: false ).first_or_create do |interaction|
        interaction.comment = comment['text']
        interaction.created_at = Time.at( comment['created_time'].to_i )
      end
    end
```

And then lets 
This is going to fail because of nonexistent `from_hash` method on `InstagramUser`.  

## Small refactoring of InstagramUser

Let’s add a spec for that inside of `instagram_users_spec`:

```
  it "should create a user from a hash" do
    InstagramUser.from_hash 'username' => 'wschenk', 'full_name' => 'Will Schenk'

    expect( InstagramUser.count ).to eq(1)
    expect( InstagramUser.first.username ).to eq( 'wschenk' )
  end

  it "should update the read counts if it find them" do
    InstagramUser.from_hash "username" => 'wschenk', "full_name" => 'Will Schenk'

    InstagramUser.from_hash "username" => 'wschenk', "full_name" => 'Will Schenk', 'counts' => { 'media' => 100 }

    expect( InstagramUser.count ).to eq(1)
    expect( InstagramUser.first.username ).to eq( 'wschenk' )
    expect( InstagramUser.first.media_count ).to eq( 100 )
  end

```

We’re checking first that it creates the instragram user as needed, and then secondly that it updates the user the second time with more info.

Here are the changes to user.rb, really just splitting up the one method into 2:

```
  def self.sync_from_user( user )
    user_info = user.instagram_client.user

    from_hash user_info, user
  end

  def self.from_hash user_info, user = nil
    u = where( :username => user_info['username']).first_or_create
    u.user_id = user.id if user
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

## Pulling in likes

For the first pass at kicking the tires, we’re going to loop through all of the users posts and make sure that the cached count in the `instagram_media` table is the same as if we query for the comments and likes directly:

```
context "interactions" do
    before( :each ) do
      VCR.use_cassette 'instagram/recent_feed_for_user' do
        InstagramMedia.recent_feed_for_user( user )
      end
    end

    it "should check that the comment count are the same as in the media object" do
      user.instagram_user.posts.each do |post,idx|
        expect( post.comments.count ).to eq( post.comments_count)
      end
    end

    it "should check that the likes count are the same as in the media object" do
      user.instagram_user.posts.each do |post,idx|
        expect( post.likes.count ).to eq( post.likes_count)
      end
    end
  end

```

Here we get some fails, and that’s because we haven’t done anything for loading likes!

Inside of `instagram_media.rb` we need to add the code to `reify`.  But we also need to pass in 

```
  def self.recent_feed_for_user( user )
    self_user = InstagramUser.sync_from_user( user )

    user.instagram_client.user_recent_media.each do |media|
      InstagramMedia.reify( media, self_user, user.instagram_client )
    end
  end

  def self.reify( media, instagram_user, client )

......

''	    client.media_likes( media['id'] ).each do |like|
      liker = InstagramUser.from_hash( like )
      InstagramInteraction.where( instagram_media_id: p.id, instagram_user_id: liker.id, is_like: true ).first_or_create
    end

```

And since we’ve change the logic for hitting the API, we’ll need to clear our the VCR cache again:

```
$ rm spec/vcr/instagram/recent_feed_for_user.yml 
```

Now when I run this, I get a few errors.  Once is that 

```
  1) InstagramMedia relationships should associate likes with a post
     Failure/Error: expect( post.likes.count ).to eq( 0 )

```

Which makes sense, since that post actually does have likes.  So I can change it to 18, which is the specific number of likes the specific post has.  Simple, we just change the number.

## API data discrepancies

The other issue that we have, at least for my feed, is the number of likes returned from the media query is different from the number of likes `media_likes` returns.  

```
  1) InstagramMedia interactions should check that the likes count are the same as in the media object
     Failure/Error: expect( post.likes.count ).to eq( post.likes_count)
       
       expected: 13
            got: 12

```

To remind everyone what we’re doing, `post.likes_count` is the data that’s returned from `user_recent_media` call, while `likes` are populated by looking at the `media_likes` call of the specific post.  Who to believe?  If I open up the instagram client on my phone and count the number of individual users who like that post, the app displays 12.  But in the main feed, it displays 13.  So the problem is on instagram’s side, not our code, sigh.

Lets change the test to be a little more flexible, to work around this _issue_ on the API side:

```
    it "should check that the likes count are the same as in the media object" do
      user.instagram_user.posts.each do |post,idx|
        expect( post.likes.count ).to be_within( 2 ).of( post.likes_count)
      end
    end
```

Now the tests pass, but they aren’t testing things as specifically.

## Check in the code

We’ve got the basics of syncing in place now, lets take a look at the UI next.  We’ll check in the code now to make our mark.

```
$ git add .
$ git commit -a -m "InstagramMedia and Interactions"
```

(Github link: https://github.com/HappyFunCorp/instacrush_tutorial/commit/08c97152f210caf88f370413ac3ae8ceb998d81e)