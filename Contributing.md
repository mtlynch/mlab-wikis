# Submitting a patch #

We use [Rietveld](http://code.google.com/p/rietveld/) for code reviews and the base should be set to http://REPO.m-lab.googlecode.com/git/ where "REPO" is replaced with the specific repository in the m-lab project.

Patches will not be accepted without a valid review. Anyone in the repository's AUTHORS file can be used a a reviewer. i.e. [AUTHORS](http://libraries.m-lab.googlecode.com/git/trunk/AUTHORS).

If the patch fixes a bug, please include that in the patch description in the form:

BUG=<bug number>

## Setup ##

To install the upload script, see https://code.google.com/p/rietveld/wiki/UploadPyUsage

These are the currently registered m-lab projects:

  1. http://m-lab.googlecode.com/git/trunk/
  1. http://ns.m-lab.googlecode.com/git/
  1. http://ops.m-lab.googlecode.com/git/

Once you have made a change that you would like to submit for review:

```
    $ upload.py --base http://ops.m-lab.googlecode.com/git/ \
        --reviewers author1@email.com,author2@email.com \
	<branch to diff against>
```

The branch to diff against will almost certainly be origin/master, unless you're doing something unusual like maintaining a chain of local branches.

You can visit the URL in the output to verify the patch was submitted, and to edit it (such as to add additional reviewers).

# Github #
We're now using https://github.com/m-lab for analysis and libraries (and other projects) and accept pull requests for those. Other projects may move over to that system soon.