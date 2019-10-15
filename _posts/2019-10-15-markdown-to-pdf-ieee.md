---
layout: post
title:  "Markdown to PDF With IEEE Style"
subtitle: ""
date: 2019-10-15
categories: []
---

* TOC
{:toc}

This post is a how-to guide on how a research paper can be written in
[Markdown] and then converted to a PDF following [IEEE style]. In case you are
not familiar, IEEE style is usually the format how technical research papers
are written.

## Why?

If you ever need to write an [IEEE style] research/technical paper your options
are pretty much as follows:

* write in Microsoft Word

  This is definitely an option however if you are a developer and are used
  to writing things in plaintext, with version control, this might not be an
  appealing option.

* write in LaTeX

  LaTeX is really amazing at generating PDFs however its syntax is very
  flexible but complicated (at least to a beginner) which makes it a really
  good option if you are already familiar with LaTeX but not very appealing if
  you are new to LaTeX or simply don't like the syntax.

* write in anything but LaTeX and then convert to LaTeX

  Since LaTeX is so good at generating PDFs, it would be amazing to be able to
  write the paper in a much simpler markup language such as Markdown and then
  convert it to LaTeX to generate the PDF. Enter [Pandoc]! Pandoc is a document
  swiss-army converter between various markups, including Markdown and LaTeX.
  This post will guide in more detail how exactly to do that.

## Prerequisites

This post assumes you are using a Mac however many of the steps should
be compatible with Linux or if not might need only minor changes.

First lets install all dependencies with [Homebrew]:

```bash
$ brew install \
    fish \
    pandoc \
    pandoc-citeproc \
    pandoc-crossref
$ brew cask install basictex
```

[BasicTeX] is a lightweight distribution of LaTeX however as such it does not
support some things such as SVG images. If you need to use SVG images in
the paper, instead of `basictext` you should install the following:

```bash
$ brew install rsvg-convert
$ brew cask install mactext-no-gui
```

## Sample repository

All examples in this post can be found in [`md2pdf-ieee-sample`] GitHub
repository. All `cd` commands in the examples indicate in which folder the of
the sample repository commands should be executed in. To clone the repository:

```bash
$ git clone https://github.com/miki725/md2pdf-ieee-sample.git
$ cd md2pdf-ieee-sample
```

## Basics

As all dependencies are ready, lets try a basic example. We will convert a
simple Markdown document to a PDF:

```markdown
# Basic Title

## Very Important Subtitle

**Super** amazing article *here* about
[cats](https://www.pexels.com/search/cat/).

![[amazing cat](http://bit.ly/2pilsGS)](cat.jpg)
```

If all goes well final PDF should look something like:

![basics pdf](/assets/images/2019-10-15-markdown-to-pdf-ieee/basics.png)

Here are the commands to create the PDF:

```bash
$ cd basics
$ pandoc --from markdown --to latex --output basics.pdf basics.md
```

Lets break the `pandoc` command down:

* `--from` specifies the input file markup format name
* `--to` specifies the output file markup format name
* `--output` specifies the output filename. Note that even though `to`
  file format is `latex`, since filename has extension `pdf`, pandoc will
  automatically create PDF from converted LaTeX document. If you omit the
  `output` parameter, pandoc will show the converted LaTeX document:

    ```bash
    $ pandoc --from=markdown --to=latex basics.md
    ```

## YAML LaTeX Metadata

Full research paper requires more metadata than Markdown allows to annotate
therefore some LaTeX cannot be avoided. Pandoc allows to specify such
metadata with [YAML]. It can be provided either as a separate file with
`--metadata-file` parameter or be directly embedded in Markdown file as YAML
header. Here is a simple metadata header example:

```
---
title: Basic Title
subtitle: Very Important Subtitle
author:
  - name: Miroslav Shubernetskiy
    affiliation: GameChanger
    location: New York
    email: example@gc.com
numbersections: yes
lang: en
abstract: |
    This paper is a basic example with LaTeX metadata.
header-includes: |
  \usepackage{booktabs}

...

---
```

