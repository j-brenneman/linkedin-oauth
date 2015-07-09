https://dry-ridge-9623.herokuapp.com/

## Boilerplate Part 1 - Create and deploy your app

Generate an express app that includes a `.gitignore` file:

```
express --git linkedin-oauth
cd linkedin-oauth
npm install
nodemon
```

Visit http://localhost:3000/ and make sure that the app loads correctly.  Then initialize a git repository:

```
git init .
git add -A
git commit -m "Initial commit"
```

Now create a repository on Github, set the remote properly and push.

## Boilerplate Part 2 - Deploy

Create an app on Heroku, deploy to it and verify that your app works on Heroku:

```
heroku apps:create
git push heroku master
heroku open
```

Now that you have a Heroku URL:

1. add a README file
1. add your Heroku URL to the README
1. git add, commit and push to Github

## Boilerplate Part 3 - Install and configure dotenv and cookie-session

Passport requires that your app have a `req.session` object it can write to.  To enable this, install and configure `cookie-session`.  In order to keep your secrets safe, you'll need to also configure and install `dotenv`.

```
npm install dotenv --save
touch .env
echo .env >> .gitignore
npm install cookie-session --save
```

In your `.env` file add two environment variables: `SESSION_KEY1` and `SESSION_KEY2`.  It should look something like this:

```
SESSION_KEY1=abc123
SESSION_KEY2=abc123
```

In `app.js`, configure `cookie-session` appropriately:

```js
// in the require section up at the top...
var session = require('cookie-session');
require('dotenv').load()

// after app.use(express.static(path.join(__dirname, 'public')));
app.use(session({ keys: [process.env.SESSION_KEY1, process.env.SESSION_KEY2] }))
```

Ensure that you app works locally, then

1. git add, commit, push to Github
1. deploy to Heroku

Once you've deployed, be sure to set `SESSION_KEY1` and `SESSION_KEY2` on Heroku:

```
heroku config:set SESSION_KEY1=abc123 SESSION_KEY2=def234
```

Verify that your app still works correctly on Heroku.

## Boilerplate Part 4 - Set the HOST environment variable

For your app to work both locally and on production, it will need to know what URL it's being served from.  To do so, add a HOST environment variable to `.env` locally to localhost:

```
echo HOST=http://localhost:3000 >> .env
```

Then set the HOST environment variable on Heroku to your Heroku URL:

```
heroku config:set HOST=https://your-heroku-app.herokuapp.com
```

NOTE: do _not_ include the trailing slash.  So `https://guarded-inlet-5817.herokuapp.com` instead of `https://guarded-inlet-5817.herokuapp.com/`

There should be nothing to commit at this point.

## Register your LinkedIn Application

1. Login to https://www.linkedin.com/
1. Visit https://www.linkedin.com/developer/apps and create a new app
1. For Logo URL, add your own OR you can use https://brandfolder.com/galvanize/attachments/suxhof65/galvanize-galvanize-logo-g-only-logo.png?dl=true
1. Under website your Heroku URL
1. Fill in all other required fields and submit

On the "Authentication" screen:

- Under authorized redirect URLs enter http://localhost:3000/auth/linkedin/callback
- Under authorized redirect URLs enter your Heroku url, plus `/auth/linkedin/callback`

You should see a Client ID and Client Secret.  Add these to your `.env` file, and set these environment variables on Heroku.  Your `.env` file should look something like:

```
LINKEDIN_CLIENT_ID=abc123
LINKEDIN_CLIENT_SECRET=abc123
```

Set those variables on Heroku as well:

```
heroku config:set LINKEDIN_CLIENT_ID=abc123 LINKEDIN_CLIENT_SECRET=def234
```

There should be nothing to add to git at this point.

## Install and configure passport w/ the LinkedIn strategy

```
npm install passport --save
npm install passport-linkedin-oauth2 --save
```

First, add the Passport middleware to `app.js`:

```js
// up with the require statements...
var passport = require('passport');

// above app.use('/', routes);
app.use(passport.initialize());
app.use(passport.session());
```

Then tell Passport to use the LinkedIn strategy:

```js
// up with the require statements...
var LinkedInStrategy = require('passport-linkedin-oauth2').Strategy

// below app.use(passport.session());...
passport.use(new LinkedInStrategy({
    clientID: process.env.LINKEDIN_CLIENT_ID,
    clientSecret: process.env.LINKEDIN_CLIENT_SECRET,
    callbackURL: process.env.HOST + "/auth/linkedin/callback",
    scope: ['r_emailaddress', 'r_basicprofile'],
    state: true
  },
  function(accessToken, refreshToken, profile, done) {
    done(null, {id: profile.id, displayName: profile.displayName, token: accessToken})
  }
));
```

Finally, tell Passport how to store the user's information in the session cookie:

```js
// above app.use('/', routes);...
passport.serializeUser(function(user, done) {
  done(null, user);
});

passport.deserializeUser(function(user, done) {
  done(null, user)
});
```

Run the app locally to make sure that it's still functioning (and isn't throwing any errors).

## Create the auth and oauth-related routes

