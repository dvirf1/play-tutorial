# Play Framework Tutorial

In this tutorial we will create a new Play Framework application from scratch and play around with its build.
We will see how to work with config, use compile-time dependency injection, how to dockerize the app, work with json,
access the DB, construct the logic so we will have easier time to update Play Framework and so on.

The tutorial is available at:  
[https://dvirf1.github.io/play-tutorial/](https://dvirf1.github.io/play-tutorial/)

## Development:

### tl;dr

Complete the first time setup and then:

```bash
bundle exec jekyll serve
# browse to http://localhost:4000
```

### Installation Instructions

You should install ruby and bundler.  
On macOS:

1. Install [rbenv](https://github.com/rbenv/rbenv#homebrew-on-macos): `brew install rbenv`
2. Run `rbenv init`.
Add the output of this command to your shell config.
e.g For fish users: add `status --is-interactive; and source (rbenv init -|psub)` to your `~/.config/fish/config.fish`
e.g For zsh/bash users: add `eval "$(rbenv init -)"` to your `~/.bash_profile`
3. Close your Terminal window and open a new one so your changes take effect
4. Verify that rbenv is properly set up using this rbenv-doctor script:
`curl -fsSL https://github.com/rbenv/rbenv-installer/raw/master/bin/rbenv-doctor | bash`
5. In your terminal, run `rbenv install 2.6.5`.
6. In your project directory, make sure that `ruby -v` returns the correct version (2.6.5)
7. Install bundler with `gem install bundler`
8. Install dependencies with `bundle install`

Then use

```bash
bundle exec jekyll serve
```

and browse to [http://localhost:4000/play-tutorial/](http://localhost:4000/play-tutorial/).


### Create a new version:

From the project directory:

```bash
bundle exec jekyll build
JEKYLL_ENV=production bundle exec jekyll build -d docs # builds production version to `docs` directory
```

### Directory Structure

The main files and related brief introductions are listed below.

```sh
play-tutorial/
├── _data
├── _includes      
├── _layouts
├── _posts          # posts stay here
├── _scripts
├── assets      
├── tabs
│   └── about.md    # the ABOUT page
├── .gitignore
├── 404.html
├── Gemfile
├── LICENSE
├── README.md
├── _config.yml     # configuration file
├── tools           # script tools
├── docs
├── feed.xml
├── index.html
├── robots.txt
└── sitemap.xml
```

## Credits

- [Play Framework](https://www.playframework.com/)
- [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy/)

## License

This work is published under [MIT](https://github.com/cotes2020/jekyll-theme-chirpy/blob/master/LICENSE) License.
