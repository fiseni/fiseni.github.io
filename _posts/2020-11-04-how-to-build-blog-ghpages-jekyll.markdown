---
# layout: post
# author: Fati Iseni
title: "How to build a blog with GitHub Pages and Jekyll!"
date: 2020-11-04 12:00:00 +0100
description: Tutorial on how to set up a blog with GitHub Pages and Jekyll.
categories: [Blogging, Tutorial]
tags: [Productivity]
image: /assets/img/pozitron-cover.png
pin: false
# math: true
# toc: true
---
I recently decided to change my blogging platform. I simply want to focus on writing new content, and not deal with the peculiarities of different CMS tools. Ideally, I wanted the whole content to be stored in a git repository, and to be able to post articles in a form of markdown files. The design might not be anything fancy, but should be clean and easy to navigate for the users.

First of all, I defined my requirements and listed the features I want the site to have. 
- Use git repository as storage, and publish it publicly on GitHub.
- Write articles in form of .md files, and publish them by simply pushing to the repository.
- I should be able to divide the articles into categories and should be able to assign tags.
- It should support pagination.
- It should support searching through articles (title and description)
- It should support comments (disquss maybe?)
- It should list related articles, recent articles, etc.
- Users should be able to browse through categories or tags
- Users should be able to browse the archive.

There are plenty of options that would fit these requirements. But, ultimately I decided to host my blog as a static site on GitHub Pages. It has built-in support for Jekyll (as a static site generator), it's integrated with GitHub repositories (obviously) and on top of that I can use GitHub actions if there is a need for complex builds/deployments.

In the following sections, I'll walk you through the whole process and provide more details for each step.

## Preparing the custom domain

If you want to use your custom domain name for GH Pages, then you need to update your DNS records, so they correctly point to GH hosting servers. I had my domains on GoDaddy, but since Cloudflare offers free SSL certificates (even on the free tier), I moved the DNS management to Cloudflare. If you don't have SSL for your domain, then I strongly urge to get one. 

Create an account and create your domains on Cloudlfare. Once you do that they will provide their nameservers, which you will have to provide to your registrar. Simply login to your domain registrar's site and change the nameservers into the ones provided by Cloudflare. The change propagation might take some time (up to 48 hours in some cases).

Once the change is validated by Cloudflare, then update your DNS records as shown below

![DNS Settings](/assets/img/posts/02/dns-settings.png)

The A records should point to the GH servers, and you can copy these lines as they are. You should also add a CNAME for `www` subdomain. You can ignore the MX record, it has to do with mail configuration (I'm pointing to my mail server). Lastly, if you add your site to Google Search, you will need a TXT record for site verification.

This is all you need in order to use your custom domain.

## Choosing a template

There are a lot of jekyll templates that you can choose from. Here is a list of some theme galleries

- [jamstackthemes.dev](https://jamstackthemes.dev/ssg/jekyll/)
- [jekyllthemes.org](http://jekyllthemes.org/)
- [jekyllthemes.io](https://jekyllthemes.io/)
- [jekyll-themes.com](https://jekyll-themes.com/)

I want a simple and clean template. No longer I want any additional side content (e.g twitter feeds).
You can find the template I chose in the following [link](https://github.com/cotes2020/jekyll-theme-chirpy). The author, Cotes Chung, did an awesome job. Also, I rarely see such an organized and comprehensive GH repository. Documentation, examples, contribution guidelines, templates for issues. Kudos to the author!

I did additional changes to the template, and you can find my repo [here](https://github.com/fiseni/fiseni.github.io/commits/master). If you like it, please feel free to use it as you wish. Still, I urge you to give some credit to the original author (starring the project at least).


## Installation

I will try to simplify the installation steps here. If you need more information, check [here](https://github.com/fiseni/fiseni.github.io/commits/master). If you need to build the site locally, then you will need a linux environment. Now that we have WSL2 on Windows that's quite easy to accomplish, and it's not worth trying to make it work natively on Windows.

1. Fork from [Cotes Chung's repository](https://github.com/cotes2020/jekyll-theme-chirpy) or [Fati Iseni's Blog](https://github.com/fiseni/fiseni.github.io/commits/master) (based on your preference). You can copy the content instead of forking if that's more convenient for you.
2. Change the name of the repository into `{username}.github.io`, where `username` is your github username. You get one site per GitHub account and organization, and unlimited project sites.
3. Clone the repository locally

    ```
    git clone https://github.com/USERNAME/USERNAME.github.io.git
    ```

    (Now you will need to prepare the local environment)
4. Install `Ruby`, `RubyGems`, and `Jekyl`. Additional information [here](https://jekyllrb.com/docs/installation/)

    ```
    sudo apt-get install ruby-full build-essential zlib1g-dev
    ```
    ```
    echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
    echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
    echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    ```
    ```
    gem install jekyll bundler
    ```
5. Go to the root directory of the project and complete the installation of the Jekyll plugins 

    ```
    bundle install
    ```
6. Install a few more tools that we will need.

    ```
    sudo apt-get install coreutils
    ```
    ```
    sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys CC86BB64
    sudo add-apt-repository ppa:rmescandon/yq
    sudo apt update
    sudo apt install yq -y
    ```
7. Now that the required tools are installed, go to the root directory of the project and run the init job. This will you do only once, it simply just cleans the repository and makes it ready for you.

    ```
    bash tools/init.sh
    ```
8. Update the configuration and customize it for your needs

    - Update the `_config.yml` file with your information
    - Update the `CNAME` file, enter your domain name
    - Update the images under `/assets/img/`

9. Run the site locally

    ```
    bash tools/run.sh
    ```

## Deploying on GitHub Pages

Usually, pushing the content is all you need. GitHub Pages will build it automatically and your site will be up and running.
In our case, we're using build scripts in order to generate some additional content (list of tags, categories, archives, etc). GitHub Pages does not allow script execution (for safety reasons), so we will use GitHub Actions to build the site manually using our own build script.

Everything is already configured (there is yaml file under .github folder), you just have to follow these steps
1. Push the content to master branch. Once the build is complete, new branch will be generated `gh-pages`
2. Open the repository setting under GitHub, and change the publish source to `gh-pages`

![GitHub Pages Settings](/assets/img/posts/02/ghpages-settings.png)

Now, your site will be up and running.

## Publishing content

In order to publish new articles, all you have to do is create a new .md file under `posts` folder. Each file should contain the following information at the top

```
---
title: "Welcome to my new blog!"
date: 2020-10-30 12:00:00 +0100
description: My new blog based on GitHub Pages and Jekyll.
categories: [Blogging, Tutorial]
tags: [Productivity]
image: /assets/img/pozitron-cover.png
pin: true
# math: true
# toc: true
---
```

If you want to learn more about all the available variables that you can use, then click [here](https://github.com/cotes2020/jekyll-theme-chirpy/wiki/Writing-a-new-post).