# What is this?

This is a XiangShan document built using the mkdocs tool. This tool can convert markdown documents into web pages and display them in a beautiful way. Projects built with mkdocs can be hosted on the Internet by readthedocs for access. Projects such as Chipyard and BOOM all make their documents public in this way. (The slight difference is that they use the rst format writing specification, while we may prefer markdown)

# Process of adding new content

This document is bound to the [XiangShan-doc](https://github.com/OpenXiangShan/XiangShan-doc) project on Github. Whenever the main branch receives a git push update, the web page will automatically reconstruct and display the new content.

Therefore, if you want to add new content, follow the steps below:：

- Clone the [XiangShan-doc](https://github.com/OpenXiangShan/XiangShan-doc) project.
- Install the mkdocs environment locally (see [MkDocs Installation](https://www.mkdocs.org/user-guide/installation/))
- Install the Material for MkDocs theme locally (see [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/getting-started/)) (TLDR: pip install -r docs/requirements.txt)
- Modify the document content locally
- Preview the document locally through the mkdocs serve command and check whether it meets your expectations in the browser
- `git push`` to the remote end

# Project structure

The mkdocs project structure is very simple. There is a mkdocs.yml file in the root directory to record configuration information. Among them, we need to focus on the nav sub-item, which determines the directory structure of the entire document displayed on the web page, which can be modified as appropriate; the docs folder in the root directory saves the markdown format documents, which is also the actual content we need to add or modify.

# Material for MkDocs

We use Material for MkDocs for the following reasons:：

1. Better multi-level heading support
1. Friendly to export to pdf好
1. There are many things you can do, see: https://squidfunk.github.io/mkdocs-material/reference//

!!! note
    For example, this kind of thing.。

# Export to pdf

Using the Material for MkDocs theme with the mkdocs-with-pdf plugin, we can automatically export the entire document to a pdf file. mkdocs.yml already contains a preset pdf export configuration. For how to configure the pdf export environment and how to use it, please refer to https://pypi.org/project/mkdocs-with-pdf/

!!! info
    Note that the native theme has some problems when exporting code blocks to pdf. It is recommended to use the material theme when exporting to pdf.
