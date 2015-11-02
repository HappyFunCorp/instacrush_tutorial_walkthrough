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

## F