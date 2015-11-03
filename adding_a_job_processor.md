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

This creates the 