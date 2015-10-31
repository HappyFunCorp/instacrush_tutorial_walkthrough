# Wiring up the UI

We have the user, we have syncing, we have the crush, we have wireframes, lets start putting everything together!

We’re going to go straight through the flow, and put each page in place.  As we move things out of the welcome controller, we’ll remove the routes from `routes.rb` and we’ll see where things break.



## Landing

Change the button in `welcome/landing.html.haml` to link to the `user_omniauth_authorize_path` instead of the loading page.

```
        = button_to "Link with instagram", user_omniauth_authorize_path(:instagram), class: "btn btn-default btn-lg"
```

Now when we click on that, we get the flash message and are put right back on the landing page.  So lets edit the controller and  have it redirect logged in users to the crush page.

But first we have to create a crush controller!  Let’s create one with a few actions first:

```
$ rails g controller crush index show loading
```

and tweak the routes to something that makes more sense (_remove the get crush stuff_).  Also, lets comment out the calculating and show crush mocksup.

```
  resources :crush, only: [:index, :show] do
    collection do
      get 'loading'
    end
  end

 # get 'welcome/calculating'
 # get 'welcome/show_crush'
```

Now inside of `welcome_controller.rb`, change the `landing` action:

```
  def landing
    if current_user.present?
      redirect_to crush_index_path
    end
  end
```

Load the page, and we’ll have a quick error we need to fix in `header.html.haml`

```
      = link_to "Crush", crush_index_path
```

Head back to the browser and make sure that everything is working correctly.

## Crushes

Lets work on the crush controller.  We’re going to first make some filters.  If these filters don’t work, it will direct to a place that will hopefully make it work.

- that the user exists and has a instagram token
- that the user has a recently synced feed
- that the user has a recently synced crush

We’re going to put these in `application_contoller.rb`:

```
  def require_instagram_user
    if current_user.nil? || current_user.instagram.nil?
      store_location_for( :user, request.path )
      redirect_to user_omniauth_authorize_path( :instagram )
      return false
    elsif current_user.instagram_user.nil?
      redirect_to crush_index_path
      return false
    end
  end

  def require_fresh_user
    iu = current_user.instagram_user
    if iu.last_synced.nil? || iu.last_synced < 12.hours.ago
      InstagramMedia.recent_feed_for_user current_user
      iu.update_attribute :lasted_synced, Time.now
      flash[:notice] = "We've spoken to instagram"
    end
  end
```

We’re going to refactor `require_fresh_user` later, but this is simple place to put it for now.

OK, on to `crush_controller.rb`:

```
class CrushController < ApplicationController
  before_filter :require_instagram_user
  before_filter :require_fresh_user
  
  def index
    redirect_to Crush.find_for_user( current_user )
  end

  def show
    @crush = Crush.find( params[:id] )
  end

  def loading
  end
end
```

Now when we go to the site, we see that it takes a while when you click on Crush, and you’ll get a notice that “We’ve spoken to Instagram”.  After this, it goes quickly.  Try logged out, and then pressing on the nav links to see what happens, and to make sure that you actually get logged in.

In fact, you’ll see that you never actually go to the `index.html.haml` page, since you always get redirected to a show page.  So delete that page now:

```
$ rm app/views/crush/index.html.haml
```

But we do have the show page, so lets move over our `show_crush` mockup:

```
$  mv app/views/welcome/show_crush.html.haml app/views/crush/show.html.haml 
```

Now lets wire it all up.  Here’s the full html, but the changes are pretty simple:

```
.container
  .row
    .center-panel.text-center
      = panel title: "@#{@crush.instagram_user.username}'s Crush" do
        .panel-body
          .crush_pane
            .person
              = image_tag @crush.crush_user.profile_picture
            .heart
              %span.glyphicon.glyphicon-heart
            .person
              = image_tag @crush.instagram_user.profile_picture

          .stats
            %code= "@#{@crush.crush_user.username}"
            has hearted 
            %b= @crush.likes_count
            and commented on
            %b= @crush.comments_count
            of 
            %code
              ="@#{@crush.instagram_user.username}'s"
            recent photos!


      .row
        .col-xs-6.text-center
          = link_to "Top Users", welcome_top_users_path, class: "btn btn-primary btn-lg"
        .col.xs-6.text-center
          = link_to "Top Posts", welcome_top_posts_path, class: "btn btn-primary btn-lg"
```

Do you see your changes?  Fun stuff!

## Marking time

```
$ git add .
$ git commit -a -m "Crush page and calculation"
```