Create a new route file for your authentication routes:

```
touch routes/auth.js
```

In `routes/auth.js`, you'll need to add a route for logging in, and one for logging out.  In addition, you'll have to create the route that LinkedIn will call once the user has authenticated properly:

```
var express = require('express');
var router = express.Router();
var passport = require('passport');

router.get('/auth/linkedin', passport.authenticate('linkedin'));

router.get('/logout', function (req, res, next) {
  req.session = null
  res.redirect('/')
});

router.get('/auth/linkedin/callback',
  passport.authenticate('linkedin', {
    failureRedirect: '/',
    successRedirect: '/'
  }));

module.exports = router;
```

Back in `app.js`, be sure to require and use the new routes file:

```js
// up with the require statements...
var authRoutes = require('./routes/auth');

// right after app.use('/', routes);
app.use('/', authRoutes);
```

With this setup, you should be able to login with LinkedIn successfully by visiting the following URL directly:

http://localhost:3000/auth/linkedin

If it's successful, you should be redirected to the homepage.  If you check your terminal output, you should see a line in there like:

```
GET /auth/linkedin/callback?code=AQRVExUd3t12eS0Zm3y
```

That indicates that LinkedIn successfully authenticated the user!

## Configure the views

Now you'll want to:

- add login and logout links
- display the name of the currently logged-in user

To do so, add the following lines inside the `body` tag in `./views/layout.jade`:

```jade
    if user
      p
        |Hello #{user.displayName}
        ||
        a(href="/logout") Logout
    else
      p
        a(href="/auth/linkedin") Login
```

Now add some middleware that will set the `user` local variable in all views:

```js
// right above app.use('/', routes);
app.use(function (req, res, next) {
  res.locals.user = req.user
  next()
})
```

Passport sets the `req.user` property for you automatically.

You should now be able to login and logout with LinkedIn!!!

1. Git add, commit and push
1. Deploy to Heroku
1. Check that your app works on Heroku

## Make an API call to LinkedIn on the user's behalf

Install unirest:

```
npm install unirest --save
```

Update your `routes/index.js` file to this:

```js
var express = require('express');
var router = express.Router();
var unirest = require('unirest');

/* GET home page. */
router.get('/', function(req, res, next) {
  if(req.isAuthenticated()) {
    unirest.get('https://api.linkedin.com/v1/people/~:(id,num-connections,picture-url)')
      .header('Authorization', 'Bearer ' + req.user.token)
      .header('x-li-format', 'json')
      .end(function (response) {
        res.render('index', { profile: response.body });
      })
  } else {
    res.render('index', {  });
  }
});

module.exports = router;
```

Update your `views/index.jade` file to this:

```
extends layout

block content
  if profile
    img(src=profile.pictureUrl)
```

In your browser, you should now see your LinkedIn profile picture appear (if you have one).

Go to https://apigee.com/console/linkedin to see all the various API options you have available.

- Git add, commit and push to Github
- Deploy to Heroku

## Stretch Option: Save profiles to your Mongo database

Whenever a user logs in, save their data in a document in your users collection.  To begin:

1. Install monk
1. Configure monk to use an environment variable for its connection string (MONGOLAB_URI)
1. Add Mongolab to Heroku (`heroku addons:create mongolab`)

Then you'll need to find-and-update or insert a document every time someone logs in.  Mongo has an [`upsert` option](http://docs.mongodb.org/manual/reference/method/db.collection.update/#db.collection.update) in its `update` method that's perfect for this.

Here's a decent place to add this mongo code (see comments inline):

```js
passport.use(new LinkedInStrategy({
    clientID: process.env.LINKEDIN_CLIENT_ID,
    clientSecret: process.env.LINKEDIN_CLIENT_SECRET,
    callbackURL: process.env.HOST + "/auth/linkedin/callback",
    scope: ['r_emailaddress', 'r_basicprofile'],
    state: true
  },
  function(accessToken, refreshToken, profile, done) {
    // here, find or create a document in the user collection, and update it's contents
    // NOTE: the profile object is unnecessarily big.  Only store the parts you care about here.
    done(null, {id: profile.id, displayName: profile.displayName, token: accessToken})
  }
));
```

## Stretch Option: Post updates to LinkedIn

Add functionality to post an update to LinkedIn.  CAREFUL!  It will actually update your real LinkedIn feed :)

## Stretch Option: Only store the user id in the session

This tutorial has you store the user's LinkedIn id, displayName and accessToken in the session.

Instead, store all that info on the users document, and then only store the user's `_id` in the session.

Now you have a record of everyone who logged in, and you can easily expand your site to accomodate multiple other providers.

## Stretch Option: Add new strategies

Allow users to also login with Twitter, or GitHub, or any strategy listed on http://passportjs.org/

## Resources

- https://developer.linkedin.com/docs/oauth2
- https://github.com/auth0/passport-linkedin-oauth2
- http://passportjs.org/docs
- http://passportjs.org/docs/configure#configure
- https://apigee.com/console/linkedin
- http://docs.mongodb.org/manual/reference/method/db.collection.update/#db.collection.update
