# Modeling the data

We’ve played around with the api on the console and have a basic idea of what we can get to.  There are 3 major objects that we have.

1. `InstagramUser` which has things like `username`, `uid`, and in some cases number of posts and each directional relationship.
2. `InstagramMedia` which contains post information.
3. `InstagramInteraction` which is an association between `InstagramUser` and `InstagramMedia`, and can either be a like or a comment.

We’re going to add `sync!` methods on `InstagramUser` which will sync the user meta data and recent media, and then a `sync!` method on `InstagramMedia` which will populate the likes and comments as needed.  And we’re going to add a `sync!` method on `User` that will get triggered by the UI.

## Test Driven Model Development

We’re going to use a TDD process to build out these models.  Testing can be overdone, and as you get further and further up the application towards the user interface sometimes the payoff for all the effort isn’t there.  But for low level model logic, especially when dealing with an external API, it’s always nice to have some coverage for how the gritty stuff works, since its never how you expect.  We’re going to start by scaffolding out the models, write specs and make them pass, and then tweak the final rails views to start seeing what’s there.

After that, we’ll build a real interface on top of it, and phase out the views.


