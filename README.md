# SqlMeUp

- [Master Build Status](https://travis-ci.org/ingenerator/sqlmeup.png?branch=master)](https://travis-ci.org/ingenerator/sqlmeup)

SqlMeUp is an opinionated database migrations tool, built from scratch to make it easier to manage database schema
versioning in distributed development and deployment workflows. It aims to have uses minimal external dependencies and
to be flexible enough to manage database provisioning for applications in any framework or language.

> **We practice Readme Driven Development. Not everything described in this readme actually works yet.
>   Read the [build report](https://travis-ci.org/ingenerator/sqlmeup) to see which features are implemented
>   so far, and if something you need is missing feel free to send a pull request!**

## tl;dr

Add SqlMeUp to your composer.json and run `composer update` to install it.

```json
{
  "require": { "sqlmeup/sqlmeup": "dev-master" }
}
```

Create a database configuration file in db_schema/sqlmeup_connect.json. This should be mode 0600 to keep it safe from
other users. Generally, you'll add db_schema/sqlmeup_connect.json to your .gitignore and deploy it with your
provisioning tool of choice.

```json
{
  "driver":   "pdo_mysql",
  "dbname":   "mydb",
  "user":     "root",
  "password": "secret",
  "host":     "localhost",
}
```

Create a new migration file with `vendor/bin/sqlmeup create-migration getting_started`. This will create an empty .sql
file in db_schema/migrations/{timestamp}_{username}_getting_started. Put some SQL in that file.

```sql
/**
 * Getting Started
 * @author <you>
 * Created at: Mon, Feb 2, 2014 14:30:00+0000
 */
CREATE TABLE test (
  id INT AUTO_INCREMENT NOT NULL,
  PRIMARY KEY(id))
  DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci ENGINE = InnoDB;
```

Run `vendor/bin/sqlmeup refresh-schema` to generate the schema tracking file.

Commit and push the new files.

Run `vendor/bin/sqlmeup migrate` to run your migration. The table will be created, as will the tables used by sqlmeup
to track status. You can safely run migrate on every instance in your cluster - the first process to run will run the
migrations and all others will wait until it has finished before continuing their deployment.

## Why?

We didn't find an existing database migrations tool that suited our needs. Problems in the tools we came across
included:

 * Unpredictable or downright broken behaviour when deploying migrations originally built in multiple development branches.
 * Poor conflict detection when working on migrations across multiple development branches.
 * Using main application database credentials, resulting in the web user having schema management privileges.
 * Forcing the user to handle scheduling and co-ordinating migrations between multiple machines during deployment.
 * Using abstraction layers and tool-specific code to define the migrations themselves, reducing readability.
 * Binding users to defining their full database schema in one or other database access libraries.

## What doesn't it support?

We don't support:

 * Using different database platforms in development and production, or writing abstract migrations that will
   automatically run on whatever database platform you decided to use today. To switch databases, you'll have to delete
   your old migrations and write a new one that creates whatever initial schema you want on the new system.

 * Rolling back. No, really. To undo a migration, you need to write and deploy a new migration that reverses it. If you
   can't write and deploy a fix fast enough, you'll need to restore a backup. You have them, right?

 * Doing fancy things with your ORM during a migration. Migrations themselves are written in and executed in SQL. *we
   may in future support running arbitrary shell tasks as part of applying a migration*

If you need to do these things, you should choose a different tool.

## What does it do then?

### Co-ordinates deployment of database migrations across a whole cluster

The `sqlmeup migrate` command checks which of your migrations have already been applied to the current database, and
executes the SQL commands required to bring it up to date. So far, so ordinary. But unlike others, you can run `sqlmeup
migrate` on **every instance in your cluster**. We track in-progress migrations with database locks - the first instance
to hit the migration command will run the migrations and **the other instances will wait patiently for it to finish**.
If for some reason the first instance dies without migrating, the next instance takes over and runs whatever migrations
are still required. Providing you sequence your deploy right, every instance fails before your new application code
is deployed.

Your application can use the same mechanism to detect whether a migration is in progress - making it dead simple to
implement an automatic maintenance page during migrations if you need to.

### Keeps your privileged database access secure

SqlMeUp runs in the shell and uses an entirely separate database connection to your application. You would usually
provide it with root or superuser database credentials - and can then grant only SELECT, DELETE, UPDATE etc to the
database user that your main application runs with.

### Causes merge conflicts between potentially conflicting migrations, and detects diverged production databases

Each time you build a migration, SqlMeUp produces a snapshot of the expected full database schema after is applied. If
someone creates a migration in another branch that touches the same columns or tables, the database schema file will
produce a merge conflict - alerting you to problems before they hit production.

After running migrations, `sqlmeup migrate` compares the resulting schema with the expected file and can be set to emit
a warning or fail - allowing you to ensure that all production database changes are properly tracked and applied.

### Co-ordinates deployment of breaking database changes with the deployment of your app

Often you need to make a change that is not compatible with the current deployed version of your app. For example, you
might want to rename a column, or drop one. You can often handle this by splitting the migration into two - one to add
a new column and copy the data, and another to tidy up and drop the old one. SqlMeUp makes this easy to handle - you can
add both migrations in the same commit, like this:

```sql
/**
 * Do something
 */
ALTER TABLE `foo` ADD `new_name` INT DEFAULT NULL;
UPDATE `foo` SET `new_name` = `old_name`;
```

```sql
/**
 * Do this later
 * @sqlmeup:delayed_migration
 */
ALTER TABLE `foo` DROP `old_name`;
```

SqlMeUp will detect the `@sqlmeup:delayed_migration` comment in the second migration file and instead of running it
immediately will schedule it to run the next time you run migrations. Delayed migrations will not run within 5 minutes
(by default) of being scheduled - to prevent another instance in the cluster running them before all instances have
deployed the new release. Your external deployment solution could run `sqlmeup migrate` again once it knows the deploy
is complete, or if you deploy regularly you can just leave them to be applied at the next deployment.

The migration above is still risky if your application writes to the `old_name` column - your old code will still be
live for a short period after you copy the values to `new_name` and you may lose data. In this case, there's no
alternative to taking your application offline while the migration is applied.

SqlMeUp can help here too. Define your migration like this:

```sql
/**
 * Unsafe migration
 * @sqlmeup:offline_migration
 */
ALTER TABLE `foo` CHANGE COLUMN `old_name` `new_name` INT DEFAULT NULL;
```

Somewhere in your application - your bootstrap perhaps, or a controller/router filter:

```php
$config = however_your_app_gets_its_db_config(); // do NOT use the root credentials!
$status = new SqlMeUp\MigrationStatus($connection_config);
if ($status->isReadyForCurrentDatabaseVersion()) {
  throw new Http_Exception_503(); // or however you want to show the user a maintenance page
}
```

As soon as your migrations start running, your app will take itself offline. As each instance deploys the new source
code into production, it will again start handling requests.

Note that this technique adds an additional (small, and almost certainly cached) SQL query to every pageload. Use with
caution on heavily loaded sites.

### Helps you build migrations whatever way suits you

By default, you'll write your migrations by hand. SqlMeUp provides a simple `sqlmeup refresh-schema` command that you
can run after adding a migration to update the snapshot of the database schema.

But you can also plug in tools to generate migrations from whatever ORM / database framework you're using. If your tools
need to diff against an actual database, SqlMeUp can provide them with a temporary database based on the last-committed
schema. This helps you be sure that your migrations aren't affected by any local changes to your developer database.

The first official migration generator will be for the Doctrine2 ORM, but it will be easy to write your own.

## Testing and developing

SqlMeUp has a full suite of [PhpSpec](http://phpspec.net) specifications - run them with `vendor/bin/phpspec run`.
Contributions will only be accepted if they are accompanied by well structured specs. Installing with composer should
get you everything you need to work on the project.

## License

SqlMeUp is copyright 2014 inGenerator Ltd and released under the [BSD license](LICENSE).
