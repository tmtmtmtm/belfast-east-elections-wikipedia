
The code and queries etc here are unlikely to be updated as my process
evolves. Later repos will likely have progressively different approaches
and more elaborate tooling, as my habit is to try to improve at least
one part of the process each time around.

---------

Step 1: Configure config.json
=============================

All the relevant metadata now lives in config.json: ideally nothing will
need tweaked after this. We need to be careful here to get the history
of Wikidata IDs for the constituency correct.

Step 1: Scrape the results
==========================

```sh
jq -r '.wikipedia | "\(.url) \(.section)"' config.json | xargs bundle exec ruby scraper.rb | tee wikipedia.csv
```

Some tweaks needed to the single-result version of this, but not as many
as I'd expected. Also now also picking up the 'ranking' in the results:
this seems a little fragile, but seems correct in this layout at least.

Step 2: Generate possible missing IDs
=====================================

```sh
xsv search -v -s id 'Q' wikipedia.csv | xsv select name | tail +2 |
  sed -e 's/^/"/' -e 's/$/"@en/' | paste -s - |
  xargs -0 wd sparql find-candidates.js |
  jq -r '.[] | [.name, .item.value, .election.label, .constituency.label, .party.label] | @csv' |
  tee candidates.csv
```

I can't find any of the others in Wikidata manually.

Step 3: Combine Those
=====================

```sh
xsv join -n --left 2 wikipedia.csv 1 candidates.csv | xsv select '10,1-8' | sed $'1i\\\nfoundid' | tee combo.csv
```

Step 4: Generate QuickStatements commands
=========================================

```sh
bundle exec ruby generate-qs.rb config.json | tee commands.qs
```

Then sent to QuickStatements as https://editgroups.toolforge.org/b/QSv2T/1597123895437/
