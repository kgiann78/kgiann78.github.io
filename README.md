# My personal's page README 

This README contains information about two things: 

* how to run and configure this page and run it with Jekyll
* the Jekyll Theme that is currently used

## How to configure it

In the `_config.yml` file you will find a lot of information regarding 

* switches that can be turned on or off (i.e. to show or not certain links and pages)
* the permalinks (how the url of each page is shown)
* personal information to show

After you change something in the `_config.yml` file, it is good to remove completely the `_sites` directory (no worries it will be reconstructed) and run again the site (locally) with:

    bundle exec jekyll serve --incremental

The original posts that came with the theme are stored in the `original_content` directory in order to advise it for markdown format.



