---
title: "Deploying Leiningen projects to GitHub Packages in 2020"
categories: [blog, software engineering]
tags: [clojure, leiningen, github, packages]
---

Recently, I've attended the [Lviv.py #9](https://www.meetup.com/uapycon/events/270181750/) meetup and was 
inspired by a presentation of GitHub Actions. Actions were around for years, but I decided
it is a great chance to finally use them. I decided to run a little test on 
[a small Clojure project](https://github.com/shapiy/charon) of mine.

A particular aspect of most CI pipelines is artifact deployment. Currently,
GitHub actions support five repository types: npm, Docker, Maven, NuGet and RubyGems[^1].
Leiningen provides great support of Maven interop including publishing artifacts in 
Maven repositories[^2].

As is often the case, when doing new things in Clojure, Internet may not give you a
complete recipe for your problem. Solving it often takes reading various articles, 
inspecting source code, and making educated guesses. This time, I did find an excellent 
walk-through[^3] but I thought things might become simpler since then. Let's 
setup deployment of Leiningen artifacts in three simple steps. 

## I – Set up a new workflow in your GitHub repo

GitHub actions already hold a template for Clojure based projects. We'll start with 
the simplest thing possible.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    # Checkout the source code
    - uses: actions/checkout@v2
    # Build JAR
    - name: JAR
      run: lein jar
```

## II – Add target repository to project.clj

Add a binary repository to your `project.clj` with:

- url: `https://maven.pkg.github.com/{username}/{repo}`
- username: your GitHub username
- password: we'll read it from `GITHUB_TOKEN` environment variable

```clojure
:repositories [["github" {:url "https://maven.pkg.github.com/shapiy/charon"
                          :username "shapiy"
                          :password :env/github_token}]]
```

GitHub automatically populates a set of environment variables such as 
`GITHUB_RUN_ID`, `GITHUB_ACTION`, etc[^4]. `GITHUB_TOKEN` is a special secret 
that is automatically created by GitHub Actions to simplify authentication[^5].
Unlike other variables, it is not a part of environment variables, so we'll
have take additional step to make it work.

## III – Add deployment step

Let's give our workflow the finishing touch by throwing `GITHUB_TOKEN` secret among
the environment variables.

```yaml
- name: Deploy
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: lein deploy github
```

Voilá!

![GitHub package](/blog/assets/charon.png)

# Conclusions

As you can see, GitHub Actions and GitHub Repositories are fully suitable for 
building and deploying Clojure packages in 2020. 

# Footnotes

[^1]:
    GitHub packages landing page: [https://github.com/features/packages](https://github.com/features/packages)

[^2]:
    See Leiningen documentation on [library deployment](https://github.com/technomancy/leiningen/blob/master/doc/DEPLOY.md).

[^3]:
    Jiayu Yi's blog on [the same matter](https://blog.jiayu.co/2019/09/deploying-leiningen-projects-to-github-package-registry/).

[^4]:
    [https://help.github.com/en/actions/configuring-and-managing-workflows/using-environment-variables](https://help.github.com/en/actions/configuring-and-managing-workflows/using-environment-variables)

[^5]:
    [https://help.github.com/en/actions/configuring-and-managing-workflows/authenticating-with-the-github_token](https://help.github.com/en/actions/configuring-and-managing-workflows/authenticating-with-the-github_token)   