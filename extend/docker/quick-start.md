---
title: Docker Extension Quick Start
permalink: /extend/docker/quick-start/
---

* TOC
{:toc}

This tutorial guides you through the process of creating a simple Docker Application in PHP.
As in the [Custom Science Quick Start](/extend/custom-science/quick-start/), the application logic is trivial: it takes a table with numbers as an input, and creates another table with an extra column containing those numbers multiplied by two. A test in KBC is included.

## Before You Start

- Have a [KBC project](/#development-project) where you can test your code.
- Get yourself acquainted with [Docker](/extend/docker/tutorial/). You must be
able to [run `docker`](/extend/docker/tutorial/setup/) commands.
- You should be able to send API requests. Although you can use the [Apiary](https://apiary.io/) client console, we
recommend using [Postman](https://www.getpostman.com/) as it is
more convenient. If you do use Postman, you can [import a list](/overview/api/)
of [sample requests](https://www.getpostman.com/collections/87da6ac847f5edcac776).

## Step 1 -- Preliminaries

Create a public git repository ([Github](https://github.com/) or [Bitbucket](https://bitbucket.org/) is recommended, although any other host should work as well).

## Step 2 -- Write the Application Code

In the root of your repository, create a PHP script named
[`main.php`](https://github.com/keboola/docs-docker-example-basic/blob/master/main.php) with the following contents:

{% highlight php %}
<?php

$fhIn = fopen('/data/in/tables/source.csv', 'r');
$fhOut = fopen('/data/out/tables/destination.csv', 'w');

$header = fgetcsv($fhIn);
$numberIndex = array_search('number', $header);
fputcsv($fhOut, array_merge($header, ['double_number']));

while ($row = fgetcsv($fhIn)) {
    $row[] = $row[$numberIndex] * 2;
    fputcsv($fhOut, $row);
}

fclose($fhIn);
fclose($fhOut);
echo "All done";
{% endhighlight %}

As mentioned above, this script reads a CSV file, takes a column named
_number_, multiplies its values by 2 and adds the new values as a new column.
We take care to properly find the column index (`$numberIndex`), as it is not safe to rely on the order of columns.
Finally, the result is written to another CSV file. Note that we open both the input and output files simultaneously; as soon as a row is processed,
it is immediately written to _destination.csv_. This approach keeps only a single row of data in the memory and is
generally very efficient. There is no requirement to implement the processing in this way, but keep in mind that data files
coming from KBC can by quite large (i.e. Gigabytes).

You can test the code with our [sample table](/extend/source.csv):

number | someText | double_number
--- | --- | ---
10 | ab | 20
20 | cd | 40
25 | ed | 50
26 | fg | 52
30 | ij | 60


## Step 3 -- Wrap the Application in a Docker Image
You need to create a Docker Image containing and running your application.

### Step 3.1 -- Wrap the Application in an Image
Create a file named
[`Dockerfile`](https://github.com/keboola/docs-docker-example-basic/blob/master/Dockerfile) in the root of the repository:

    FROM quay.io/keboola/docker-base-php56:0.0.2
    COPY . /home/
    ENTRYPOINT php /home/main.php

The image inherits from our [Keboola PHP56 base image](https://quay.io/repository/keboola/docker-base-php56),
which is defined by its own [Dockerfile](https://github.com/keboola/docker-base-php56/blob/master/Dockerfile).
The instruction `COPY . /home/` copies the application code (only the `main.php` file in this simple application)
from the *build context* (the same folder in which the Dockerfile resides) into the image.
The `ENTRYPOINT` line specifies that when the image is run, the PHP application script is executed.

The Dockerfile and the application can be in two separate repositories or in a
single [git repository](https://github.com/keboola/docs-docker-example-basic). Using a single repository makes
things generally easier, but it is not required.

### Step 3.2 -- Build the Image
On the command line, navigate to the folder with your repository and run the following command (including the dot at the end):

    docker build --tag=test .

It should produce output similar to the one below:

{: .image-popup}
![Docker build output](/extend/docker/build-output.png)

Out of that output, the most important thing is the *Successfully built ded5321d5ba5* message which gives us a
hash of the new image: `ded5321d5ba5`. It can be abbreviated to the first three characters, so we can
later refer to it as `ded`. We can also use the tag of the image: `test`.

## Step 4 -- Obtaining Sample Data and Configuration
Data between KBC and your Docker image are exchanged using [CSV files](/extend/common-interface/); they will be
injected into the image when we run it. To simulate this, download an archive containing the data files
and [configuration](/extend/common-interface/config-file/) in the exact same format as you will obtain it
in the production environment.

To obtain the configuration, send a [Sandbox API Request](/extend/common-interface/sandbox/). You will receive an
archive containing a [/data/ folder](/extend/common-interface/) with tables and files from the input mapping, and a
configuration depending on the request body. A sample request to `https://syrup.keboola.com/docker/sandbox?format=json`:

{% highlight json %}
{
    "config": "my-test-config",
    "configData": {
        "storage": {
            "input": {
                "tables": [
                    {
                        "source": "in.c-main.test",
                        "destination": "source.csv"
                    }
                ]
            },
            "output": {
                "tables": [
                    {
                        "source": "destination.csv",
                        "destination": "out.c-main.test"
                    }
                ]
            }
        },
        "parameters": {
        }
    }
}
{% endhighlight %}

The sample request corresponds to the following setting in the UI (though the UI for your component will become
available only after your extension has been completed and [registered](/extend/registration/)).

{: .image-popup}
![Configuration Screenshot](/extend/docker/configuration-sample.png)

Alternatively -- to quickly get the picture, download a [random sample data folder](/extend/docker/data.zip),
 which can be used together with the above [sample application](https://github.com/keboola/docs-docker-example-basic).

## Step 5 -- Running the Application with Sample Data
Once you have prepared the data folder with sample data and configuration, inject it into the Docker Image.
In addition to the options shown in the example, there are many [other options](/extend/common-interface/config-file/) available.

When you run an image, a *container* is created in which the application is running isolated.
Use the following command to run the image:

    docker run --volume=physicalhostpath:/data/ imageTag

An Image tag can be either the tag you supplied in the `--tag` parameter for `docker build` or the image hash you received
when the image was build (`ded` in the above example).
The physical host path depends on the system you are running. If in doubt,
see [Setting up Docker](/extend/docker/tutorial/setup/#sharing-files). In our example image with default Windows installation of Docker, this would be:

    docker run --volume=/c/Users/ondre/data/:/data/ test

Where the contents of the sample data folder are put in the user's home directory. If you have set everything correctly,
you should see **All done**; and a `destination.csv` file will appear in the `data/out/tables/` folder.

### Step 5.1 -- Debugging

Chances are, that you received an ugly error or warning. In that case, you might want to check out the
contents of the image; specifically, if all the files are where you expect
them to be -- see [debugging](/extend/docker/running/).

To work with the application container interactively, use the following command:

    docker run --volume=physicalhostpath:/data/ -i -t --entrypoint=/bin/bash imageTag

For instance:

    docker run --volume=/c/Users/JohnDoe/data:/data/ -i -t --entrypoint=/bin/bash test

You can then inspect the container contents: 'ls /data/'. For more details, see [Howto](/extend/docker/running/).


### Step 6 -- Deployment

It is best to use a Docker registry and set up [automated builds](/extend/docker/tutorial/automated-build/) for deployment. To
be usable in KBC, the image must be available in one of the supported registries ([Dockerhub](https://hub.docker.com/), [Quay](https://quay.io/)) or [AWS ECR](https://aws.amazon.com/ecr/).
We support both public and private images.
AWS ECR supports only private images and we will host your image and prodvide you with push credentials. 
Contact us at [support@keboola.com](mailto:support@keboola.com) to obtain AWS ECR credentials for your image.

To deploy the application to production, it must first be [registered](/extend/registration/). Once the
application is registered with us, we will automatically pull the image and make it available in production.
There are two modes for deployment:

- automatic: use the *latest* or *master* tags on the Docker Image. If you commit a new code to your application
repository and [(automatically) rebuild] the image, the next time your application is run,
it will be updated to the latest version.
- manual: use [Semantic versioning](http://semver.org/) versioning tags on your Docker Images.
Let us know when you want to change the image tag to a new version.

There is no need to specify the deployment mode; it is fully determined by the type of tags you use.
At the beginning, it is probably more straightforward to work in the automatic mode because your deployment is
fully automated and requires no interaction with us. Once the application gets more mature, you should probably
switch to manual versioning and perhaps notify your users about any modifications.


