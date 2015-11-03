# Adding a Job Processor

Now that we've put ActiveJob into the system, lets setup a job processor so we can see what it does.  We'll use `delayed_job` at first, since that doesn't require a new database backend.

## DelayedJob

In your `Gemfile` we add:

```
gem 'delayed_job_active_record'
```

Then run:

```
$ bundle
$ rails generate delayed_job:active_record
$ rake db:migrate
```

This creates a `delayed_jobs` table where the jobs get stored, and then polled from a seperate worker process.  Lets open up our `Procfile` and setup that process to run:

```
web: bundle exec puma -C config/puma.rb
worker: bundle exec rake jobs:work
```

Now lets startup a server using `foreman start`.  In the logs now you should we stuff from `web.1` which is the puma/rails process, and `worker.1` which is our delayed job.  It's probably polling quite a lot.

## Testing out queuing

Inside of `instagram_user.rb`, lets expose our sync method with a little refactoring:

```
  def sync_if_needed
    if stale? && state != "queued"
      sync!
    end
  end

  def sync!
    update_attribute :state, "queued"
    UpdateUserFeedJob.perform_later( self.id )
  end
```

Now lets start up the console and see what happens:

```
$ rails c
[1] instacrush_tutorial »  InstagramUser.first.sync!
```

And we get pages of SQL queries! _The reason that it didn't work is that we never told rails about the queing_.  Lets go and add that now inside of `application.rb`:

```
    config.active_job.queue_adapter = :delayed_job
```

And restart foreman and the rails console.  Now when we run the sync command from the console we get:

```
[1] instacrush_tutorial »  InstagramUser.first.sync!
Enqueued UpdateUserFeedJob (Job ID: b58ef359-ff28-4c3f-96dd-adeacd3cc598) to DelayedJob(default) with arguments: 1
[ActiveJob] Enqueued UpdateUserFeedJob (Job ID: b58ef359-ff28-4c3f-96dd-adeacd3cc598) to DelayedJob(default) with arguments: 1
[ActiveJob]    (0.7ms)  commit transaction
=> #<UpdateUserFeedJob:0x007f995552d258 @arguments=[1], @job_id="b58ef359-ff28-4c3f-96dd-adeacd3cc598", @queue_name="default">

```

And if we flip back to the foreman log you can see the job running.

Let's clear out the sync state in the console, and see what we see on the front end.

```
[2] instacrush_tutorial »   InstagramUser.first.update_attribute( :last_synced, 1.day.ago )
```

![](Crush___Seed_Site_and_instagram_user_rb_—_instacrush_tutorial.jpg)

That's not what we want!  Lets move over our previous design.

```
$ mv app/views/welcome/calculating.html.haml app/views/crush/loading.html.haml
```

And change it to refresh the page:

```
          window.location = '#{ loading_crush_index_path }';
```

Lets reset our user once again:

```
[10] instacrush_tutorial »  InstagramUser.first.update_attribute( :last_synced, 1.day.ago )
```

