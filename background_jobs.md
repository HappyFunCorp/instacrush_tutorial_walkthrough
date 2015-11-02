# Background Jobs

We are currently doing the talking to Instagram inline, which can take upwards of 20 seconds.  We may not realize it since we are caching it, but lets integrate ActiveJob so that the user can go about their day while we make the first connection.

The strategy is to make sure all of our tests pass, then change the 