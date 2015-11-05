# Moving to Sidekiq

Delayed Job is fine, but requires a lot of workers to parallelize the tasks.  Lets switch over to sidekiq instead, which is a little more complicated to deploy but starts running jobs faster and runs the jobs in a multithreaded environment.

## Installing redis locally

If you are on OSX, the best way to install redis is to use homebrew, which you can learn more about at http://brew.sh

Once you have that installed:

```
$ brew install redis
```

And then run

```
$ redis-server
```

## Installing redis on heroku

This is even easier:

```
$ heroku addons:create heroku-redis:hobby-dev
Creating redis-fitted-4457... done, (free)
Adding redis-fitted-4457 to happyfancyname... done
Setting REDIS_URL and restarting happyfancyname... done, v12
Instance has been created and will be available shortly
Use `heroku addons:docs heroku-redis` to view documentation.
```


## Installing sidekiq

Remove `delayed_job_active_record` from `Gemfile` and add `sidekiq`.

```
# gem 'delayed_job_active_record'
gem 'sidekiq'
gem 'sinatra', :require => nil
```

```
$ bundle
```

Inside of `Procfile`:

```
worker: bundle exec sidekiq 
```

And inside of `config/application.rb`:

```
config.active_job.queue_adapter = :sidekiq
```

And we'll mount the sidekiq admin panel using in our `routes.rb` file:

```
require 'sidekiq/web'
authenticate :admin_user do
  mount Sidekiq::Web => '/sidekiq'
end
```

OK, lets deploy this to heroku now.

```
$ git commit -a -m "Moving to sidekiq"
$ git push heroku master
```

(Github link: https://github.com/HappyFunCorp/instacrush_tutorial/commit/958442e641c768eb468d5634dbbab60edcdd4a22)