```bash
$ cd paper-metadata
$ pandoc --from markdown --to latex --output basics.pdf basics.md
```

Generated PDF should look similar to:

![paper with metadata pdf](/assets/images/2019-10-15-markdown-to-pdf-ieee/paper-metadata.png)

Its not perfect but we will be improving it in the following sections.

## IEEE Template

Above example adds some metadata which is only used by IEEE template hence the
`true` in the paper header. Applying IEEE template fixes those issues and
makes PDF look like this:

![paper with ieee template pdf](/assets/images/2019-10-15-markdown-to-pdf-ieee/ieee-template.png)

Much better :D.

IEEE template itself consists of 2 files:

* [`IEEEtran.cls`] - LaTeX class file with all IEEE specifications
* [`template.latex`] - paper template which uses the `IEEEtran.cls` to format
  the LaTeX document

The command which uses all the files to generate PDF:

```bash
$ cd ieee-template
$ pandoc \
    --from=markdown \
    --to=latex \
    --template=template.latex \
    --output=ieee-template.pdf \
    ieee-template.md
```

## Referencing Figures

Referencing figures is relatively simple. Adding a figure is as simple as
including an image in Markdown except at the end a figure label is added which
allows to reference it in other places:

```markdown
See @fig:cat.

![[amazing cat](http://bit.ly/2pilsGS)](cat.jpg){#fig:cat}
```

![figures](/assets/images/2019-10-15-markdown-to-pdf-ieee/figures.png)

When multiple figures are present pandoc will automatically cross reference the
correct figure number however that requires providing additional `--filter`
flags to pandoc:

```bash
$ cd figures
$ pandoc \
    --from=markdown \
    --to=latex \
    --template=template.latex \
    --filter=pandoc-crossref \
    --filter=pandoc-citeproc \
    --output=figures.pdf \
    figures.md
```

Note that the order of `pandoc-crossref` and `pandoc-citeproc` is important.

## Bibliography

Any good research paper should include references. Pandoc allows to provide
a bibliography via a [BibTeX] file. PDF with References section as well as a
citation to a reference should look like this:

![bibliography](/assets/images/2019-10-15-markdown-to-pdf-ieee/bibliography.png)

A BibTeX reference looks like the following and multiple references can be
combined in `bibliography.bib`:

```
@article{shell2019use,
  title={How to use the IEEEtran LATEX class},
  author={Shell, Michael},
  journal={IEEE Design \& Test},
  year={2019},
  publisher={IEEE}
}
```

Some services like [Google Scholar] allow to download references directly in
BibTeX format:

![Google Scholar BibTeX](/assets/images/2019-10-15-markdown-to-pdf-ieee/google-scholar.png)

All references can be included in Markdown with `nocite: '@*'` YAML document.
Also note that each BibTeX includes a label. For the example above, that label
is `shell2019use` and it can be used to cite references in the paper simply
with a `@<label>`:

```markdown
Shell [@shell2019use] writes more about `IEEEtran` file.

# References

---
nocite: '@*'
---
```

By default references are not formatted with IEEE style therefore additional
configuration file is necessary:

* [`bibliography.csl`] - LaTeX class file for formatting references

The command to generate PDF is now:

```bash
$ cd bibliography
$ pandoc \
    --from=markdown \
    --to=latex \
    --template=template.latex \
    --filter=pandoc-crossref \
    --filter=pandoc-citeproc \
    --bibliography=bibliography.bib \
    --csl=bibliography.csl \
    --output=bibliography.pdf \
    bibliography.md
```

## Bibliography in Markdown

Instead of relying on `bibliography.bib`, there is a way to define all
references in the YAML header however it has a different format compared to
BibTeX which means BibTeX references cannot be simply copy-pasted. There is a
hack though :D. Copy BibTeX references to YAML as is, and then extract them
into `bibliography.bib` with a script. Given Markdown YAML header:

