This document shows the result of a few weeks long journey of finding the best solution to automatically deploy a project that's being hosted on GitHub to Amazon Web Services. This is not a theoretical document, but a guide that aims to help you deploy your code faster and more frequent.

I'm not a devOps guy. I'm a developer that did go through some pain to deploy his code in a painless and automated way to free up time for more interesting things (playing ping pong with co-workers, more coding, working on stuff that may or may not involve drones).

First, we will build a small test application, an API really. I'm using the nodejs framework [hapi](http://hapijs.com/) here and connect it to a [Neo4j](http://neo4j.com/) database which is hosted for free on [Graph Story](http://www.graphstory.com/).

For a great build experience, I've signed up for a Startup Account at [Shippable](https://www.shippable.com/) which is at the time of this writing $12/year.

I'm building the application image on a dedicated t2.small AWS EC2 instance.

For the actual deployment of the different application environments (just development and production here) we're utilizing AWS ElasticBeanstalk.

If everything works out, a docker image gets created and pushed to [Docker Hub](https://hub.docker.com/), either as a public or a private repository.


Setting this infrastructure up takes a little bit of time, but the big payoff is that a commit to the master branch will result in the deployment to the production environment, while a commit to the develop branch will result in the deployment to the development environment.

Say good-bye to manual deployments.

## The Application

### The Database

While I could fill pages talking about how just awesome Neo4j is, I'm just letting you know that you have to take a look at this database. Graph databases are not just for social networks. For the sake of simplicity, the application is not going to utilize any of the great benefits of Neo4j or graph databases in general. The focus of this post is after all on deploying software and not how to design and write it.

After signing up and logging into the free [Graph Story](http://www.graphstory.com/) account, you should log create a database and click on the Neo4j Web UI button to access the built-in web interface of Neo4j.
![graph story dashboard](https://s3-us-west-2.amazonaws.com/deployment-article/images/application_the_database_web_ui.png).

Once we're in the web interface, let's create some nodes (or edges, as they're more commonly called in graph theory) with cypher.

```cypher
CREATE (u:User {name: 'Matthias Sieber', email: 'matthiasksieber@gmail.com'}),(v:User {name: 'Test User', email: 'test@example.com', isFake: true}) RETURN u,v
```

This query will create two nodes with a user label and some properties. We're also returning those nodes, so you'll have something to look at.

![created user nodes](https://s3-us-west-2.amazonaws.com/deployment-article/images/application_the_database_create_users.png)

And that's actually all the data we're creating now.

On to our hapijs app.

## The API

We will call our nodejs application Epione. It's become a practice at the companies I'm working at to name our projects after gods, spirits and other mythological beings. According to [Wikipedia](http://en.wikipedia.org/wiki/Epione), Epione was the goddess of soothing of pain.

As with any node application, I'm starting with a new directory and the package.json. Here are the contents for this sample application.

```json
{
  "name": "epione",
  "private": true,
  "version": "0.0.1",
  "repository": {
    "type": "git",
    "url": "git://github.com/PiichMe/epione.git"
  },
  "description": "Epione is the goddess of soothing of pain.",
  "author": "Matthias Sieber <matthiasksieber@gmail.com>",
  "dependencies": {
    "boom": "^2.7.0",
    "hapi": "^8.4.0",
    "joi": "~6.1.0",
    "request-promise-json": "^1.0.4"
  },
  "devDependencies": {
    "better-console": "^0.2.4",
    "chai": "^2.2.0",
    "gulp": "^3.8.11",
    "gulp-env": "^0.2.0",
    "gulp-lab": "^1.0.5",
    "gulp-nodemon": "^2.0.2",
    "lab": "^5.5.1"
  },
  "engines": {
    "node": "0.12"
  },
  "scripts": {
    "test": "gulp test",
    "start": "gulp serve"
  }
}
```

Since this is a rather simple application, I'm going to put all the server logic into the server.js.

```javascript,linenums=true
'use strict';

var Hapi = require('hapi');
var Boom = require('boom');
var rp = require('request-promise-json');
var constants = require('src/config/constants.js');
var host = constants.application.host;
var port = constants.application.port;
var commitURL = constants.database.commitUrl;
var server = new Hapi.Server({
  connections: {
    routes: {
      cors: {
        origin: ['*'] // to allow API requests from our front end later
      }
    }
  }
});

server.connection({ host: host, port: port });

server.route({
  method: 'GET',
  path: '/realusers',
  handler: function(request, reply) {
    var query = 'MATCH (user:User) WHERE NOT HAS (user.isFake) RETURN user';
    var options = {
      uri: commitURL,
      method: 'POST',
      body: { 'statements': [{'statement': query }] }
    };
    return rp.request(options).then(function(result) {
      if (result.results.length > 0) {
        return reply(result.results[0].data);
      } else {
        return reply(Boom.notFound());
      }
    });
  }
});

server.route({
  method: 'GET',
  path: '/fakeusers',
  handler: function(request, reply) {
    var query = 'MATCH (user:User) WHERE HAS(user.isFake) RETURN user';
    var options = {
      uri: commitURL,
      method: 'POST',
      body: { 'statements': [{'statement': query }] }
    };
    return rp.request(options).then(function(result) {
      if (result.results.length > 0) {
        return reply(result.results[0].data);
      } else {
        return reply(Boom.notFound());
      }
    });
  }
});

if (!module.parent) {
  server.start(function() {
    console.log('Server running at: ', server.info.uri);
  });
}

module.exports = server;
```

If you take a look at the source code, you might have noticed that we're referencing a constants.js. This is a configuration file that's especially beneficial when we're developing locally with different environments. That way we can store environment variables in our ~/.zshenv or wherever you store your environment variables of your shell of choice.

Create this file in src/config/constants.js:

```javascript,linenums=true
'use strict';

module.exports = (function() {

  var env = process.env.NODE_ENV || 'development';

  var databaseConfig = function() {
    return {
      'production': {
        'protocol': process.env.EPIONE_DB_PROTOCOL,
        'host': process.env.EPIONE_DB_HOST,
        'user': process.env.EPIONE_DB_USER,
        'password': process.env.EPIONE_DB_PASS,
        'port': process.env.EPIONE_DB_PORT
      },
      'development': {
        'protocol': process.env.EPIONE_DEVELOPMENT_DB_PROTOCOL,
        'host': process.env.EPIONE_DEVELOPMENT_DB_HOST,
        'user': process.env.EPIONE_DEVELOPMENT_DB_USER,
        'password': process.env.EPIONE_DEVELOPMENT_DB_PASS,
        'port': process.env.EPIONE_DEVELOPMENT_DB_PORT
      },
      'test': {
        'protocol': process.env.EPIONE_TEST_DB_PROTOCOL,
        'host': process.env.EPIONE_TEST_DB_HOST,
        'user': process.env.EPIONE_TEST_DB_USER,
        'password': process.env.EPIONE_TEST_DB_PASS,
        'port': process.env.EPIONE_TEST_DB_PORT
      }

    };
  };

  var applicationConfig = function() {
    return {
      'production': {
        'url': 'http://' + process.env.EPIONE_NODE_HOST + ':' + process.env.EPIONE_NODE_PORT,
        'host': process.env.EPIONE_NODE_HOST,
        'port': process.env.EPIONE_NODE_PORT
      },
      'development': {
        'url': 'http://' + process.env.EPIONE_DEVELOPMENT_NODE_HOST + ':' + process.env.EPIONE_DEVELOPMENT_NODE_PORT,
        'host': process.env.EPIONE_DEVELOPMENT_NODE_HOST,
        'port': process.env.EPIONE_DEVELOPMENT_NODE_PORT
      },
      'test': {
        'url': 'http://' + process.env.EPIONE_TEST_NODE_HOST + ':' + process.env.EPIONE_TEST_NODE_PORT,
        'host': process.env.EPIONE_TEST_NODE_HOST,
        'port': process.env.EPIONE_TEST_NODE_PORT
      }
    };
  };

  var dbConstants = databaseConfig();
  var appConstants = applicationConfig();

  var obj = {
    application: {
      url: appConstants[env].url,
      host: appConstants[env].host,
      port: appConstants[env].port
    },
    database: {
      host: dbConstants[env].host,
      user: dbConstants[env].user,
      password: dbConstants[env].password,
      port: dbConstants[env].port,
      commitUrl: dbConstants[env].protocol + '://' + dbConstants[env].user + ':' + dbConstants[env].password + '@' +
                 dbConstants[env].host + ':' + dbConstants[env].port +
                 '/db/data/transaction/commit'
    },
    server: {
      defaultHost: 'http://localhost:8001'
    }
  };

  if (!obj.application.host) {
    throw new Error('Missing constant application.host. Check your environment variables.');
  } else if (!obj.application.port) {
    throw new Error('Missing constant application.port. Check your environment variables.');
  } else if (!obj.database.host) {
    throw new Error('Missing constant database.host. Check your environment variables.');
  } else if (!obj.database.port) {
    throw new Error('Missing constant database.port. Check your environment variables.');
  } else if (!obj.database.user) {
    throw new Error('Missing constant database.user. Check your environment variables.');
  } else if (!obj.database.password) {
    throw new Error('Missing constant database.password. Check your environment variables.');
  }

  return obj;
}());
```

Since we only want to deploy our code when all the tests pass, we need to create some tests first.

Create a simple test for each of our two end points in test/api/user.js:
```javascript,linenums=true
'use strict';

var Lab = require('lab');
var lab = exports.lab = Lab.script();
var server = require('server');
var assert = require('chai').assert;

lab.experiment('Email/pw authentication', function() {
  lab.test('Returns real users', function(done) {
    var options = {
      method: 'GET',
      url: '/realusers',
    };
    server.inject(options, function(response) {
      assert.equal(response.statusCode, 200);
      var result = JSON.stringify(response.result);
      var expected = JSON.stringify([{"row":[{"name":"Matthias Sieber","email":"matthiasksieber@gmail.com"}]}]);
      assert.strictEqual(expected, result);
      done();
    });
  });
  lab.test('Returns fake users', function(done) {
    var options = {
      method: 'GET',
      url: '/fakeusers',
    };
    server.inject(options, function(response) {
      assert.equal(response.statusCode, 200);
      var result = JSON.stringify(response.result);
      var expected = JSON.stringify([{"row":[{"name":"Test User","email":"test@example.com","isFake":true}]}]);
      assert.strictEqual(expected, result);
      done();
    });
  });
});
```

Now we need a nice way to test the app and serve it as well. We're using gulp for this. In your project root, create a gulpfile.js:

```javascript,linenums=true
'use strict';

var gulp = require('gulp');
var env = require('gulp-env');
var nodemon = require('gulp-nodemon');
var lab = require('gulp-lab');
var betterConsole = require('better-console');

gulp.task('test', ['set-test-env', 'set-node-path'], function() {
  return gulp.src([
      'test/**/*.js',
      '!./{node_modules,node_modules/**}'
    ], { read: false })
      .pipe(lab({
        args: '-c -t 85',
        opts: {
          emitLabError: true
        }
      }));
});

gulp.task('set-test-env', function() {
  return env({
      vars: {
        NODE_ENV: 'test'
      }
    });
});

gulp.task('set-node-path', function() {
  return env({
      vars: {
        NODE_PATH: '.'
      }
    });
});

gulp.task('serve', function() {
  env({
    vars: {
      NODE_PATH: '.',
      NODE_ENV: 'development'
    }
  });

  nodemon({
    script: 'server.js',
    ext: 'js',
    nodeArgs: ['--debug']
  });
});
```

And that's it for now with the application.

Now run `npm install` to install all the dependencies.
Save your graph story neo4j environment variables into your ~/.zshenv (don't forget to source it afterwards) or prepend these to `gulp test` like this:

    EPIONE_TEST_DB_PROTOCOL='https' EPIONE_TEST_DB_HOST='neo-yourhost-from-graph-story.do-stories.graphstory.com' EPIONE_TEST_DB_PORT='7473' EPIONE_TEST_DB_USER='graphstoryuser' EPIONE_TEST_DB_PASS='graphstorypass' EPIONE_TEST_NODE_HOST='127.0.0.1' EPIONE_TEST_NODE_PORT='8000' gulp test

Your two tests should pass and the code coverage should be over 85%.

### Push to GitHub
Now that your application is locally tested, we can push it to [GitHub](http://github.com).

I won't go over the git work flow, but after you've initialized your git repository via `git init` it's a good idea to create a .gitignore file and exclude some files and directories. My minimal .gitignore for a node project on a Mac using vim looks usually like this:
```
node_modules
npm-debug.log
.DS_Store
.*.swp
.*.swo
```

After adding the files, I commit and push to the master branch of my newly created private GitHub repository.

Now let's make it so that whenever we push to master (or the yet to be created develop branch), a build/deployment process will take place.

## AWS & Shippable
We've decided to host our application on [AWS](http://aws.amazon.com/) primarily for scalability reasons. We will build a dedicated host that takes care of our automated deployments and utilize Elastic Beanstalk to share our application with the world.

[Shippable](https://www.shippable.com/) is a containerized continious integration platform. Because we want to build a docker image for our application from the official node docker image later on, I recommend signing up for the Startup account. This tutorial requires it (at least at this point) and the currently $12/year are well spent.

### Setting up the dedicated host
Now is the time to set up a dedicated host that will communicate with Shippable to make the seemingless deployment possible. For this example, I've chosen to set up a t2.small EC2 instance in North Virginia (USA), also known as [us-east-1](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1). Note: Shippable won't run on a t2.micro, as 2 GB of RAM are required.

Use the Ubuntu 14.04 image on a t2.small instance (or better) with 30 GB of storage (or more). Everything else can stay the same. You'd also need SSH access, so be sure to generate a new key-pair or chose an existing one you can use.
![ubuntu image](https://s3-us-west-2.amazonaws.com/deployment-article/images/aws_dedicated_host.png)
After the instance has launched, I recommend giving it an Elastic IP as a public not changing IP address is required in shippable.

Talking about Shippable, let's head over there.
Once you set-up your startup account for yourself or your organization, we now need to make our AWS instance available for Shippable.
![set up the dedicated host in shippable](https://s3-us-west-2.amazonaws.com/deployment-article/images/aws_dedicated_host_shippable.png)

Now click *Add Node*, connect to your AWS EC2 instance that's going to be the dedicated host, copy the command that's showing up on the modal and paste it into your ssh session. You can then exit the ssh session. Now fill out the connection details and hit save.
![add node](https://s3-us-west-2.amazonaws.com/deployment-article/images/aws_dedicated_host_shippable_add_node.png)

After the node is added, we can initialize it by hitting the *power icon*. This process might take a few minutes. After that, there's one more icon to hit to deploy the builder.
![deploy builder](https://s3-us-west-2.amazonaws.com/deployment-article/images/aws_dedicated_host_shippable_deploy_builder.png)

Your AWS EC2 instance is now ready to build and deploy.

#### Enable GitHub repo for Shippable
Let's find our GitHub repo in Shippable by browsing to app.shippable.com and enable the repo where our application code is hosted.
![enable repo](https://s3-us-west-2.amazonaws.com/deployment-article/images/shippable_select_repo.png)

Now the repository has been activated and Shippable uses the webhooks to start a build process. As you can see, no builds have been run yet. We're about to change that.
![repository activated](https://s3-us-west-2.amazonaws.com/deployment-article/images/shippable_repository_activated.png)

In order to do that, we need to switch between AWS, Shippable and our application source code quite a bit.

#### Get AWS credentials
If you haven't done it already, create a user in the [IAM Management Console](https://console.aws.amazon.com/iam/home?region=us-east-1#users) for the buildserver. You will need the credentials in a bit.

#### Create AWS Elastic Beanstalk
Now we're preparing our Elastic Beanstalk application and environments. I invite you to create a new EBS application in [us-east-1](https://console.aws.amazon.com/elasticbeanstalk/home?region=us-east-1#/newApplication).

I fill out the Application name with *epione*.
On the next page I select *Create Web Server*.
One step further the correct settings for this scenario are *Docker* in "Predifined configuration" and *Load balancing, auto scaling* in "Environment type". I confirm those settings and also accept the default for "Application version" on the next page, which is *sample application*.

On the next page we're setting up our production environment. We are in luck as the name epione is not taken yet.
![setting up the environment](https://s3-us-west-2.amazonaws.com/deployment-article/images/aws_ebs_env_production.png)

On the following pages you can set up this environment to your liking. I stuck with the defaults in this case.

While our *production* environment is launching with a sample application, we should go ahead and set up another environment for development.
![create new env](https://s3-us-west-2.amazonaws.com/deployment-article/images/aws_ebs_new_env.png)

The steps are the same, but you'll need a different environment name and url. Also, you will probably use less powerful instances in development than in production.
![dev env](https://s3-us-west-2.amazonaws.com/deployment-article/images/aws_ebs_dev_env.png)


### Preparing the app for AWS EBS
So let's actually get back to our application source code. In order for our git push to have the desired effect of deploying to AWS, we need to do just a little bit of preparation.

First, let's create and work on a *develop* branch: `git checkout -b develop`

In our project's root, create a .elasticbeanstalk directory: `mkdir .elasticbeanstalk`

In the newly created directory, create a config.yml with these contents:
```yaml
branch-defaults:
  develop:
    environment: epione-dev
  master:
    environment: epione
  default:
    environment: epione-dev
global:
  application_name: epione
  default_ec2_keyname: null
  default_platform: Docker 1.5.0
  default_region: us-east-1
  profile: eb-cli
  sc: git
```

Up next is the shippable.yml:
```
build_image: node
language:
  - node_js
node_js:
  - "0.12"
branches:
  only:
    - master
    - develop
before_install:
  - apt-get install -y python-dev
  - pip install awsebcli
install:
  - npm install
env:
 global:
    - EPIONE_NODE_HOST=0.0.0.0 EPIONE_NODE_PORT=8000 EPIONE_DB_PROTOCOL=https EPIONE_DEVELOPMENT_NODE_HOST=0.0.0.0 EPIONE_DEVELOPMENT_NODE_PORT=8000 EPIONE_DEVELOPMENT_DB_PROTOCOL=https EPIONE_TEST_NODE_HOST=0.0.0.0 EPIONE_TEST_NODE_PORT=8000 EPIONE_TEST_DB_PROTOCOL=https EPIONE_DEVELOPMENT_DB_PORT=7473 EPIONE_TEST_DB_PORT=7473 EPIONE_TEST_DB_PROTOCOL=https
    - secure: YOURSECRETENVVARIABLESENCODEDHERE
before_script:
  - mkdir -p ~/.aws
  - echo '[profile eb-cli]' > ~/.aws/config
  - echo "aws_access_key_id = $AWSAccessKeyId" >> ~/.aws/config
  - echo "aws_secret_access_key = $AWSSecretKey" >> ~/.aws/config
script:
  - npm test
commit_container: manonthemat/epione
after_success:
  - eb init && eb deploy --timeout 20
notifications:
  email: false
```
Some applications might take longer to deploy and while the deployment will still succeed, Shippable will return an error due to EBS not responding in time. To avoid that scenario I recommend increasing the timeout from the default of 10 minutes to 20 minutes with the timeout option in the deploy step as shown above.

Shippable allows you to encrypt the environment variable definitions and keep your configurations private using the secure tag as shown above. To do so, browse to the organization dashboard or individual dashboard page from where you have enabled your project and click on ENCRYPT ENV VARS button.
![encrypt env variables](https://s3-us-west-2.amazonaws.com/deployment-article/images/shippable_encrypt+envs.png)

I've encrypted these environment variables:
- EPIONE_DB_HOST
- EPIONE_DEVELOPMENT_DB_HOST
- EPIONE_TEST_DB_HOST
- EPIONE_DB_USER
- EPIONE_DB_PASS
- EPIONE_DEVELOPMENT_DB_USER
- EPIONE_DEVELOPMENT_DB_PASS
- EPIONE_TEST_DB_USER
- EPIONE_TEST_DB_PASS
- AWSAccessKeyId
- AWSSecretKey

Replace the line `- secure: YOURSECRETENVVARIABLESENCODEDHERE` in the shippable.yml with your encrypted environment variables.

![encoding secrets](https://s3-us-west-2.amazonaws.com/deployment-article/images/shippable_encrypt_dialog.png)

If you have your own IRC Server and/or channel feel free to add shippable notifications to your channel, too. At the end of the file add a line in the format of `irc: "irc.yoursever.com#yourchannel"`. Make sure its indentation matches the disabled email notifications.

## Docker
### Create the docker hub repo
Replace the `commit_container: manonthemat/epione` with your DockerHub repository. If you want it to be a private repository, you have to create it first on Docker Hub and set that repository (not automated build) to private, before you initiate the build process.
![create private docker hub repository](https://s3-us-west-2.amazonaws.com/deployment-article/images/dockerhub_create_private_repo.png)

Also note that when you are using a private repository, you have to give shippable access to your docker hub account. You can do that from the same screen where you started setting up your dedicated host and also started to encrypt your env variables.
![docker hub on](https://s3-us-west-2.amazonaws.com/deployment-article/images/shippable_docker_hub_on.png)

### Dockerfile
Now we're creating the `Dockerfile`:
```
FROM node:0.12

MAINTAINER Matthias Sieber <matthiasksieber@gmail.com>

EXPOSE 8000

COPY . /data
WORKDIR /data
RUN npm install

ENV EPIONE_DEVELOPMENT_DB_PROTOCOL="https" EPIONE_DEVELOPMENT_DB_PORT="7473" EPIONE_DEVELOPMENT_NODE_HOST="0.0.0.0" EPIONE_DEVELOPMENT_NODE_PORT="8000" EPIONE_DEVELOPMENT_DB_HOST="neo-blablabla.do-stories.graphstory.com" EPIONE_DEVELOPMENT_DB_USER"user" EPIONE_DEVELOPMENT_DB_PASS="password" EPIONE_DB_PROTOCOL="https" EPIONE_DB_PORT="7473" EPIONE_NODE_HOST="0.0.0.0" EPIONE_NODE_PORT="8000" EPIONE_DB_HOST="neo-blablabla.do-stories.graphstory.com" EPIONE_DB_USER="user" EPIONE_DB_PASS="password"

CMD ["npm","start"]

```
Since the resulting Docker image will be deployed to AWS EBS in either production or development, we don't include the env variables for test. As before, set the values to your values. It is important to set the node host to 0.0.0.0 for proper access.

Notice also how we're putting all the environment variables in one line to reduce the layers in the resulting Docker image.

We also need a minimalistic `Dockerrun.aws.json` for AWS ElasticBeanstalk to expose port 8000:

```
{
  "AWSEBDockerrunVersion": "1",
  "Ports": [
    {
      "ContainerPort": "8000"
    }
  ]
}
```

### Commit
Now let's commit these changes to GitHub to kick off our build process and deployment to AWS EBS and the creation of a Docker image.

```
git add .
git commit -m 'ready to deploy'
git push origin develop
```

Since shippable registered a webhook with GitHub, shippable will be notified about the recent push of the develop branch and start building the image. You can see the live process by clicking into the build group icon.
![first deploy](https://s3-us-west-2.amazonaws.com/deployment-article/images/shippable_first_build_running.png)

When the build is successful (which also means our application tests passed), the image gets deployed to AWS ElasticBeanstalk.
![deploy to AWS EBS](https://s3-us-west-2.amazonaws.com/deployment-article/images/aws_ebs_new_dev_version.png)

You can check that in your AWS EBS dash board.

### Develop deployed
Browse to your equivalent of http://epione-dev.elasticbeanstalk.com/realusers to see the develop branch of your application live.
![develop deployed](https://s3-us-west-2.amazonaws.com/deployment-article/images/development_deployed.png)

We're happy with this so we're going to merge the develop branch into master and push that to GitHub, so the app will be deployed to the production environment as well.

### Ready to launch
Without changing anything, let's go ahead and merge the develop branch into master.

In our application source code directory:
```
git checkout master
git merge develop
git push origin master
```

Once again, GitHub notifies shippable about the changes on our repository and the build & deployment process starts again. While the application is being deployed, you can check that the old version (the sample application) of the production environment is still live. This is known as a rolling deployment and it's good!

Once AWS is finished with the deployment, go ahead and test your live application.

## But wait, there's more!
One of the reasons we're using docker is that we can have an exact match of our application on a local machine.

One of the many use cases is that I can give my colleagues on the front-end access to my docker repository of my API. They can then develop against an imutable docker container and not have to mess with git for that application.

The last step in our build/deployment process is the upload of our docker image to docker hub. A developer can then go ahead and run a specific version locally and use that to further develop his/her own application against it.

    docker run -p 8000:8000 manonthemat/epione:master.2

Will run the application locally and map the applications port 8000 to another port on the host machine.

### Testing with the Docker image

Try running the tests:

    docker run manonthemat/epione:master.2 npm test

Instead of actually running the tests, this should fail as the node environment get switched to *test* and we haven't included any test environments in the dockerfile. So let's set the expected environment variables for the docker container:

    docker run -e EPIONE_TEST_NODE_HOST='0.0.0.0' -e EPIONE_TEST_NODE_PORT='8000' -e EPIONE_TEST_DB_PROTOCOL='https' -e EPIONE_TEST_DB_PORT='7473' -e EPIONE_TEST_DB_HOST='neo-blablabla.do-stories.graphstory.com' -e EPIONE_TEST_DB_USER='graphstoryuser' -e EPIONE_TEST_DB_PASS='graphstorypassword' manonthemat/epione:master.2 npm test

Now the tests should pass and everything is awesome.

## Things to do when you share your Docker image with the world
There's tne thing to keep in mind when you want to share your created Docker images with the world or you don't want to expose environment variables that are critical to the app running on AWS. The environment variables you define in your Dockerfile are readable by the user downloading that docker image. Since you're running Docker as platform for your Amazon Elastic Beanstalk, you still need these environment variables for our application, but there's an easy fix for this scenario.

First, let's get rid of the env variables in our Dockerfile, so this is all that's left in it.
```
FROM node:0.12

MAINTAINER Matthias Sieber <matthiasksieber@gmail.com>

EXPOSE 8000

COPY . /data
WORKDIR /data
RUN npm install

CMD ["npm","start"]
```

Without the environment variables your docker image is safe, but even if you now deploy your application to AWS Elastic Beanstalk, nginx will probably greet you with a HTTP Status Code 502 "Bad Gateway". A look into the logs and you'll notice that you have not supplied the environment variables needed for our hapi app.

The easy solution is to go to your local git repository and add an `.ebextensions` directory in the root of your project. In this directory, create a file env.config and fill it with your environment variables (the ones you had in your dockerfile) using this format.

```
option_settings:
  - option_name: EPIONE_DEVELOPMENT_DB_PROTOCOL
    value: "https"
  - option_name: EPIONE_DEVELOPMENT_DB_PORT
    value: "7473"
  - option_name: EPIONE_DEVELOPMENT_NODE_HOST
    value: "0.0.0.0"
  - option_name: EPIONE_DEVELOPMENT_NODE_PORT
    value: "8000"
  - option_name: EPIONE_DEVELOPMENT_DB_HOST
    value: "neo-blabla.do-stories.graphstory.com"
  - option_name: EPIONE_DEVELOPMENT_DB_USER
    value: "graphstoryuser"
  - option_name: EPIONE_DEVELOPMENT_DB_PASS
    value: "graphstorypass"
  - option_name: EPIONE_DB_PROTOCOL
    value: "https"
  - option_name: EPIONE_DB_PORT
    value: "7473"
  - option_name: EPIONE_NODE_HOST
    value: "0.0.0.0"
  - option_name: EPIONE_NODE_PORT
    value: "8000"
  - option_name: EPIONE_DB_HOST
    value: "neo-blabla.do-stories.graphstory.com"
  - option_name: EPIONE_DB_USER
    value: "graphstoryuser"
  - option_name: EPIONE_DB_PASS
    value: "graphstorypass"
```

Now add this directory and the file to your git commit. Before you push this commit however, let's take a break and think about what we're doing. Surely, our Amazon Elastic Beanstalk application will have all the information needed for a successful deployment, but so will have the docker image that we'll be pushing to docker hub.

// TODO
We shall remove the AWS specific configuration files from the docker image that's being pushed to docker hub.

// TODO
One more thing. We also don't want the shippable configuration file to be included in the docker image.

Your deployment should be successful and your created docker image should not contain any environment variables you don't want the outside world to know.

## Where to go from here
This deployment strategy saved my development teams a lot of pain, stress and all-around suffering. Because deploying is now *real simple*, we can focus on developing software again.

One thing that I want to do is to include a *after_failure* step.

For example: When the build breaks, the *Release The Drones* protocol will be activated and will find the developer who's responsible for the build breaking. This developer then will get attacked by the swarm and a stationary [Nerf turret](http://amzn.to/1a65ORa).

If this article gets shared over 2500 times via Airpair's social media widget, I shall write a follow up article on that topic.

