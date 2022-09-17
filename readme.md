<img src=rockmigrations.svg height=100>

> The swiss (or austrian in this case) army knife for migrations.

wbmnfktr, 13.7.2022

### Module Description

RockMigrations has an easy API to do all the things you can do in the PW backend via code. This means you can **fully version control your site** or app simply by adding all the necessary fields and templates not via clicking but via writing simple scripts that do that tasks for you.

The module also contains several helpers that make it extremely easy to implement **fully automated CI/CD pipelines**.

## QuickStart

The first thing I always do is to add this to my development config ([see here how to manage different configs for dev/live](https://processwire.com/talk/topic/18719-maintain-separate-configs-for-livedev-like-a-boss/)):

```php
$config->rockmigrations = [
  'syncSnippets' => true,
];
```

This copies the VSCode snippets to the .vscode folder and that makes VSCode show helpful snippets, eg for loading RockMigrations and showing code suggestions:

<img src=https://i.imgur.com/F8Gd4q0.png height=200>

Then you can simply choose `rmf-datetime` for example and get the code to use to create your field:

<img src=https://i.imgur.com/EirBznI.png height=200>

The `rm` snippet lets you quickly access the RockMigrations module with typehints so that you get proper IntelliSense:

<img src=https://i.imgur.com/j11eVD3.png height=250>

To write your first migrations just put this in your `site/migrate.php`. The example code uses `bd()` calls for dumping data. You need TracyDebugger installed!

```php
/** @var RockMigrations $rm */
$rm = $modules->get("RockMigrations");
bd('Create field + template via RM');
$rm->createField('demo', 'text', [
  'label' => 'My demo field',
  'tags' => 'RMDemo',
]);
$rm->createTemplate('demo');
$rm->setTemplateData('demo', [
  'fields' => [
    'title',
    'demo',
  ],
  'tags' => 'RMDemo',
]);
```

Reload your site and you will see the new field and template in the backend and you'll see the message in the tracy debug bar.

Now do a modules refresh (not a regular page reload) and note that the migration run again (the same message appears in tracy). There are two important things to understand here:

1. Migrations can run multiple times and will always lead to the same result.
2. If you put your migrations in ready.php or an autoload module they will run on every request. This not a good idea as it may slow down your site significantly

Now do a regular page reload. The migration will not run as nothing has changed. A modules refresh does always force to run migrations.

## Limitations

RockMigrations might not support all external fields, especially not ProFields like RepeaterMatrix. Adding support has no priority for me because I'm not using it. If you need support for it please provide a PR or if you are interested in sponsoring that feature please contact me via PM in the forum.

But not to forget: You can still use the regular PW API to create fields and manipulate all kinds of things. It might just not be as convenient as the RockMigrations API.

## Where do I find out all those field and template properties?

1. You can edit your field or template and copy the code from there (I recommend to only copy the settings you need to make your migration files more readable):
   ![img](https://i.imgur.com/IAHV3VZ.png)

2. Hover the caret on the very right of the field of the setting you want to set:
   ![img](https://i.imgur.com/hmydzf5.png)

## Magic

RockMigrations does not only help you with your migrations and deployments but it also adds a lot of helpers that make developing with ProcessWire even more fun.

### Magic Page Classes

If you are not using [custom page classes](https://processwire.com/blog/posts/pw-3.0.152/#new-ability-to-specify-custom-page-classes) yet I highly recommend to start using them now! Without them every page is a "stupid" page object, but when using custom page classes you add so much more logic to your code and suddenly every event is an EventPage and every newsitem is a NewsPage. You avoid hook-hell and your IDE can assist you because it suddenly understands your code!

The problem with custom page classes is, that they do not trigger `init()` and `ready()` automatically. That means you can't attach hooks in custom page classes by default. But hooks that belong to the page class should in my opinion be written in the pageclass file and not in /site/ready.php! When using RockMigrations you can make your pageclass even smarter and attach hooks directly from within it's own pageclass file:

```php
// example pageclass
<?php namespace ProcessWire;
class DemoPage extends Page {
  use RockMigrations\MagicPage;
}
```

That's all you need to do! Now you can attach hooks in init() and ready():

```php
// example pageclass
<?php namespace ProcessWire;
class DemoPage extends Page {
  use RockMigrations\MagicPage;

  public function init() {
    // attach hooks here
    $this->wire->addHookAfter(...);
  }

  public function ready() {
    // or here
    $page = $this->wire->page; // currently viewed page
    $this->wire->addHookAfter(...);
  }

}
```

A `MagicPage` does also have other magic methods that make the most common hooks a lot easier to use:

- editForm($form) instead of hooking ProcessPageEdit::buildForm
- onSaveReady() instead of hooking Pages::saveReady

For all available methods see `MagicPages::addMagicMethods()`!

Note that this feature loads one page of each available template at every boot so it has a little performance penalty. If you have many templates you can disable the feature by setting `$config->useMagicClasses = false`.

### Snippets

Another option that helps you get started with migration syntax is using the shipped VSCode snippets. I highly recommend enabling the `syncSnippets` option in your config:

```php
// site/config.php
$config->rockmigrations = [
  "syncSnippets" => true,
];
```

## Watching files, paths or modules

RockMigrations can watch files, paths and modules for changes. It will detect changes on any of the files on the watchlist and trigger migrations to run if anything changed.

As from version 1.0.0 (29.8.2022) RockMigrations will not run all migrations if one file changes but will only migrate this single changed file. This makes the migrations run a lot faster!

When run from the CLI it will still run every single migration file to make sure that everything works as expected and no change is missed.

Sometimes it is necessary that even unchanged files are migrated. RockMatrix is an example for that, where the module file triggers the migrations for all Matrix-Blocks. In that case you can add the file to the watchlist using the `force` option:

```php
$matrix = ...;
$rm->watch($matrix, true, ['force'=>true]);
```

### Watching modules

You can easily watch any ProcessWire module for changes and trigger the `migrate()` method whenever the file is changed:

```php
// module needs to be autoload!
public function init() {
  $rm = $this->wire->modules->get('RockMigrations');
  if($rm) $rm->watch($this);
}
public function migrate() {
  bd('Migrating MyModule...');
}
```

### Watching files

You can watch single files or entire paths:

```php
$rm->watch(__FILE__, false);
$rm->watch(__DIR__."/foo");
```

Note that you need to define `FALSE` as second parameter if the file should not be migrated but only watched for changes. If you set it to `TRUE` the file will be included and executed as if it was a migration script (see examples below).

## Running migrations

RockMigrations will run migrations automatically when a watched file was changed. In case you want to trigger the migrations manually (eg after deployment) you can use the `migrate.php` file:

```php
php site/modules/RockMigrations/migrate.php
```

You can disable automatic running of migrations either by enabling CLI mode or by calling `noMigrate()`:

```php
// in your cli script
define('RockMigrationsCLI', true);

// in site/ready.php
/** @var RockMigrations $rm */
$rm = $this->wire->modules->get('RockMigrations');
$rm->noMigrate();
```

## Files On Demand

You can instruct RockMigrations to download files on demand from a remote server. This makes it possible to create content on the remote system (eg on the live server), pull data from the database to your local machine and as soon as you open a page RockMigrations will fetch the missing files from your remote server.

```php
// without authentication
$config->filesOnDemand = 'https://example.com';

// with http basic authentication
$config->filesOnDemand = 'https://user:password@example.com';
```

#### YAML

```php
$rm->watch("/your/file.yaml");
```

```yaml
fields:
  foo:
    type: text
    label: My foo field
```

#### PHP

```php
$rm->watch("/your/file.php");
```

```php
<?php namespace ProcessWire;
$rm->createField('foo', 'text');
```

### Auto-Watch

RockMigrations automatically watches `/site/migrate.php` and files like `YourModule.migrate.php`.

## Working with YAML files

RockMigrations ships with the Spyc library to read/write YAML files:

```php
// get YAML instance
$rm->yaml();

// get array from YAML file
$rm->yaml('/path/to/file.yaml');

// save data to file
$rm->yaml('/path/to/file.yaml', ['foo'=>'bar']);
```

## Working with fieldsets

Working with fieldsets is a pain because they need to have an opening and a closing field. That makes it complicated to work with it from a migrations perspective, but RockMigrations has you covered with a nice little helper method that can wrap other fields at runtime:

```php
// syntax
$rm->wrapFields($form, $fields, $fieldset);

// usage
$wire->addHookAfter("ProcessPageEdit::buildForm", function($event) {
  $form = $event->return;

  /** @var RockMigrations $rm */
  $rm = $this->wire->modules->get('RockMigrations');
  $rm->wrapFields($form, [
    'title' => [
      // runtime settings for title field
      'columnWidth' => 50,
    ],
    // runtime field example
    [
      'type' => 'markup',
      'label' => 'foo',
      'value' => 'bar',
      'columnWidth' => 50,
    ],
    'other_field_of_this_template',
  ], [
    'label' => 'I am a new fieldset wrapper',
  ]);
})
```

# Deployments

You can use RockMigrations to easily create fully automated CI/CD pipelines for Github. It only takes these simple steps:

- Setup SSH keys and add secrets to your repository
- Create workflow yaml file
- Push to your repo

## Setup SSH keys and add secrets to your repo

To use this workflow you need to set the referenced secrets in your git repo.

Create a keypair for your deploy workflow. Note that we are using a custom name `id_rockmigrations` instead of the default `id_rsa` to ensure that we do not overwrite an existing key. If you are using RockMigrations on multiple projects you can simply overwrite the key as you will only need it once during setup:

    ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rockmigrations -C "rockmigrations-[project]"

Copy content of the private key to your git secret `SSH_KEY`:

    cat ~/.ssh/id_rockmigrations

Copy content of keyscan to your git secret `KNOWN_HOSTS`

    ssh-keyscan your.server.com

Add the public key to your remote user:

    ssh-copy-id -i ~/.ssh/id_rockmigrations user@your.server.com

Or copy the content of the public key into the authorized_keys file

    cat ~/.ssh/id_rockmigrations.pub

Try to ssh into your server without using a password:

    ssh -i ~/.ssh/id_rockmigrations user@your.server.com

## Create the workflow yaml

Now create the following yaml file in your repo:

```yaml
# code .github/workflows/deploy.yaml
name: Deploy via RockMigrations

# Specify when this workflow will run.
# Change the branch according to your setup!
# The example will run on all pushes to main and dev branch.
on:
  push:
    branches:
      - main
      - dev

jobs:
  test-ssh:
    uses: baumrock/RockMigrations/.github/workflows/test-ssh.yaml@main
    with:
      SSH_HOST: your.server.com
      SSH_USER: youruser
    secrets:
      SSH_KEY: ${{ secrets.SSH_KEY }}
      KNOWN_HOSTS: ${{ secrets.KNOWN_HOSTS }}
```

Commit the change and push to your repo. You should see the workflow showing up in Github's Actions tab:

![img](https://i.imgur.com/JFvMqkE.png)

Once you got your SSH connection up and running you can setup the deployment. Remove or comment the job "test" and uncomment or add the job "deploy" to your `deploy.yaml`:

```yaml
jobs:
  deploy:
    uses: baumrock/RockMigrations/.github/workflows/deploy.yaml@main
    with:
      # specify paths for deployment as JSON
      # syntax: branch => path
      # use paths without trailing slash!
      PATHS: '{
        "main": "/path/to/your/production/webroot",
        "dev": "/path/to/your/staging/webroot",
      }'
      SSH_HOST: your.server.com
      SSH_USER: youruser
    secrets:
      SSH_KEY: ${{ secrets.SSH_KEY }}
      KNOWN_HOSTS: ${{ secrets.KNOWN_HOSTS }}
```

If you are using submodules just set the `SUBMODULES` input variable and add a `CI_TOKEN` to your repo secrets:

```yaml
# .github/workflows/deploy.yaml
name: Deploy via RockMigrations
on:
  push:
    branches:
      - main
      - dev
jobs:
  deploy:
    uses: baumrock/RockMigrations/.github/workflows/deploy.yaml@main
    with:
      # specify paths for deployment as JSON
      # syntax: branch => path
      # use paths without trailing slash!
      PATHS: '{
        "main": "/path/to/your/production/webroot",
        "dev": "/path/to/your/staging/webroot",
      }'
      SSH_HOST: your.server.com
      SSH_USER: youruser
      SUBMODULES: true
    secrets:
      CI_TOKEN: ${{ secrets.CI_TOKEN }}
      SSH_KEY: ${{ secrets.SSH_KEY }}
      KNOWN_HOSTS: ${{ secrets.KNOWN_HOSTS }}
```

See https://bit.ly/3ru8a7e how to setup a Personal Access Token for Github. You need to _create_ this token only once for your Github Account, not for every project, but you need to add it to every project that should be able to access private submodules!

Your workflow should copy files but fail at step `Trigger RockMigrations Deployment`. That is because you need to create a `site/deploy.php` file:

```php
// code site/deploy.php
<?php namespace RockMigrations;
require_once __DIR__."/modules/RockMigrations/Deployment.php";
$deploy = new Deployment($argv, "/path/to/your/deployments");
// custom settings go here
$deploy->run();
```

Note that you must set a path as second argument when creating a new instance of `Deployment`. This path ensures that if you run your deployment script on another machine (for example on a local DDEV environment) it will run "dry" and will not execute any commands. This only works if your local path is different from your remote path of course!

This is how it looks like if everything worked well:

![img](https://i.imgur.com/hSML6Ym.png)

## Debugging

Debugging can be hard when using CI/CD pipelines. If you get unexpected results during the PHP deployment you can make the script more verbose like this:

```php
...
$deploy->verbose();
$deploy->run();
```

## Migration Examples

### Field migrations

CKEditor field

```php
$rm->migrate([
  'fields' => [
    'yourckefield' => [
      'type' => 'textarea',
      'tags' => 'MyTags',
      'inputfieldClass' => 'InputfieldCKEditor',
      'contentType' => FieldtypeTextarea::contentTypeHTML,
      'rows' => 5,
      'formatTags' => "h2;p;",
      'contentsCss' => "/site/templates/main.css?m=".time(),
      'stylesSet' => "mystyles:/site/templates/mystyles.js",
      'toggles' => [
        InputfieldCKEditor::toggleCleanDIV, // convert <div> to <p>
        InputfieldCKEditor::toggleCleanP, // remove empty paragraphs
        InputfieldCKEditor::toggleCleanNBSP, // remove &nbsp;
      ],
    ],
  ],
]);
```

Image field

```php
$rm->migrate([
  'fields' => [
    'yourimagefield' => [
      'type' => 'image',
      'tags' => 'YourTags',
      'maxFiles' => 0,
      'descriptionRows' => 1,
      'extensions' => "jpg jpeg gif png svg",
      'okExtensions' => ['svg'],
      'icon' => 'picture-o',
      'outputFormat' => FieldtypeFile::outputFormatSingle,
      'maxSize' => 3, // max 3 megapixels
    ],
  ],
]);
```

Files field

```php
$rm->migrate([
  'fields' => [
    'yourfilefield' => [
      'type' => 'file',
      'tags' => 'YourTags',
      'maxFiles' => 1,
      'descriptionRows' => 0,
      'extensions' => "pdf",
      'icon' => 'file-o',
      'outputFormat' => FieldtypeFile::outputFormatSingle,
    ],
  ],
]);
```

Options field

```php
$rm->migrate([
  'fields' => [
    'yourfield' => [
      'type' => 'options',
      'tags' => 'YourTags',
      'label' => 'Options example',
      'options' => [
        1 => 'ONE|This is option one',
        2 => 'TWO',
        3 => 'THREE',
      ],
    ],
  ],
]);
```

Options field with multilang labels:

```php
$rm->createField('demo_field', 'options', [
  'label' => 'Test Field',
  'label1020' => 'Test Feld',
  'type' => 'options',
  'optionsLang' => [
    'default' => [
      1 => 'VERYLOW|Very Low',
      2 => 'LOW|Low',
      3 => 'MIDDLE|Middle',
      4 => 'HIGH|High',
      5 => 'VERYHIGH|Very High',
    ],
    'de' => [
      1 => 'VERYLOW|Sehr niedrig',
      2 => 'LOW|Niedrig',
      3 => 'MIDDLE|Mittel',
      4 => 'HIGH|Hoch',
      5 => 'VERYHIGH|Sehr hoch',
    ],
  ],
]);
```

Note that RockMigrations uses a slightly different syntax than when populating the options via GUI. RockMigrations makes sure that all options use the values of the default language and only set the label (title) of the options.

Page Reference field

```php
$rm->migrate([
  'fields' => [
    'yourfield' => [
      'type' => 'page',
      'label' => __('Select a page'),
      'tags' => 'YourModule',
      'derefAsPage' => FieldtypePage::derefAsPageArray,
      'inputfield' => 'InputfieldSelect',
      'findPagesSelector' => 'foo=bar',
      'labelFieldName' => 'title',
    ],
  ],
]);
```

Date field

```php
$rm->migrate([
  'fields' => [
    'yourfield' => [
      'type' => 'datetime',
      'label' => __('Enter date'),
      'tags' => 'YourModule',
      'dateInputFormat' => 'j.n.y',
      'datepicker' => InputfieldDatetime::datepickerFocus,
      'defaultToday' => 1,
    ],
  ],
]);
```
