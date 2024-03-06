<h1 align="center">
  Hello! 👋
</h1>

This repository is the home for my Project Blog website, intended to document projects I've been working on. Some of my initial posts have been tutorial-esque, but I'll be moving away from that format as it takes a LONG TIME to write guides and tutorials. I like to teach about what I'm studying but that's not my focus right now--maybe in the future!

## Table of Contents

1. [Acknowledgment](#acknowledgment)
2. [About the Project](#about-the-project)
3. [Secondary Goals](#secondary-goals)
4. [Notes/To Do](#notesto-do)

## Acknowledgment

This website is built with [Jekyll](https://jekyllrb.com) using the [Chirpy theme](https://chirpy.cotes.page), forked from [here](https://github.com/cotes2020/chirpy-starter). Be sure to check out those projects for more information.

## About the Project

For this project, I wanted to experiment with DNS and the process of building and hosting a simple static website. This was my first time using a Static Site Generator; fortunately, Jekyll made it really simple and easy to get up and running. I've built/designed/hosted websites in the past using HTML/CSS and hosting platforms like Github Pages, but this time around I wanted to try with a VPS. The primary goal was to get a workflow set up that looked a bit like this:

- Create a GH repo to store the code and dev tools (dockerized Jekyll instance)
- Develop the site locally using VSCode and Git
- Push the code to Github
- Build the site [*see secondary goals]
- Sync the build files to the remote VPS

## Secondary Goals

In addition to the main goal of figuring out how to host a website on a remote destination, I also wanted to automate the build and deploy steps. This would make updating the site a lot easier and more hands-off. For this I used Github Actions. This was an intro to Actions--in the future I'd like to learn more about creating my own pipelines rather than modifying others' examples. For this project, just getting things working was enough.

## Notes/To Do

For now, I don't plan any major changes to this project. It's my intent to continue posting about projects as I complete them, but this may be infrequent. Here are some items I'd like to tackle:

- Migrate from VPS to Cloud (VPS expires in June)
- Review and revise CI/CD pipeline