```Markdown
title: Basic Title
subtitle: Very Important Subtitle
author:
  - name: Miroslav Shubernetskiy
    affiliation: GameChanger
    location: New York
    email: example@gc.com
numbersections: yes
lang: en
bibliography: |
  @article{shell2019use,
    title={How to use the IEEEtran LATEX class},
    author={Shell, Michael},
    journal={IEEE Design \& Test},
    year={2019},
    publisher={IEEE}
  }
abstract: |
    This paper is a basic example with LaTeX metadata
    with IEEE template.
header-includes: |
  \usepackage{booktabs}

...

---
```

All bibliography references can be extracted with a bit of Python:

```bash
$ cd bibliography-md
$ cat bibliography-md.md | python -c "
import itertools
import sys
print('\n'.join(
    list(
        map(
            lambda i: i.strip(),
            itertools.takewhile(
                lambda i: i.strip() != i or i.startswith('bibliography'),
                itertools.dropwhile(
                    lambda i: not i.startswith('bibliography'),
                    sys.stdin.read().splitlines()
                ),
            )
        )
    )[1:]
))
" > bibliography.bib
$ pandoc \
    --from=markdown \
    --to=latex \
    --template=template.latex \
    --filter=pandoc-crossref \
    --filter=pandoc-citeproc \
    --bibliography=bibliography.bib \
    --csl=bibliography.csl \
    --output=bibliography-md.pdf \
    bibliography-md.md
```

Granted this is a bit hacky but it allows paper to be self-contained.
In the next section we will make the paper to be even more self-contained.

## [`md2pdf-ieee.fish`]

Now we can generate research paper PDF with IEEE style however the pandoc
command has lots of parameters and we need to download multiple files
first into the current directory before generating PDF. We can abstract
all that away though with [`md2pdf-ieee.fish`] [Fish Shell] function.

The function does the following:

* downloads all necessary files to `~/.pandoc`
* extracts the bibliography from the Markdown file
* generates PDF with pandoc
* opens the generated file

Installing and using the function is pretty easy. Simply download
[`md2pdf-ieee.fish`] to `~/.config/fish/functions/md2pdf-ieee.fish` or download
with `curl`:

```bash
$ mkdir -p ~/.config/fish/functions
$ curl -L http://bit.ly/2MHJgMn \
    > ~/.config/fish/functions/md2pdf-ieee.fish
```

If you use `fish` as a default shell simply call the function:

```bash
$ cd md2pdf-ieee
# source should automatically happen
# if you start another fish shell
$ source ~/.config/fish/functions/*
$ md2pdf-ieee paper.md
```

Or if you use any other shell you can call fish function with:

```bash
$ cd md2pdf-ieee
$ fish -c 'md2pdf-ieee paper.md'
```

This allows to write a complete paper in a single Markdown file and convert to
PDF with a single command. Easy-peasy!

## Conclusion

Congratulations. You should be able to create, if I may, beautiful, documents.
Now all you need is a good research idea!

[IEEE style]: https://en.wikipedia.org/wiki/IEEE_style
[Pandoc]: https://pandoc.org/
[Markdown]: https://en.wikipedia.org/wiki/Markdown
[YAML]: https://yaml.org/
[LaTeX]: https://www.latex-project.org/
[BibTeX]: http://www.bibtex.org/
[BasicTeX]: https://www.tug.org/mactex/morepackages.html
[Homebrew]: https://brew.sh/
[Google Scholar]: https://scholar.google.com
[Fish Shell]: https://fishshell.com/
[`IEEEtran.cls`]: https://raw.githubusercontent.com/miki725/md2pdf-ieee/master/IEEEtran.cls
[`template.latex`]: https://raw.githubusercontent.com/miki725/md2pdf-ieee/master/template.latex
[`bibliography.csl`]: https//raw.githubusercontent.com/miki725/md2pdf-ieee/master/bibliography.csl
[`md2pdf-ieee.fish`]: https://raw.githubusercontent.com/miki725/.dotfiles/master/.config/fish/functions/md2pdf-ieee.fish
[`md2pdf-ieee-sample`]: https://github.com/miki725/md2pdf-ieee-sample
