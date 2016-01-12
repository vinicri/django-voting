A generic voting application for Django projects.

_As of [revision 56](https://code.google.com/p/django-voting/source/detail?r=56), django-voting requires a recent SVN checkout of Django as it is tracking [Backwards Incompatible Changes](http://code.djangoproject.com/wiki/BackwardsIncompatibleChanges/) made to Django since version 0.96._

This application is based on the voting application in Jacob Kaplan Moss' [CheeseRater](http://www.cheeserater.com/) project, which kindly makes available its source code.

The original code had:
  * registering votes against any `Model` instance.
  * retrieval of the score for an object.
  * retrieval of top and bottom-rated objects for a particular `Model`.

This application adds:
  * the ability to clear votes.
  * a template tag library.
  * a generic view for wiring up voting for a `Model`: GET requests result in a confirmation page, POST requests submit votes.
  * a generic view for voting using XMLHttpRequest - as a bonus, if the non-XMLHttpRequest generic view detects that a request was made via XMLHttpRequest, it will automatically use this view to process the request, which makes it trivial to progressively enhance your project with XMLHttpRequest-based voting.

Use this command to check out the latest source:

```
svn checkout http://django-voting.googlecode.com/svn/trunk/voting/
```

The latest version of this application's documentation, in reStructuredText format, is always available in its [overview.txt](http://django-voting.googlecode.com/svn/trunk/docs/overview.txt) file.