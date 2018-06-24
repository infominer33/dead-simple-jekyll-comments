Dead Simple Jekyll Comments
===========================
DIY comment system for Jekyll

- Works on any repo (Github, Bitbucket, ...)
- No need to trust a third party
- No JavaScript
- No PHP
- No Ruby
- Only Liquid and one shell script
- Supports threaded conversations

How it works
------------
- A static HTML form is included in each article
- Form submissions are intercepted by [`webhook`]() and processed by a bash script
- This script creates a separate branch and writes the form values to a yaml file in `_data/comments`
- Comment is reviewed and merged into master
- Comment files are rendered using Liquid includes

Detailed Instructions
---------------------
Outline:

0. Install prerequisits
1. Create a few sample comment files
2. Render these comments + a HTML comment form
3. Create the `add-comment.sh` script
4. Create a `hooks.yaml` configuration file
5. Launch webhook and test the whole thing

### Install prerequisits
On Ubuntu:

    # apt install yq webhook passwd

### Sample comment files
The file name should be `<comment id>.yaml` where the comment id is on the format `<unix timestamp>-<random identifier>`.

**Example:** `1516695540-quRch1Gi.yaml`

The format is carefully chosen: The timestamp makes it trivial to show the comments in the correct order, and the random suffix avoids collisions when two comments are submitted the same second **and** makes the comment id unguessable, should you decide to have some approve/reject moderation link.

The file should live in `_data/comments`.

The comment file format looks like follows:

    reply_to: /some-post.html
    author: John Doe
    email: john.doe@gmail.com
    text: Nice article!

The above would be a top level comment on `/some-post.html`. Had it been a reply to another comment, it would have had something like

    reply_to: 1516695540-quRch1Gi

Here are some sample comments: [`sample-comments.zip`](sample-comments.zip). Unzip and place these files in `_data/comments`.

### Render comments + a comment HTML form
Download the following files:

- [`test-page.md`](test-page.md), save in the root of your site

  Apart from some sample CSS, this page contains
  
    [...]
    
    Comments
    --------
    {% include comments.html replies_to=page.url %}
    {% include comment-form.html replies_to=page.url %}

- [`comments.html`](comments.html), save it in `/_includes`

  This file displays a comment tree. The `replies_to` parameter should be `page.url`. Internally it includes itself recursively for comment replies. The `replies_to` parameter then refers to the parent comment.

- [`comment-form.html`](comment-form.html), save it in `/_includes`

  A simple comment form.

Build the site and open the resulting `test-page.html`. You should see something like this:

![Sample comments screenshot](screenshot.png)

### Create a script for adding comments
Download the [`add-comment.sh`](add-comment.sh) bash script. Place it in a new directory called `comments-server` outside your repository.

The usage is

    ./add-comment.sh <reply_to> <author> <email> <text>

The script does the following:

1. Makes sure the local copy of the repo is up to date
2. Clones a working copy
2. Creates a branch off of master
3. Writes the arguments to a yaml file according to the comment file format
4. Commits and pushes
5. Removes the working copy of the repo

Note that the script assumes there's a local copy of the repository in `./repo`, so from within `comments-server/` do

    git clone --bare <YOUR REPO URL.git> repo

The reason for creating a temporary copy of repository is to allow multiple concurrent comment submissions. An alternative is to invoke `add-comment.sh` through the brilliant program [`tsp`](http://vicerveza.homeunix.net/~viric/soft/ts/) which serializes all invocations.

It depends on [`yq`](https://github.com/mikefarah/yq) for writing yaml files to simplify handling of multiline strings.

### Create a webhook configuration file
Download [`hooks.yaml`](hooks.yaml) and place it in `comments-server`.

This configuration says that requests to `/hooks/add-comment` should trigger `add-comment.sh`.

### Launch webhook and test
Launch `webhook`:

    $ webhook -verbose -hooks hooks.yaml

Go to `<site url>/test-page.html` and try to submit a comment. Make sure a new branch is created with the comment. Merge the branch, rebuild your site and refresh the page.

Additional Features
-------------------

### Comment Count
The file [`count-comments.md`](count-comments.md) counts the number of comments in a comment thread and stores the result in a variable called `comment_count`. Put it in `_includes` and use it as follows:

    {%- include count-comments.md replies_to=page.url -%}
    <h2>Comments ({{- comment_count -}})</h2>

or, a more elaborate version:

    {% include count-comments.html replies_to=page.url %}
    {% if comment_count %}
    <h2>Comments ({{ comment_count }})</h2>
    {% include comments.html replies_to=page.url %}
    <h3>Add comment</h3>
    {% else %}
    <h2>Comments</h2>
    Be the first to comment!
    {% endif %}
    {% include comment-form.html reply_to=page.url %}

### Email features
`ssmtp` is trivial to configure and allows you to...
- Notify you about new comments
- This email can contain links to other webhooks that
  - accepts the comment (merges into master)
  - rejects the comment (deletes the branch)
- Verify the users email by:
  - First putting the comment file in an "unverified" directory
  - Send an email to the provided email address with a link to a webhook
  - The webhook triggers a script that creates a branch, moves comment file to the data directory and commits / pushes.
- If you have a CI system that builds all branches, you can include a link to the comment branch to easily review how the comment looks rendered.