(Github link: https://github.com/HappyFunCorp/instacrush_tutorial/commit/bde0cec2aa90fb1d828abed9f8bb7a1352e90c04)

## Instagram media

Lets do the top posts now, and rip out the `InstagramMedia` scaffolding.  We’re going to first strip down the controller to have only two actions, `index` and `show`.  Lets first start with the controller:

```
class InstagramMediaController < ApplicationController
  before_filter :require_instagram_user
  before_filter :require_fresh_user, only: [:index]
  before_action :set_instagram_medium, only: [:show]

  def index
    @instagram_media = current_user.instagram_user.posts.order( "likes_count desc" )
  end 

  def show    
    @instagram_medium = InstagramMedia.find(params[:id])
  end 
end
```

And inside of `app/views/instagram_media` lets delete all of those files, and then move the top posts over:

```
$ git rm app/views/instagram_media/*
$ mkdir app/views/instagram_media
$ git mv app/views/welcome/top_posts.html.haml app/views/instagram_media/index.html.haml
```

And lets tweak the routes a bit.  We’re going to remove the last two wireframes, `top_users` and `top_posts` paths, the interactions routes all together, and then change the extern url path for the `:instragram_meda` post:

```
  # get 'welcome/top_posts'
  # resources :instagram_interactions
  resources :instagram_media, only: [:index, :show], path: 'instagram/posts'
  resources :instagram_users, only: [:index, :show], path: 'instagram/users'
```

Since we’ve removed the `welcome_top_users_path ` and`welcome_top_posts_path` routes, search for that and change it to `instagram_users_path` and `instagram_media_path` in `crush/show.html.haml` and the `_header.html.haml` partial.

Now lets fix up the `instagram_media/index.html.haml` file:

```
.container
  .page-header
    %h1
      Top posts
      %span.small based on interactions

  .row
    - @instagram_media.each do |post|
      .col-xs-6.col-sm-4.col-md-3
        .thumbnail
          = link_to post.link do
            = image_tag post.standard_url

          .caption
            %p
              = post.likes_count
              likes,
              = time_ago_in_words post.created_at
              ago.

            / %p
            /   Interacted with by:
            /   - post.interactions.each do |i|
            /     = link_to i.instagram_user.username, i.instagram_user
```

I’ve commented out the list of people who’ve interacted with the photo since it makes the wrapping funny, but play around with it!

## Top users

Let’s first move over our `top_users.html.haml` file:

```
$ mv app/views/welcome/top_users.html.haml app/views/instagram_users/index.html.haml
```

And we are similarly going to rip out the controller to something much more manageable:

```
class InstagramUsersController < ApplicationController
  before_filter :require_instagram_user
  before_filter :require_fresh_user, only: [:index]
  before_action :load_stats
 
  def index
  end

  def show
    @user = InstagramUser.find( params[:id] )
	@instagram_media = @instagram_user.
      posts.
      joins( :interactions ).
      where( instagram_interactions: { instagram_user_id: @user.id } ).
      order( "likes_count desc")
    render :index
  end

  private
  def load_stats
    @top_users = current_user.instagram_user.top_interactors.limit( 10 )
    @user_hash = {}
    InstagramUser.where( "id in (?)", @top_users.collect { |x| x[:instagram_user_id] } ).each do |user|
      @user_hash[user.id] = user
    end
    @instagram_media = current_user.instagram_user.posts.order( "likes_count desc" )
  end
end
```

There are a few things here.  First we are calling `load_stats` for each method, and that’s going to load up our sidebar users `@top_users`, the users themselves in `@user_hash` and a list of posts.  The `top_interactors` method returns ids and counts, but in order to display the ui we need to have the InstagramUser itself.  So we are loading them all in one query and then sticking them in a hash for easy lookup.

Finally, we load up the posts for the user.  In the default action, we just get all of the posts.  But in the `show` action, we only want to return posts that the user interacted with.  We’re using a `join` here, to filter our the `InstagramMedia` (aka _posts_) for only the ones that have an `InstagramInteraction` with the passed in user id.  ActiveRecord can get complicated, so it helps to understand the underlying SQL.

```
[5] instacrush_tutorial »  @instagram_media = @instagram_user.    
                        »  posts.      
                        »  joins( :interactions ).      
                        »  where( instagram_interactions: { instagram_user_id: @user.id } ).      
                        »  order( "likes_count desc").to_sql      
=> SELECT "instagram_media".* FROM "instagram_media" 
	INNER JOIN "instagram_interactions" ON 
		"instagram_interactions"."instagram_media_id" = "instagram_media"."id" 
	WHERE "instagram_media"."instagram_user_id" = 1 
		AND "instagram_interactions"."instagram_user_id" = 3  
	ORDER BY likes_count desc
```

- `instagram_user.posts` will return a set of InstagramMedia objects that the user has made.  In the query the represents the `SELECT instagram_media.* FROM instagram_media` and `"instagram_media"."instagram_user_id" = 1` (assuming `instagram_user.id` is 1)
- `joins( :interactions )` will filter the results using the `has_many :interactions` relationship defined in InstagramMedia.  In the SQL, this represents `INNER JOIN "instagram_interactions" ON instagram_interactions"."instagram_media_id" = "instagram_media"."id"`.
- And now we want to only show interactions that were done by the user.  This is the `where( instagram_interactions: { instagram_user_id: @user.id } )` clause, which translates into `AND "instagram_interactions"."instagram_user_id" = 3`.
- Finally we order by the number of likes, descending.

In cases like these, I actually find it easier to write the SQL first and then translate it into ActiveRecord!

Now lets edit the `app/views/instagram_users/index.html.haml` file:

```
.container
  .row
    .col-sm-3
      = panel title: "Top Users" do
        .panel-body
          = nav as: :pills, layout: :stacked do
            - @top_users.each do |top_user|
              - user = @user_hash[top_user[:instagram_user_id].to_i]
              = link_to user do
                = user.username
                %span.badge= top_user[:count]

    .col-sm-9
      - if @user
        .row
          .col-sm-12
            = panel title: "@#{@user.username}: #{@user.full_name}" do
              .panel-body.user_info
                .col-xs-3
                  = image_tag @user.profile_picture, class: "img-responsive"

                .col-xs-9
                  - @top_users.each do |top_user|
                    - if top_user[:instagram_user_id].to_i == @user.id
                      %p= "@#{@user.username} has liked #{top_user[:liked]} of your photos, and made #{top_user[:count] - top_user[:liked]} comments."

      .row
        - @instagram_media.each do |post|
          .col-xs-6.col-sm-4.col-md-3
            .thumbnail
              = link_to post.link do
                = image_tag post.standard_url

              .caption
                %p
                  = post.likes_count
                  likes
                %p
                  = time_ago_in_words post.created_at
                  ago.

```

## Whew, we made it!

```
$ git add .
$ git commit -a -m "Top users and top posts"
```

(Github link: https://github.com/HappyFunCorp/instacrush_tutorial/commit/592bd438cd794d701b099574e6d47e18bdfc2761)
