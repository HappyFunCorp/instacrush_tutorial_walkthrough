# Playing with the data

Now that we have the data in the database, lets build something that actually looks at the data and selects a “crush”.  Lets build a model, and we’ll play around with some specs.

We’re just going to build a model here, and not a scaffold, since we already know what the interface is going to be.  Simple.

```
$ rails g model crush user:references last_synced:datetime slug:string:index instagram_user_id:integer crush_user_id:integer likes_count:integer comments_count:integer
```

A crush is going to have a user object, which is the person that generated it.  That will user will have the same `instagram_user_id` at the `crush` object.  The person who has the crush will have an `InstagramUser` of`crush_user_id`, which may or may not have a user object.

## Building out some rspec test scaffolding

First thing we’re going to do is to build out some spec code that will let us create media, likes and comments.  Once we get this working, then we’ll start in with the crush logic itself:

```
require 'rails_helper'

RSpec.describe Crush, type: :model do
  let( :main_user ) { create( :user )}

  let( :user_1 ) { create( :instagram_user, username: "user_1", user: main_user ) }
  let( :user_2 ) { create( :instagram_user, username: "user_2" ) }
  let( :user_3 ) { create( :instagram_user, username: "user_3" ) }
  let( :user_4 ) { create( :instagram_user, username: "user_4" ) }
  let( :user_5 ) { create( :instagram_user, username: "user_5" ) }
  let( :user_6 ) { create( :instagram_user, username: "user_6" ) }
  let( :user_7 ) { create( :instagram_user, username: "user_7" ) }
  let( :user_8 ) { create( :instagram_user, username: "user_8" ) }


  def make_media( likers, commenters )
    post = create( :instagram_medium, instagram_user: user_1 )
    [likers].flatten.each do |user|
      post.likes.create( instagram_user: user )
    end
    [commenters].flatten.each do |user|
      post.comments.create( instagram_user: user )
    end
    post
  end

  context "plumbing" do
    it "should create a post with likes and comments" do
      make_media [user_2, user_3], user_4

      expect( InstagramMedia.count ).to eq(1)
      expect( InstagramInteraction.count ).to eq(3)

      post = InstagramMedia.first

      expect( post.likes.count ).to eq(2)
      expect( post.comments.count ).to eq(1)
    end
  end
end
```

When we run this, I get 

```
  1) Crush plumbing should create a post with likes and comments
     Failure/Error: post.likes.create( instagram_user: user )
     ActiveRecord::UnknownAttributeError:
       unknown attribute 'instagram_user' for InstagramInteraction.
```

Lets add that reference inside of `instagram_interaction.rb` now:

```
class InstagramInteraction < ActiveRecord::Base
  belongs_to :instagram_media
  belongs_to :instagram_user
end
```

Now when we rerun all of the tests, everything should be green.

## Interactions should count

Lets add a couple of test cases, and then see what it would take to make them all green:

```
  it "should count likes towards a crush" do
    make_media [user_2, user_3], []
    make_media user_2, []

    c = Crush.find_for_user main_user

    expect( c.instagram_user ).to eq( main_user.instagram_user )
    expect( c.crush_user ).to eq( user_2 )
    expect( c.likes_count ).to eq( 2 )
    expect( c.comments_count ).to eq( 0 )
  end

  it "should count comments towards a crush" do
    make_media [user_4, user_5], [user_6]
    make_media [user_7, user_8], [user_6]

    c = Crush.find_for_user main_user

    expect( c.instagram_user ).to eq( main_user.instagram_user )
    expect( c.crush_user ).to eq( user_6 )
    expect( c.likes_count ).to eq( 0 )
    expect( c.comments_count ).to eq( 2 )
  end

  it "should used the combined total" do
    make_media [user_2, user_4, user_5], [user_6]
    make_media [user_7, user_8], [user_2]

    c = Crush.find_for_user main_user

    expect( c.instagram_user ).to eq( main_user.instagram_user )
    expect( c.crush_user ).to eq( user_2 )
    expect( c.likes_count ).to eq( 1 )
    expect( c.comments_count ).to eq( 1 )
  end
```

## Getting interactions for the user

We need to query the database to see who has interacted with our user the most.  Our model is InstagramUser `have_many` InstagramMedia `have_many` InstagramInteraction `belongs_to` InstagramUser.  **Whew!**  This is going to get tricky.  Let’s get into it.

