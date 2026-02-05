+++
author = "Grant Houser"
title = "Creating My New Personal Website"
date = "2026-02-05"
description = "Architecting a static site using Hugo, Anatole, and GitHub Actions for project tracking."
tags = ["github", "hugo", "powershell", "ci-cd"]
categories = ["web development"]
+++

I’ve spent a lot of years in this business, and if there is one thing I have learned, it is that even the best projects get forgotten if you don’t write them down. I built this site to serve as a home for my project notes, a bit of a "how-to" library, and a way to share what I’m working on with anyone who is interested.

## 1. Motivation
It is not always easy to keep track of every hurdle and victory over a long career. I wanted a spot where I could look back at what I have done and what skills I used to get there. My hope is that by documenting my own process, I can build a resource for others who are tackling the same issues.

There is an old productivity tip: when you start a to-do list, the first item should be "Create a to-do list." That way, you get to cross something off immediately. This post is my version of that - documenting the very process of getting this site live.

<br>
## 2. Strategy
I’m a big fan of keeping things lean. I didn't need a flashy, complicated site with a massive database to manage. I wanted something fast, secure, and easy to maintain.

  ### Architecture
  * **Framework:** Hugo (A static site generator that handles the heavy lifting).
  * **Theme:** Anatole (I love the minimalist, clean look).
  * **Hosting:** GitHub Pages (Reliable and integrates perfectly with my workflow).
  * **Automation:** GitHub Actions (I set this up so that the moment I finish writing, the site updates itself automatically).

  ### Core Tools
  * **Hugo Extended:** Required for SCSS processing in modern themes.
  * **VS Code:** Primary IDE for Markdown and TOML configuration.
  * **PowerShell:** Execution environment for local builds and Git operations.

<br>
## 3. Implementation

  ### Initializing the Local Environment
  Using `winget` for package management, I installed the Hugo binary and initialized the directory structure.

  ```powershell
  # Install Hugo Extended
  winget install Hugo.Hugo.Extended

  # Create the site and initialize version control
  hugo new site my-project
  cd my-project
  git init
  ```

  ### Theme & Configuration
  I selected the <b>Anatole</b> theme for its minimalist aesthetic and clear navigation. To maintain a clean upstream path, I integrated the theme as a Git submodule.

  ```powershell
  # Add Anatole as a tracked submodule
  git submodule add [https://github.com/lxndrblz/anatole.git](https://github.com/lxndrblz/anatole.git) themes/anatole

  # Deploy the example configuration as a baseline
  Copy-Item -Path "themes/anatole/exampleSite/config.toml" -Destination "hugo.toml" -Force
  ```

  ### Continuos Deployment (CI/CD)
  I configured a GitHub Action (.github/workflows/deploy.yml) using the standard Hugo deployment workflow. This allows me to push Markdown files and let GitHub handle the site compilation automatically.

  ### Final Push
  The local source was linked to my remote repository and pushed to the main branch:
  ```powershell
  git add .
  git commit -m "Initial commit: Infrastructure setup & theme integration"
  git branch -M main
  git remote add origin [https://github.com/karatetakeout/karatetakeout.github.io.git](https://github.com/karatetakeout/karatetakeout.github.io.git)
  git push -u origin main
  ```

<br>
## 4. Results & Roadmap
The site is now live at granthouser.info. The deployment pipeline is verified, and the Anatole interface is successfully rendering content.

  ### Next Steps
  Now that the foundation is laid, the real work begins. I’ll be going back through my recent projects to outline them here, and I’ll be posting updates on any new initiatives I start.