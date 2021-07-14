# Overview

These Bitbucket pipelines are based on the original CircleCI pipeline defined in the .circleci folder in this project. Bitbucket and CircleCI have some pretty differing feature sets, so I opted to strip the pipeline down to the components that could run on the default infrastructure for both providers. In general, my approach for a task like this is to get the basic CI functionality up and running so teams can start (or keep) shipping, then build out the more advanced features later, if they're missed.

There are 3 pipelines:
- The pull request pipeline, which has steps to:
    - runs unit tests (Python 3.7 and 3.8)
    - runs crowdsource tests
    - runs teacher tests
    - ensure that the website HTML bundle builds.
- The staging branch pipeline, which builds the website HTML bundle (but does not publish/deploy it)
- The master branch pipeline, which has steps to:
    - publishes the parlai package to PyPi
    - builds the website container image, tagged with BITBUCKET_COMMIT
    - pushes the website image to ecr
    - updates the website ECS service to use the image tagged with BITBUCKET_COMMIT

Additionally, each pipeline runs an install step which installs all python dependencies into a virtualenv,
and caches it for use with further builds.

# Changes from the CircleCI pipeline
- **Removed test parallelism:** The test suite was using some provider-specific tooling to facilitate running tests in parallel across multiple build agents, and since Bitbucket pipelines don't have a comparable toolset I opted to just run the tests serially.

- **Removed executor specifications:** Bitbucket pipelines don't expose as much control over selecting build agents as CircleCI does, so we can't choose instance sizes, OS families, or GPU support. We could use Bitbucket runners to achieve most of this goal, but setting up ASGs for all of the various instance types would represent a pretty significant amount of scope creep. I just kept the tasks that could run on Linux CPUs, and the other builds can be set up in a subsequent task.

- **Got rid of the S3 deployment:** In the new pipeline, we deploy an updated task definition to an imaginary ECS cluster using the ECS deployment pipe.

- **Added PyPi publishing:** I didn't see any packagine work in the original repo, but this is definitely a pypi package, so I used a pipe to publish the package on pushes to `master`.

# Things considered
- **Speed:** Bitbucket pipelines have a set amount of build minutes per month, and I tried to make good use of caching and small deployments to reduce how long the build runs. However, with the loss of larger instances or GPU agents, moving CI providers would likely represent an increase in build times in the short term.
- **Keeping the pipeline legible:** I made use of YAML anchors to reduce the amount of code that would have to be rewritten for each pipeline. Using anchors also makes it easier to separate "function" definition from instantiation, so it's easier to read the pipeline at a glance and see _what_ the pipeline does without having to get into the nitty-gritty of _how_. I tried to use descriptive anchor names to achieve that legibility goal as well.
-** Caching:** ensuring pip dependencies are properly cached so that they can be reused across builds, which will reduce build time.
- **Image tagging conventions:** I like to tag docker images with their corresponding commit hash, so that it's very easy to identify the commit corresponding to a deploy. The commit hash gets threaded through to the `image` argument in the ECS task definition to ensure depoyment immutability. If you rebuild the build for commit `a1b2c3d4`, you're sure that's image tag `a1b2c3d4` will get deployed as well.
- **Tracking deployments:** I made sure to mark the deployment stage so we can get metrics on deployment count, mean time to recovery, change failure rate, etc. later.
- **Limiting the amount of new code written:** normally, I'd wrap multiple commands in a script. I find that paradigm makes it easy to test your CI pipelines locally, and you can do more compmlicated operations with a single command. However, all of this pipeline code has already been written, and I don't think there's any benefits to taking that approach now.
- **Making use of pipes:** this one also relates to the above bullet point, re: limiting the amount of code written. I was sure to use Bitbucket pipes to ensure that I didn't have to fully orchestrate ECS deploys and image pushes. Why use a script where a plugin will do?
- **Keeping docker images simple:** I opted for a minimal install for the website image. I thought about adding an `nginx.conf`:
```
server {
    listen 80;                      # Bind to port 80
    server_name parl.ai localhost;  # Sets hostnames associated with this server

    include mime.types;             # Map filename to content-type header
    location / {
        root /srv/html;             # Serve files from /srv/html/
        index index.html index.html # Set default index page
    }
}
```
However, it was unnecessary. In prod we'd have CloudFront to handle caching and content type headers, and the ALB to terminate SSL and route traffic to the right server based on hostname. We don't really need any of the things that'd go into a custom nginx.conf, so I just left it out.