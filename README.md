# Installing Mediawiki on Pivotal Web Services

## About this page
This is a rough draft, written for someone with a lot of context about using Cloud Foundry.

## Setup
[Download the latest stable Mediawiki](https://www.mediawiki.org/wiki/Download) and unpack it. Remove the `/vendor` directory; it's better handled by Composer.

I've included in this repo a `composer.lock` for Mediawiki 1.28.2, and a cf manifest sample. Copy those into the Mediawiki folder.

Rename the manifest sample into manifest.yml, and edit the placeholder values for the wiki host and name.

You should be able to `cf push` now, go to the https page, and be able to start entering configuration details.

## Database setup
We're going to bind a database to the wiki app, and pry the details out of it just long enough for Mediawiki to set up the database.

Let's attach a cleardb database. I chose that because it looked like the only MySQL / MariaDB offering that was free.

```
cf create-service cleardb spark jameswikidb
cf bind-service jameswiki jameswikidb
```

Ok, let's try Postgres instead, because I'm getting size complaints from cleardb.

```
cf create-service elephantsql turtle jameswikidb
cf bind-service jameswiki jameswikidb
```

Ah, a new Mediawiki setup squeaks in just below the 5mb cutoff for cleardb.
ElephantSQL has a more gracious 20mb cutoff for free accounts.

## Wiki setup
We'll use details from `cf env jameswiki` to set up the wiki database.

### cleardb / mysql
Specifically under `VCAP_SERVICES.cleardb.credentials` on the wiki setup:

* Database host: `hostname`
* Database name: `name`
* Database table prefix: you can make one up, like `jameswiki`
* Database username: `username`
* Database password: `password`

### ElephantSQL / postgres
The details we need are under `VCAP_SERVICES.elephantsql.credentials.uri`. The syntax:
`postgres://(username):(password)@(hostname):5432/(database name)`

### Further wiki setup
If you let the wiki install ask more questions, you can:

* make it a private wiki.
* email is on by default.
This might not work as-is, or be a good idea.
* enable file uploads - this is disabled by default, and is tricky on cf.
* ... continue. it will give you a `LocalSettings.php` file to download.


## Tweak LocalSettings.php
*This is a Day Two sorta thing: nice to have, but not crucial.*

We can tweak `LocalSettings.php` to handle service creds rotation and move out secrets.

Copy `LocalSettings.php` into the Mediawiki folder. Edit it, along with the manifest.

Replace the `## Database settings` section:

```
$url = parse_url(getenv("DATABASE_URL"));

$wgDBtype = "mysql"; # or "postgres", as appropriate
$wgDBserver = $url["host"];
$wgDBname = substr($url["path"], 1);
$wgDBuser = $url["user"];
$wgDBpassword = $url["pass"];
```

While we're at it, we can move the values for `wgSecretKey` and `wgUpgradeKey` into the manifest. Manually copy them over, and replace them in `LocalSettings.php` with: 

```
$wgSecretKey = getenv("SECRET_KEY");
$wgUpgradeKey = getenv("UPGRADE_KEY");
```

## Update the app
After that, `cf push` should put the config in place, and the wiki should be usable.
You should make a new user so you're not always running as admin:
Log in as Admin, hit the sidebar's "Tools / Special Pages," and see "Login / Create account", "Create account."

And then it's all success and unicorns and rainbows.

## Upgrading
Preserve `LocalSettings.php` and the manifest.

### Updating the composer files
Download the new Mediawiki version tarball locally and unpack it.
(Git would be nicer, but the repo lacks skins by default.)
Remove the `/vendor` directory, to avoid confusing composer.

Install composer, if you don't have it already:

```
curl -sS https://getcomposer.org/installer | php
```

That will put an executable `composer.phar` in the current directory.

To make the new `composer.lock`, run `./composer.phar update`.

## References
* [Ron Gross' blog post](http://v1.ripper234.com/p/how-to-setup-a-free-mediawiki-on-heroku/) about running Mediawiki on Heroku
* [caseywatts' gist on Mediawiki on Heroku](https://gist.github.com/caseywatts/d04bda6626ef2c6c8f97), largely based on Ron Gross', above. This has info on adding the vector skin if you go the Mediawiki git repo route. This has a cleaner way of parsing the database connection string.