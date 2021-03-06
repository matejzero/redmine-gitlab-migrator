Redmine to Gitlab migrator
==========================

[![Build Status](https://travis-ci.org/matejzero/redmine-gitlab-migrator.svg?branch=master)](https://travis-ci.org/matejzero/redmine-gitlab-migrator)

Migrate code projects from Redmine to Gitlab, keeping issues/milestones/metadata

Does
----

- Per-project migrations
- Migration of issues, keeping as much metadata as possible:
  - redmine trackers become tags
  - redmine categories become tags
  - issues comments are kept and assigned to the right users
  - issues final status (open/closed) are kept along with open/close date (not
    detailed status history)
  - issues assignments are kept
  - issues numbers (ex: `#123`)
  - issues/notes authors
  - issues/notes original dates, but as comments
  - issue attachements
  - issue related changesets
  - issues custom fields (if specified)
  - relations including children and parent (although gitlab model for relations is simpler)
  - keep creation/edit dates as metadata
  - remember who closed the issue
  - convert Redmine's textile format issues to GitLab's markdown
  - possible to map to different users in GitLab
- Migration of Versions/Roadmaps keeping:
  - issues composing the version
  - statuses & due dates

Does not
--------

- Migrate users, groups, and permissions (redmine ACL model is complex and
  cannot be transposed 1-1 to gitlab ACL)
- Migrate repositories (piece of cake to do by hand, + redmine allows multiple
  repositories per project where gitlab does not)
- Migrate wikis (because we don't use them at @oasiswork, feel free to contribute)
- Migrate the whole redmine installation at once, because namespacing is different in
  redmine and gitlab
- Archive the redmine project for you
- Keep "watchers" on tickets (gitlab API v3 does not expose it)
- Keep dates/times as metadata
- Keep track of issue relations orientation (no such notion on gitlab)
- Migrate tags ([redmine_tags](https://www.redmine.org/plugins/redmine_tags)
  plugin), as they are not exposed in gitlab API

Requires
--------

- Python >= 3.4
- gitlab >= 7.0
- redmine >= 1.3
- Admin token on redmine
- Admin token on gitlab
- No preexisting issues on gitlab project
- Already synced users (those required in the project you are migrating)

(Original version was developed/tested around redmine 2.5.2, gitlab 8.2.0, python 3.4)
(Updated version was developed/tested around redmine 2.4.3, gitlab 9.0.4, python 3.6)


Let's go
--------

You can or can not use
[virtualenvs](http://docs.python-guide.org/en/latest/dev/virtualenvs/), that's
up to you.

Install it:

    pip install redmine-gitlab-migrator


(or if you cloned the git: `python setup.py install`)

You can then give it a check without touching anything:

    migrate-rg issues --redmine-key xxxx --gitlab-key xxxx \
      <redmine project url> <gitlab project url> --check

The `--check` here prevents any writing , it's available on all
commands.

    migrate-rg --help

Migration process
-----------------

This process is for each project, **order matters**.

### Create the gitlab project

It doesn't neet to be named the same, you just have to record it's URL (eg:
*https://git.example.com/mygroup/myproject*).

### Create users

Manual operation, project members in gitlab need to have the same username as
members in redmine. Every member that interacted with the redmine project
should be added to the gitlab project.
If a corresponding user can't be found in gitlab, the issue/comment will be
assigned to the gitlab admin user.

### Migrate Roadmap

If you do use roadmaps, redmine *versions* will be converted to gitlab
*milestones*. If you don't, just skip this step.

    migrate-rg roadmap --redmine-key xxxx --gitlab-key xxxx \
      https://redmine.example.com/projects/myproject \
      http://git.example.com/mygroup/myproject --check

*(remove `--check` to perform it for real, same applies for other commands)*

### Migrate issues

    migrate-rg issues --redmine-key xxxx --gitlab-key xxxx \
      https://redmine.example.com/projects/myproject \
      http://git.example.com/mygroup/myproject --check

Note that your issue titles will be annotated with the original redmine issue
ID, like *-RM-1186-MR-logging*. This annotation will be used (and removed) by
the next step.

At least redmine 2.1.2 has no closed_on field, so you have to specify the names of the states which define closed issues.
defaults to closed,rejected

    --closed-states closed,rejected,wontfix

If you want to migrate redmine custom fields (as description), you can specify

    --custom-fields Customer,ZendeskIssueId

If you're using SSL with self signed cerificates and get an *requests.exceptions.SSLError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:600)* error, you can disable certificate validation with

    --no-verify

### Migrate Issues ID (iid)

You can retain the issues ID from redmine, **this cannot be done via REST
API**, thus it requires **direct access to the gitlab machine**.

So you have to log in the gitlab machine (eg. via SSH), and then issue the
commad with sufficient rights, from there:

    migrate-rg iid --gitlab-key xxxx \
      http://git.example.com/mygroup/myproject --check

### Import git repository

A bare matter of `git remote set-url && git push`, see git documentation.

Note that gitlab does not support multiple repositories per project, you'll have
to reorganize your projects if you were using that feature of Redmine.

### Archive redmine project

If you want to.

You're good to go :).

### Optional: Redirect redmine to gitlab (for apache)

Since redmine has a common *https://redmine.company.tld/issues/{issueid}* url for issues, you can't create a generic redirect in apache.

This command creates redirect rules that you can place in your `.htaccess` file.

    migrate-rg redirect --redmine-key xxxx --gitlab-key xxxx \
      https://redmine.example.com/projects/myproject \
      http://git.example.com/mygroup/myproject > htaccess.example

The content of htaccess.example will be

    # uncomment next line to enable RewriteEngine
    # RewriteEngine On
    # Redirects from https://redmine.example.com/projects/myproject to https://git.example.com/mygroup/myproject
    RedirectMatch 301 ^/issues/1$ https://git.example.com/mygroup/myproject/issues/1
    RedirectMatch 301 ^/issues/2$ https://git.example.com/mygroup/myproject/issues/2
    ...
    RedirectMatch 301 ^/issues/999$ https://git.example.com/mygroup/myproject/999

Unit testing
------------

Use the standard way:

    python setup.py test

Or use whatever test runner you fancy.