Lets start with the media object, and see what we can get from that.  You numbers will vary depending upon the specific post you have loaded.  If you don’t have any thing loaded, you can refresh your database with `InstagramMedia.recent_feed_for_user User.first`.  (And if you don’t have a user, go back to “Exploring the Instagram API” above to fire up the server and connect with instagram to get an access token.)

```
[1] instacrush_tutorial »  post = InstagramMedia.first
[2] instacrush_tutorial »  post.interactions.count
 => 14
[3] instacrush_tutorial »  post.likes.count
=> 13
[4] instacrush_tutorial »  post.comments.count
=> 1
```

OK, so we know that for this post, we have 14 total interactions.  13 of them are likes, which we know are unique people, and one is a comment, which could be also a liker or not.  So the total number of instagram users associated with this post is 13 or 14.

We’ll want to count these up and find our who is the tops.  Can we figure out who that is now?

```
[5] instacrush_tutorial »  user = post.instagram_user
[6] instacrush_tutorial »  user.interactions
NoMethodError: undefined method `interactions' for #<InstagramUser:0x007fd1d1f750d0>
```

OK, lets add this to our `InstagramUser` model.

```
 has_many :interactions, through: :posts, class_name: "InstagramInteraction"
```

Lets reload our code:

```
[7] instacrush_tutorial »  reload!; user = User.first.instagram_user 
```

And see what it gives us:

```
[8] instacrush_tutorial »  user.interactions.to_sql
=> "SELECT \"instagram_interactions\".* FROM \"instagram_interactions\" INNER JOIN \"instagram_media\" ON \"instagram_interactions\".\"instagram_media_id\" = \"instagram_media\".\"id\" WHERE \"instagram_media\".\"instagram_user_id\" = 1"
[9] instacrush_tutorial »  user.interactions.count
=> 313
```

OK, so now we need to do some grouping and summing queries.  This is somewhat advanced SQL stuff, so just roll with it.

What we want to do is to
- look at all the `user.interactions`
- select a few columns:
- `instagram_interactions.instagram_user_id`
- `count(*) as count`
- `sum(case when is_like then 1 else 0 end) as liked`
- group by `instagram_user_id`
- order by `count desc`
- limit to 10 results

```
[10] instacrush_tutoral »  user.interactions.select( "instagram_interactions.instagram_user_id, count(*) as count, sum(case is_like when 't' then 1 else 0 end) as liked" ).group( :instagram_user_id ).order( "count desc" ).limit( 10 ).collect { |x| { instgram_user_id: x[:instagram_user_id], count: x[:count], likes: x[:liked]} }
```

What this is doing is selecting the `instagram_user_ids`, the total interactions, and the sum of the interactions (`"instagram_interactions.instagram_user_id, count(*) as count, sum(case when is_like then 1 else 0 end) as liked"`) for each of user (`group( :instagram_user_id )`), and then ordering by the ones with the most users (`order( "count desc" )`).  Finally we turn the results into a hash so the console displayed them correctly.

Simple.  Lets clean that up a little and throw that into `InstagramUser`:

```
  def top_interactors
    interactions.
      select( [
              "instagram_interactions.instagram_user_id", 
              "count(*) as count",
              "sum(case is_like when 't' then 1 else 0 end) as liked"
              ].join( ',') ).
      group( :instagram_user_id ).
      order( "count desc" )
  end
```

Now back to `crush.rb`:

```
  def self.find_for_user( user )
    return nil if user.instagram_user.nil?

    top_crush = user.instagram_user.top_interactors.first

    instagram_user_id = user.instagram_user.id
    crush_user_id = top_crush[:instagram_user_id]
    interactions = top_crush[:count].to_i
    likes = top_crush[:liked].to_i

    c = Crush.where( instagram_user_id: instagram_user_id, crush_user_id: crush_user_id ).first_or_create

    # Always update likes and comments count if needed
    c.likes_count = likes
    c.comments_count = interactions - likes

    c.save

    c
  end
```

Make sure all of our tests pass!

## Mark the spot

Ok this puts us in a good place for generating the crush.  Lets mark our work here, and then clean up from the front end.

```
$ git add .
$ git commit -a -m "Crush model"
```

(Github link: https://github.com/HappyFunCorp/instacrush_tutorial/commit/d9edd52a90344ef28f6b1d7bfb52c66c1f163668)