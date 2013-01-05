# gostatic

Gostatic is a static site generator. What differs it from most of other tools is
that it's written in Go and tracks changes, which means it should work
reasonably [fast](#speed).

Features include:

 - Dependency tracking and re-rendering only changed pages
 - Markdown support
 - Flexible [filter system](#processors)
 - Simple [config syntax](#configuration)
 - HTTP server and watcher (instant rendering on changes)

## Approach

Each given file is processed through a pipeline of filters, which modify the
state and then rendered on disk. Single input file corresponds to a single
output file, but filters can generate virtual input files.

Each file can have dependencies, and will be rendered in case it does not exist,
or its source is newer than output, or one of this is the case for one of its
dependencies.

## Speed

On late 2008 MacBook (2.4 GHz, 8 GB RAM, 5400 rpm HDD) it takes `0.5s` to
generate a site of 230 pages. It costs `0.05s` to check there are no
modifications and `0.1s` to re-render a single changed page (along with index
and tag pages, coming to 77 pages in total).

## Configuration

Config syntax is Makefile-inspired with some simplifications, look at the
example:

```Makefile
TEMPLATES = site.tmpl
SOURCE = src
OUTPUT = site

*.md:
    config
    rename *.html
    directorify
    tags tags/*.tag
    markdown
    template page

index.md: blog/*.md
    config
    rename index.html
    inner-template
    markdown
    template

*.tag: blog/*.md
    rename *.html
    directorify
    template tag
    markdown
    template page
```

Here we have constants declaration (first three lines) and then three rules. One
for any markdown file, one specifically for index.md and one for generated tags.

Note: Specific rules override matching rules, but there is no very smart logic
in place and matches comparisons are not strictly defined.

Rules consist of path/match, list of dependencies (also paths and matches, the
ones listed after colon) and commands.

Each command consists of a name of processor and (possibly) some
arguments. Arguments are separated by spaces.

### Constants

There are three configuration constants. `SOURCE` and `OUTPUT` speak for
themselves, and `TEMPLATES` is a list of files which will be parsed as Go
templates. Each file can contain few templates.

## Page header

Page header is in format `name: value`, for example:

```
title: This is a page
tags: test
date: 2013-01-05
```

Predefined properties:

- `title` - page title.
- `tags` - list of tags, separated by `,`.
- `date` - page date, could be used for blog. Accepts formats from bigger to
  smaller (from `"2006-01-02 15:04:05 -07"` to `"2006-01-02"`)

Any other properties can be assigned too, but will be treated as a string.

## Processors

You can always check list of available processors with `gostatic --processors`.

- `config` - reads config from content. Config should be in format "name: value"
  and separated by four dashes on empty line (`----`) from content.

- `ignore` - ignore file.

- `rename <new-name>` - rename a file to `new-name`. New name can contain `*`,
  then it will be replaced with whatever `*` captured in path match. Right now
  rename touches **whole** path, so be careful (you may need to include whole
  path in rename pattern) - *this may change in future*.

- `directorify` - rename a file from `whatever/name.html` to
  `whatever/name/index.html`.

- `markdown` - process content as Markdown.

- `inner-template` - process content as Go template.

- `template <name>` - pass page to a template named `<name>`.

- `tags <name-pattern>` - generate (if not yet) virtual page for all tags of
  current page. This tag page has path formed by replacing `*` in
  `<name-pattern>` with a tag name.

- `external <command> <args...>` - call external command with content of a page
  as stdin and using stdout as a new content of a page.

## Templating

Templating is provided using
[Go templates](http://golang.org/pkg/text/template/). See link for documentation
on syntax.

Each template is executed in context of a page. This means in has certain
properties and methods it can output or call to generate content, i.e. `{{
.Content }}` will output page content in place.

### Page interface

- `.Site` - global [site object](#site-interface).
- `.Rule` - rule object, matched by page.
- `.Pattern` - pattern, which matched this page.
- `.Deps` - list of pages, which are dependencies for this page.

----

- `.Source` - relative path to page source.
- `.FullSource` - full path to page source.
- `.Path` - relative path to page destination.
- `.FullPath` - full path to page destination.
- `.ModTime` - page last modification time.

----

- `.Title` - page title.
- `.Tags` - list of page tags.
- `.Date` - page date, as defined in [page header](#page-header).
- `.Other` - map of all other properties from [page header](#page-header).

----

- `.Content` - page content.
- `.Url` - page url (i.e. `.Path`, but with `index.html` stripped from the end).
- `.UrlTo <other-page>` - relative url from current to some other page.

### Page list interface

- `.Get <n>` - [page](#page-interface) number `<n>`.
- `.First` - first page.
- `.Last` - last page.
- `.Len` - length of page list.

----

- `.Children <prefix>` - list of pages, nested under `<prefix>`.
- `.HasPage <page>` - if `<page>` is contained inside of current page list.
- `.WithTag <tag-name>` - list of pages, tagged with `<tag-name>`.

### Site interface

- `.Pages` - [list of all pages](#page-list-interface).