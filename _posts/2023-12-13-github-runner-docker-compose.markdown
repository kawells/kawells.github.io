---
layout: post
title:  "Automating Docker Compose Configuration Changes Using Github Workflow"
date:   2023-12-13 19:11:00 -0800
categories: github docker automation devops
---
I've been a big fan of using Docker (and Docker Compose) for years now — regularly running 20+ containers — but a minor annoyance that I've run into is managing a fleet of `docker-compose.yml` files.
I currently run about 5 separate Docker Compose stacks. I prefer this approach because I can update a single stack without having to worry about affecting any of the containers within other stacks.
It also allows me to easily create a separate default Docker network for that stack.
For example, my monitoring stack might include [Uptime Kuma](https://uptime.kuma.pet/), [Prometheus](https://prometheus.io/), and [Grafana](https://grafana.com/).
Let's say I want to add [cAdvisor](https://github.com/google/cadvisor) to the stack.
Normally, I'd have to SSH to my server, edit the `monitor\docker-compose.yml`, then run some Docker Compose commands to get everything running with the latest configuration.
That's too much work.
Here's a better way to automate Docker Compose configuration changes using a Github Workflow and a Github Runner.

# Store Your Configurations in a Repository

This website is running on Github Pages, so of course I'm going to shill for them here.
If you aren't doing so already, save any of your files that you'd like to push to your server(s) in their own repository on Github. For me, this includes all `YML` files that Docker and any of my containers will use.
Besides being able to push these configurations to the remote server, having the configs in a version-controlled repo will help us recover when we inevitably add a tab or incorrect indentation in our YML file and our stack accidentally disappears.

What I personally do is first clone my Github repo locally. I then use [VS Code](https://code.visualstudio.com/) to make any changes, then commit those changes. Now those changes are live in the repo, but how do we get them to the server?

# Install a Github Actions Runner on the Server
If you'd like to keep it all in Docker, you can run a [Github Actions Runner in Docker](https://github.com/myoung34/docker-github-actions-runner).
That's what I did on my Synology NAS, and it was straightforward.
I used [this guide](https://oleksandrkirichenko.com/blog/github-runner-on-synology/) and had no issues.
One important note is that your Github Actions Runner must have access to file paths that your Docker containers need access to.
If any of your Docker containers require file system access, make sure that you mount those volumes using the same paths that the containers' volumes are mounted using.
After you do that, go back to Github and make sure that your runner shows as idle, and note the tags. My tags, for example, were `self-hosted`,`Linux`,`X64`, and `default`.

# Setup a Github Workflow
This was both easier and a little more tedious than I expected, just because I have only ever used Azure DevOps pipelines prior to this. Here's a sample `workflow.yml` file to get you going, and I will explain what it means. You'll have to save yours in your repository in the root directory within a `.github/workflows/` folder. If it doesn't already exist, create it.

```
name: push-docker-compose
run-name: ${{ github.actor }} pushed config change to Docker
on:
  push:
    branches:
      - main
    paths:
      # Only run this when there is a commit involving a modified docker-compose.yml file
      - '*docker-compose.yml'
jobs:
  update-docker-configs:
    # Add an array of tags here that identify your runner. Especially useful if you have more than one runner/server that use the same repository
    runs-on: [self-hosted, Linux, X64]
    steps:
      - name: Check out the repository to the runner
        uses: actions/checkout@v4
      - name: Get changed yml files
        # This action will check for any changed docker-compose.yml files
        uses: tj-actions/changed-files@v40
        id: changed-yml-files
        with:
          files: |
             *docker-compose.yml
      - name: If changed docker compose yml files, update stack
        if: steps.changed-yml-files.outputs.any_changed == 'true'
        # For each changed file, pull updated images and update the containers, then prune the docker images
        run: |
          for file in ${{ steps.changed-yml-files.outputs.all_changed_files }}; do
            echo "$file was changed"
            docker-compose -f $file pull
            docker-compose -f $file up -d --remove-orphans
          done
          docker image prune -a -f
```

There you have it! Now, every time you commit a `docker-compose.yml` to your repository, the Github Actions Runner will copy the repository to your server and update your Docker Compose configuration automatically!