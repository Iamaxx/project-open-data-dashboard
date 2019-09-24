[![CircleCI](https://circleci.com/gh/GSA/project-open-data-dashboard.svg?style=svg)](https://circleci.com/gh/GSA/project-open-data-dashboard)

The Project Open Data Dashboard provides a variety of tools and capabilities to help manage the implementation of [Project Open Data](https://project-open-data.cio.gov/). It is primary used for Federal agencies, but also provides tools and resources for use by other entities like state and local government. 

The primary place for the user-facing documentation is https://labs.data.gov/dashboard/docs

## Features

* **[Dashboard overview](https://labs.data.gov/dashboard/offices)** of the status of each federal agency's implementation of [Project Open Data](https://project-open-data.cio.gov/) for each milestone.
* **Permissioned Content Editing** for the fields in the dashboard that can't be automated. The fields are stored as JSON objects so the data model is very flexible and can be customized without database changes. User accounts are handled via Github.
* **Automated crawls** for each agency to report metrics from Project Open Data assets (data.json, digitalstrategy.json, /data page, etc). This includes reporting on the number of data sets and validation against the Project Open Data metadata schema. 
* **A [validator](https://labs.data.gov/dashboard/validate)** to validate Project Open Data data.json files via URL, file upload, or text input. This can be used for testing both data.json Public Data Listing files as well as the Enterprise Data Inventory JSON. The validator can be used both by Federal agencies as well as non-federal entities by specifying the Non-Federal schema. 
* **Converters** to transform a [CSV into a data.json](https://labs.data.gov/dashboard/datagov/csv_to_json) file or to [export](https://labs.data.gov/dashboard/export) existing data from Data.gov
* **[Changeset viewer](https://labs.data.gov/dashboard/changeset)** to see how a data.json file for an agency compares to metadata currently hosted on data.gov

## CLI Interface

In addition to the web interface, there's also a Command Line Interface to manage the crawls of data.json, digitalstrategy.json, and /data pages. This is helpful to run specific updates, but it's primary use is with a CRON job. 

From the root of the application, you can update the status of agencies using a few different options on the `campaign` controller. The syntax is:

`$ php index.php campaign status [id] [component]`

If you wanted to update all components (data.json, digitalstrategy.json, /data) for all agencies, you'd run this command:

`$ php index.php campaign status all all`

If you just wanted to update the data.json status for CFO Act agencies you'd run:

`$ php index.php campaign status cfo-act datajson`

If you just wanted to update the digitalstrategy.json status for the Department of Agriculture you'd run:

`$ php index.php campaign status 49015 digitalstrategy`

The options for [id] are: `all`,`cfo-act`, or the ID provided by the [USA.gov Federal Agency Directory API](http://www.usa.gov/About/developer-resources/federal-agency-directory/)

The options for [component] are: `all`, `datajson`, `datapage`, `digitalstrategy`, `download`. 

* The `datajson` component captures the basic characteristics of a request to an agency's data.json file (like whether it returns an HTTP 200) and then attempts to parse the file, validate against the schema, and provide other reporting metrics like the number of datasets listed. 
* The `digitalstrategy` component captures the basic characteristics of a request to an agency's digitalstrategy.json file (like whether it returns an HTTP 200) 
* The `datapage` component captures the basic characteristics of a request to an agency's /data page (like whether it returns an HTTP 200)
* The `download` component downloads an archive of the data.json and digitalstrategy.json files
* As you'd expect, `all` does all of these things at once. 

## Development

This is a [CodeIgniter](http://www.codeigniter.com/) PHP application. We use Docker and Docker compose for local development and cloud.gov for testing and production (pending migration from BSP.)

Prerequisites:

* [Docker Engine](https://docs.docker.com/install/) v18+
* [Docker Compose](https://docs.docker.com/compose/install/) v1.24+

### Setup

Install dependencies

    bin/composer install --no-dev

Start up docker containers.

    docker-compose up

Test with bats (and run the migrations):

    docker-compose exec app bats -r tests/

Open your browser to [localhost:8000](http://localhost:8000/).

### CircleCI testing

All pushes to GitHub are integration tested with our [CircleCI tests](https://circleci.com/gh/GSA/project-open-data-dashboard).

### Updating composer dependencies

Edit version constraints in [composer.json](./composer.json).

    bin/composer update

Commit the updated composer.json and composer.lock.


## Deploying to cloud.gov

### Quickstart

Assuming you're logged in for the Cloud Foundry CLI:

```sh
cf create-service aws-rds shared-mysql database

cf create-user-provided-service secrets -p '{
  "ENCRYPTION_KEY": "long-random-string"
}'

cf push
```

To do the initial migration to populate the database:

```
cf ssh app -c '/tmp/lifecycle/launcher "app" "php htdocs/index.php migrate" ""'
```

You should be able to visit https://&lt;ROUTE&gt;/offices/qa, where &lt;ROUTE&gt; is the route reported from `cf push`:

<pre><code>
Waiting for app to start...

name:              app
requested state:   started
routes:            <b><u>app-boring-sable.app.cloud.gov</u></b>
last uploaded:     Wed 28 Aug 10:02:06 EDT 2019
stack:             cflinuxfs3
buildpacks:        php_buildpack

type:            web
instances:       1/1
memory usage:    256M
start command:   $HOME/.bp/bin/start
     state     since                  cpu    memory          disk             details
#0   running   2019-08-28T14:02:25Z   0.3%   24.3M of 256M   301.7M of 512M
</code></pre>

## Known issues

The agency hierarchy is designed to be populated from the `contacts` API at https://www.usa.gov/api/USAGovAPI/contacts.json/contact, but that is no longer available, so these
following steps no longer work:

> * Federal agencies were seeded using the [USA.gov Federal Agency Directory API](http://www.usa.gov/About/developer-resources/federal-agency-directory/) and the IDs provided by that resource are used as the primary IDs on this dashboard. 
> * First populate the top of the agency hierarchy: `$ php index.php import`
> * Second, populate all the subagencies: `$ php index.php import children`
> * If you have an empty database `offices` table in the database, you'll also want to seed it with agency data by running the import script (`/application/controllers/import.php`) from a command line. You'll also need to temporarily change the `import_active` option in config.php to `true`


Currently this tool does not handle large files in a memory efficient way. If you are unable to utilize a high amount of memory and are at risk of timeouts, you should set the maximum file size that the application can handle so it will avoid large files and fail more gracefully. The maximum size of JSON files to parse can be set with the `max_remote_size` option in config.php

# Database restore

## docker compose:

```
cat downloads/cleaned_database.sql | docker-compose run   --rm   database   mysql --host=database --user=root --password=mysql dashboard
```



# What about S3?

S3 is used in a few places: 
- for put/fetch of csv_to_json
- for archiving data.json and digitalstrategy

The `archive_file` function will use config[use_local_storage] anytime it's called
but the logic doesn't apply when to `datajson_lines`. Talk about poor encapsulation.

It's also used in the office_detail view. 

This will require 2 buckets, since they have different ACLs attached.

Also, not all uploads use S3, or all use of archive.

- campaign.php
  - `public function csv_to_json($schema = null)`
    - once to PUT the file
    - again to GET the file
    - There is no remove from S3
  - We archive digitalstrategy.json and data.json using `archive_file` which puts a fetch date in the URL.
- campaign_model has 
  - `archive_file` which calls  `archive_to_s3
  - `archive_to_s3` which calls put_to_s3 and stores with a PUBLIC acl
  - `put_to_s3` which stores private by default
- views/office_detail
  - Builds a URL based on values of `config/s3_bucket`