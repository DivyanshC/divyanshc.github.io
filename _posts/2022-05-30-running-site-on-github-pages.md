---
title: Setting up docs site on GitHub Pages using Jekyll
date: 2022-05-30 15:52:00 +0530
categories: [homelab, github-pages]
tags: [homelab, servers] # tag names should always be lowercase
---

# Using Jekyll: The Static Site Generator to build our website

## Install Dependencies

```bash
sudo apt update
sudo apt install ruby-full build-essential zlib1g-dev git
```

## Install Jekyll `bundler`

```bash
gem install jekyll bundler
```

## Creating a site based on Chirpy Starter Theme

Visit https://github.com/cotes2020/jekyll-theme-chirpy#quick-start for step by step instructions or click on [Chirpy Starter](https://github.com/cotes2020/chirpy-starter/generate) on their page to automate the flow.

**Repo name should be \<YOUR-USER-NAME>.github.io**

After creating a site (forking repo) based on the template, clone your repository to your local machine.

```bash
git clone git@<YOUR-USER-NAME>/<YOUR-REPO-NAME>.git
```

then install your dependencies:

```bash
cd repo-name
bundle
```

## Make changes in \_config.yml file

```yaml
timezone: Asia/Kolkata
title: HomeLab Docs
tagline: A Doc site for my homelab
description: A Doc site for my homelab.
url: "https://divyanshc.github.io/"
theme_mode: dark
avatar: <URL-of-Profile-Photo>
```

Profile photo url can be taken from facebook, twitter.

## Adding a new post to your site:

create a new file in the `_posts` directory.

File name should be in the format of `YYYY-MM-DD-TITLE-OF-POST.md`

Image from asset:

```markdown
... which is shown in the screenshot below:
![A screenshot](/assets/screenshot.jpg)
```

Linking to a file:

```markdown
... you can [download the PDF](/assets/diagram.pdf) here.
```

Check the [Jekyll Pages](https://jekyllrb.com/docs/posts/) for more information about Local Linking of Files (Assets).

All content should be in **markdown** format. Check [Markdown Cheatsheet](https://www.markdownguide.org/cheat-sheet/) for more information.

## Running the site locally and pushing to GitHub

Serving your site locally is as simple as running the following command:

```bash
bundle exec jekyll s
```

This will start the server on port 4000. http://127.0.0.1:4000/ will be your site.

### After making changes to your site, commit and push then up to git on **main** branch

```bash
git add .
git commit -m "feat(post): <POST-TITLE>"
git push origin main
```

This site already has github actions configured, this uses CI/CD to automatically get your site from the main branch by updating gh-pages branch.

Just push to the main branch and it will automatically build and deploy from the gh-pages branch.

Check the **Actions tab** on your repo for more information.

### Change these settings in repo settings

- Your repo should be set to `Public`
- Go to pages tab in repo settings and set `Source` to `gh-pages`, then click on `Save` (push on main branch only)
- Check Enforce HTTPS

Your site will be available at **https://\<YOUR-USER-NAME>.github.io/**

## Adding Custom Domain

You can add an apex domain or sub-domain of your site. Github will redirect the traffic to your site from **https://\<YOUR-USER-NAME>.github.io/** also.

Got to Settings > Pages > Custom Domain > Add your custom domain name > Save

Create `A` records, point your apex domain to the IP addresses for GitHub Pages.

Add new record:

- Type: **A**
- Name: **@** for no subdomain or subdomain name
- IP Address: **<IP-ADDRESS-OF-GITHUB-PAGES>**

```Text
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

Add records with all four using name as **@** and **www** subdomain.

### Check if DNS configuration is correct:

```bash
dig DIVYANSHCHAUHAN.COM +noall +answer -t A
```

If _cloudflare_ is your nameserver then you will see some cloudflare IP addresses not of github pages that we added.

This will commit a cname file in 'gh-pages' branch which will get removed after new commit takes place so update it in config.yml file.

```yaml
url: "https://divyanshchauhan.com"
```

This will add a cname file in 'gh-pages' branch **automatically** on commit.

Check **Enforce HTTPS** field in repo Settings > Pages. (It can take upto 24 hours before this option is available.)

## Verify you Domain in Github

This will only allow your github account to host site using github pages on this verified domain or its subdomains.

Got to github main Settings under Profile > Code, planning, and automation > Pages > Add domain.

Follow the process to add a domain.

Check if TXT record added in DNS.

```bash
dig _github-pages-challenge-USERNAME.example.com +nostats +nocomments +nocmd TXT
```

USERNAME: **\<YOUR-GITHUB-USER-NAME>**

example: **\<YOUR-DOMAIN-NAME>**

Check [Github Docs](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain) for more information on how to setup custom domain.

## Self Hoting the site

Building your site in production mode to get output to \_site folder to deploy it ourselves:

```bash
JEKYLL_ENV=production bundle exec jekyll b
```

### Using Docker

Create a `Dockerfile` with the following:

```dockerfile
FROM nginx:stable-alpine
COPY _site /usr/share/nginx/html
```

Build the image:

```bash
docker build .
```

Be sure to ⭐ the [jekyll repo](https://github.com/jekyll/jekyll) and the [Chrirpy theme repo](https://github.com/cotes2020/jekyll-theme-chirpy)
