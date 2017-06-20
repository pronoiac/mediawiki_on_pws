# Installing Mediawiki on Pivotal Web Services

## About this page
This is a rough draft, written for someone with a lot of context about using Cloud Foundry.

## Setup
You should clone [Mediawiki](https://github.com/wikimedia/mediawiki):
`git clone https://github.com/wikimedia/mediawiki.git`
I've included in this repo a `composer.lock` for Mediawiki 1.28.2, and a cf manifest sample. Copy those into the `mediawiki` folder.

Copy the manifest sample into manifest.yml. Edit the placeholder values for the host and name.

You should be able to `cf push` it now, go to the https page, and be able to start entering configuration details.

## Database setup
We're going to attach a database to the wiki app, and pry the details out of it just long enough for Mediawiki to set up the database.

Let's attach a cleardb database. I chose that because it looked like the only MySQL / MariaDB offering that was free.

```
cf create-service cleardb spark jameswikidb
cf bind-service jameswiki jameswikidb
cf restage jameswiki # so it can have db creds... maybe later.
cf env jameswiki
```

## Wiki setup
We'll use details from under `VCAP_SERVICES.cleardb.credentials` on the wiki setup: 

* Database host: `hostname`
* Database name: `name`
* Database table prefix: you can make one up, like `jameswiki`
* Database username: `username`
* Database password: `password`

If you let the wiki install ask more questions, you can:

* make it a private wiki. 
* email is on by default.
* enable file uploads - this is disabled by default, and is tricky on cf.
* ... continue. it will give you a `LocalSettings.php` file to download.

## Tweak LocalSettings.php
We can tweak `LocalSettings.php` to handle service creds rotation and move out secrets.

Copy `LocalSettings.php` into the Mediawiki folder. Edit it, along with the manifest.

Replace `## Database settings`: 

```
$url = parse_url(getenv("DATABASE_URL"));

$wgDBtype = "mysql";
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

Upgrading should be easy, with a `git pull` and another `cf push`.