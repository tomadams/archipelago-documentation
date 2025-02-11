---
title: "Archipelago-deployment-live: upgrading Drupal 8 to Drupal 9 (1.0.0-RC2 to 1.0.0-RC3)"
tags:
  - Archipelago-deployment-live
  - Drupal 8
  - Drupal 9
---

# Archipelago-deployment-live: upgrading Drupal 8 to Drupal 9 (1.0.0-RC2 to 1.0.0-RC3)

## What is this documentation for?

If you already have a well setup and well loved Archipelago (1.0.0-RC2) running Drupal 8 (D8), this documentation will allow you to update to 1.0.0-RC3 on Drupal 9 (D9) without major issues.

D8 will no longer be supported by the end of November 2021. D9 has been around for a little while, and even if every module is not supported yet, what you need and want for **Archipelago** has long been ready for D9. However, Archipelago is still D8 compatible if it's necessary for you to stay back a little longer.

## Requirements

- An [archipelago-deployment-live](https://github.com/esmero/archipelago-deployment-live) instance 1.0.0-RC2 (working, tested) deployed using provided instructions via Docker and running Drupal 8.
- Basic knowledge and instincts (+ courage) on how to run Terminal Commands, `composer` and `drush`.
- Patience. You can't skip steps here.
- For Shell Commands documented here please copy line by line—not the whole block.

## Backing up and preparing for the upgrade

Backups are always going to be your best friends. Archipelago's code, database and settings are mostly self-contained in your current `archipelago-deployment-live` repo folder, and backing up is simple because of that.

### Step 1:

Shutdown your `docker-compose` ensemble. Move to your `archipelago-deployment-live` folder and run this:

```Shell
cd deploy/ec2-docker
docker-compose down
```

### Step 2:

Verify all containers are actually down. The following command should return an empty listing. If anything is still running, wait a little longer and run the following comman again.

```Shell
docker ps
```

### Step 3:

Now let's `tar.gz` the whole ensemble with data and configs. As an example we will save this into your `$HOME` folder. 
As a good practice we append the **current date **(YEAR-MONTH-DAY) to the filename. Here we assume today is December 1st of 2021.

```Shell

sudo tar -czvpf $HOME/archipelago-deployment-live-backup-20211201.tar.gz ../../../archipelago-deployment-live

```

The process may take a few minutes. Now let's verify all is there and the `tar.gz` is not corrupt.

```Shell
tar -tvvf $HOME/archipelago-deployment-live-backup-20211201.tar.gz 
```

You will see a listing of files. If corrupt (do you have enough space? did your ssh connection drop?) you will see the following:

```Shell
tar: Unrecognized archive format
```

### Step 4:

Restart your `docker-compose` ensemble and wait a little while for all to start.

```Shell
docker-compose up -d
```

### Step 5:

Export/backup all of your live Archipelago configurations (this allows you to compare/come back in case you lose something custom during the upgrade).

```Shell
docker exec esmero-php mkdir config/backup
docker exec esmero-php drush cex --destination=/var/www/html/config/backup
```

Good. Now it's safe to begin the upgrade.

___

## Upgrading to 1.0.0-RC3

### Step 1:

First we are going to disable modules that are not part of 1.0.0-RC3 or are not yet compatible with D9. Run the following command:

```Shell
docker exec esmero-php drush pm-uninstall module_missing_message_fixer markdown webprofiler key_value webform_views
```

From inside your `archipelago-deployment-live` repo folder we are going to open up the file `permissions` for some of your most protected Drupal files.

```Shell
cd ../../
sudo chmod 777 drupal/web/sites/default
sudo chmod 666 drupal/web/sites/default/*settings.php
sudo chmod 666 drupal/web/sites/default/*services.yml
```

### Step 2:

Time to fetch the `1.0.0-RC3` branch and update our `docker-compose` and `composer` dependencies. We are also going to stop the current `docker` ensemble to update all containers to newer versions:

```Shell
cd deploy/ec2-docker
docker-compose down
git checkout 1.0.0-RC3 
```

Then copy the appropriate `docker-compose` file for your architecture:

??? info "macOS/x86-64"

    ```Shell
    cp docker-compose-osx.yml docker-compose.yml
    ```

??? info "Linux/x86-64/AMD64"

    ```Shell
    cp docker-compose-linux.yml docker-compose.yml
    ```

??? info "macOS/Linux/ARM64"

    ```Shell
    cp docker-compose-arm64.yml docker-compose.yml
    ```

Finally, pull the images, and bring up the ensemble:

```Shell
docker compose pull 
docker compose up -d
```

Give all a little time to start. The latest `min.io` adds a new console, and your `Solr` core and `Database` need to be upgraded. Please be patient. To ensure all is well, run (more than once if necessary) the following:

```Shell
docker ps
```

You should see something like this: 

```Shell
CONTAINER ID   IMAGE                                    COMMAND                  CREATED          STATUS          PORTS                                                           NAMES
867fd2a42134   nginx                                    "/docker-entrypoint.…"   32 seconds ago   Up 27 seconds   0.0.0.0:8001->80/tcp, :::8001->80/tcp                           esmero-web
8663e84a9b48   solr:8.8.2                               "docker-entrypoint.s…"   33 seconds ago   Up 30 seconds   0.0.0.0:8983->8983/tcp, :::8983->8983/tcp                       esmero-solr
9b580fa0088f   minio/minio:latest                       "/usr/bin/docker-ent…"   33 seconds ago   Up 28 seconds   0.0.0.0:9000-9001->9000-9001/tcp, :::9000-9001->9000-9001/tcp   esmero-minio
50e2f41c7b60   esmero/esmero-nlp:1.0                    "/usr/local/bin/entr…"   33 seconds ago   Up 30 seconds   0.0.0.0:6400->6400/tcp, :::6400->6400/tcp                       esmero-nlp
300810fd6f03   esmero/cantaloupe-s3:4.1.9RC             "sh -c 'java -Dcanta…"   33 seconds ago   Up 30 seconds   0.0.0.0:8183->8182/tcp, :::8183->8182/tcp                       esmero-cantaloupe
248e4638ba2a   mysql:8.0.22                             "docker-entrypoint.s…"   33 seconds ago   Up 28 seconds   3306/tcp, 33060/tcp                                             esmero-db
141ace919344   esmero/php-7.4-fpm:1.0.0-RC2-multiarch   "docker-php-entrypoi…"   33 seconds ago   Up 28 seconds   9000/tcp                                                        esmero-php
```

Important here is the `STATUS` column. It **needs** to be a number that goes up in time every time you run `docker ps` again (and again).

### Step 3:

Now we are going to tell `composer` to actually fetch the new code and dependencies using the 1.0.0-RC3 provided `composer.lock` and update the whole Drupal/PHP/JS environment.

```Shell
docker exec -ti esmero-php bash -c "composer install"
```

This will fail (sorry!) for a few packages but no worries, they need to be patched and composer is not that smart. So simply run it again:

```Shell
docker exec -ti esmero-php bash -c "composer install"
```
 
Well done! If you see **no** issues and all ends in a **Green colored message** all is good! [Jump to Step 4](#step-4_1)

#### What if all is not OK, and I see red and a lot of dependency explanations?

If you have manually installed packages via composer in the past that are NO longer Drupal 9 compatible you may see errors. 
In that case you need to check each package website's (normally https://www.drupal.org/project/the_module_name) and check if there is a Drupal 9 compatible version. 

If so run:

```Shell
docker exec -ti esmero-php bash -c "composer require 'drupal/the_module_name:^VERSION_NUMBER_THAT_WORKS_ON_DRUPAL9_' --update-with-dependencies --no-update" and run **Step 3 ** again (and again until all is cleared)
```

If not: try to find a replacement module that does something simular, but in any case you may end having to remove before proceding. Give us a ping/slack/google group/open a github ISSUE if you find yourself uncertain about this. 

```Shell
docker exec -ti esmero-php bash -c "composer remove drupal/the_module_name --no-update"
docker exec -ti esmero-php bash -c " drush pm-uninstall the_module_name"
```

### Step 4:

We will now ask Drupal to update some internal configs and databases. They will bring you up to date with RC3 settings and D9 particularities.

```Shell  
docker exec -ti esmero-php bash -c 'scripts/archipelago/setup.sh'
docker exec -ti esmero-php bash -c 'drush urol administrator "admin"'
docker exec -ti esmero-php bash -c "drush updatedb"
```

### Step 5:

Previously D8 installations had a "module/profile" driven installation. Those are no longer used or even exist as part of core, but a profile can't be changed once installed so you have to do the following to avoid Drupal complaining about our new and simpler way of doing things (a small roll back):

```Shell 
docker exec -ti esmero-php bash -c "sed -i 's/minimal: 1000/standard: 1000/g' config/sync/core.extension.yml"
docker exec -ti esmero-php bash -c "sed -i 's/profile: minimal/profile: standard/g' config/sync/core.extension.yml"
```

### Step 6:

Now you can Sync your new Archipelago 1.0.0-RC3 and bring all the new configs and settings in. For this you have **two** options (no worries, remember you made a backup!):

#### A Partial Sync, which will bring new configs and update existing ones but will **not** remove ones that only exist in your custom setup, e.g. new Webforms or View Modes.

```Shell 
docker exec esmero-php drush cim -y --partial   
```

#### A Complete Sync, which will bring new configs and update existing ones but will also remove all the ones that are not part of RC3. It's a like clean factory reset.

```Shell 
docker exec esmero-php drush cim -y 
```

If all goes well here and you see no errors it's time to reindex `Solr` because there are new Fields. Run the following:

```Shell 
docker exec esmero-php drush search-api-reindex
docker exec esmero-php drush search-api-index 
```

You might see some warnings related to modules dealing with previously non-existent data—no worries, just ignore those.

If you made it this far you are done with code/devops (are we ever ready?), and that means you should be able to (hopefully) stay in the Drupal 9 realm for a few years!

### Step 7: Update (or not) your Metadata Display Entities and Menu items.

Recommended: If you want to add new templates and menu items 1.0.0-RC3 provides, run this:

```Shell
docker exec -ti esmero-php bash -c 'scripts/archipelago/deploy.sh'
```

Once that is done, you can choose to update all Metadata Displays (Twig templates) we ship with new 1.0.0-RC3 versions (heavily fixed IIIF manifest, Markdown to HTML for Metadata, better Object descriptions). But before you do this, we **really** recommend that you first make sure to manually (copy/paste) backup any Twig templates you have modified. If unusure, do not run the command that comes after this warning! You can always manually copy the new templates from the `d8content/metadatadisplays` folder which contains text versions (again, copy/paste) of each shipped template you can pick and use when you feel ready. 

If you are sure (like really?) you want to overwrite the ones you modified (sure, just checking?), then you can run this:

```Shell
docker exec -ti esmero-php bash -c 'scripts/archipelago/update_deployed.sh'
```

Done! (For realz now) 

Please log into your Archipelago and test/check all is working! Enjoy 1.0.0-RC3 and Drupal 9. Thanks!

___

Thank you for reading! Please contact us on our [Archipelago Commons Google Group](https://groups.google.com/forum/#!forum/archipelago-commons) with any questions or feedback, or open an ISSUE in this [Archipelago Deployment Live Repository](https://github.com/esmero/archipelago-deployment-live/).

Return to [Archipelago Live Deployment](archipelago-deployment-live.md).
