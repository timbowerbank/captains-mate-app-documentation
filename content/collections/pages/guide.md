---
id: b4df6956-cfb8-459c-8e40-0c12c15ab391
blueprint: page
title: Guide
author: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1772814562
---
# Guide to using the documentation site

## Login 
* Login at /cp/auth/login
* Please note: we're using the free edition of Statamic, a limitation is that it only comes with one user.
* Please do not change the password without consulting with Adrian Lester at The Cruising Association.
* Once logged in navigate to /cp/dashboard/

## Creating a page
* Navigate to collections
* Select -> CAptains Mate App [version number], e.g. CAptains Mate App 4.3
* Select the three dot menu on the right of a collection category, e.g. Setup, or Providers
* Select **Create Child Entry**
* Give the page a title, e.g. Help Support Screen
* Add content to the content section - add content using Markdown
* Select **Save & Publish**

## Editing a child page
* Navigate to collections
* Select -> CAptains Mate App [version number], e.g. CAptains Mate App 4.3
* Select the title of the page you'd like to amend
* Edit the page
* Select **Save & Publish**

## Create a child navigation link
* Navigate to Navigation
* Select release version you require, e.g. CAptains Mate App 4.3
* Select the three dot menu on the right of the navigation category, e.g. Screens
* Select **Add child link to entry**
* Select the page you'd like to link to, e.g. Help Support Screen
* Select **Save Changes**

## Creating a new release
* This Statamic Starter Kit comes with a Laravel Artisan command line tool to scaffold new releases.
* Please see: [https://dok.fawnsoftware.com/dok/3.x/content-overview](https://dok.fawnsoftware.com/dok/3.x/content-overview)
* For this project we only need to add releases to the project CAptains Mate App.

## Git and GitHub

These files differ from local and production

```
modified:   app/Providers/AppServiceProvider.php
modified:   app/Tags/Dok.php
modified:   composer.lock
modified:   config/app.php
modified:   config/dok.php
modified:   content/collections/captains_mate_app_43.yaml
modified:   content/collections/captains_mate_app_43/flutter-packages.md
modified:   content/collections/captains_mate_app_43/flutter-version.md
modified:   content/collections/pages/docs.md
modified:   package-lock.json
modified:   public/.htaccess
modified:   resources/views/home.antlers.html
modified:   resources/views/layout.antlers.html
modified:   routes/web.php
```

### Production

* Content lives on production branch
* SSH setup to manually push to origin/production

### Local
* Local version deploys changes from main branch
* Merge from main into production