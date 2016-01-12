# Reddit Style Voting #

A quick guide to implementing those famous [reddit](http://programming.reddit.com) up and down voting arrows which cast or clear your votes, using django-voting.

## URL Configuration ##

In this example, we'll be using two generic views:
  1. the `object_list` view which comes with Django will be used to paginate `Link` objects.
  1. the `vote_on_object` view which comes with django-voting will be used to vote on individual `Link` objects.

Here's a sample URLConf to set this up:

```
from django.conf.urls.defaults import *
from django.views.generic.list_detail import object_list

from devdocs.apps.kb.models import Link
from voting.views import vote_on_object

urlpatterns = patterns('',
    # Generic view to list Link objects
    (r'^links/?$', object_list, dict(queryset=Link.objects.all(),
        template_object_name='link', template_name='kb/link_list.html',
        paginate_by=15, allow_empty=True)),
        
    # Generic view to vote on Link objects
    (r'^links/(?P<object_id>\d+)/(?P<direction>up|down|clear)vote/?$',
        vote_on_object, dict(model=Link, template_object_name='link',
            template_name='kb/link_confirm_vote.html',
            allow_xmlhttprequest=True)),
)
```

## Template ##

As we're using the `object_list` generic view to paginate items, we'll have to use the `voting_tags` template library supplied with django-voting to load the voting and scoring information we'll need.

In this example template, corresponding to `kb/link_list.html` defined in the above URLConf, we're dealing with a list of `Link` objects, contained in the context variable `link_list`.

First, we want to load existing votes the current user has made on each `Link` in the list and the current scores and number of votes cast for each `Link` as we'll need this information to set up the initial state of our voting widgets and forms. _If we were dealing with a list of items in our own custom view function, we could instead use the appropriate helper methods in `Vote.objects` to load voting and scoring information to place on the context._

```
{% load voting_tags %}
{% votes_by_user user on link_list as vote_dict %}
{% scores_for_objects link_list as score_dict %}
```

We'll display our list of links in a basic table:

```
<table>
<col width="1"></col>
<col></col>
<thead>
  <tr>
    <th>Vote</th>
    <th>Link</th>
  </tr>
</thead>
<tbody>
  {% for link in link_list %}<tr class="{% cycle odd,even %}">
    <td class="vote">
```

So far, so easy - but now we want to set up the arrow widgets for voting on the the current `Link`.

We'll start by getting the vote made for the current `Link` by the current user and the `Link`'s current score from the dictionaries we loaded earlier:

```
      {% dict_entry_for_item link from vote_dict as vote %}
      {% dict_entry_for_item link from score_dict as score %}
```

`Vote` objects have `is_upvote` and `is_downvote` methods which we can use to check which direction the user voted in for the current `Link`. We guard against the user not having voted yet by first checking for the existence of the `vote` context variable.

In this implementation, each voting arrow will get its own form, with the `action` attribute determining the kind of vote which will be cast when the form is submitted. For the arrow voting widget in each form, an `input type="image"` is used. We'll `POST` the forms for simplicity's sake, so there won't be a vote confirmation screen.

If the user has already voted in either direction, the arrow image displayed should indicate this and submitting the form should clear the user's vote:

```
      <form class="linkvote" id="linkup{{ link.id }}" action="{{ link.id }}/{% if vote and vote.is_upvote %}clear{% else %}up{% endif %}vote/" method="POST">
        <input type="image" id="linkuparrow{{ link.id }}" src="{{ media_url }}img/aup{% if vote and vote.is_upvote %}mod{% else %}grey{% endif %}.png">
      </form>

      <form class="linkvote" id="linkdown{{ link.id }}" action="{{ link.id }}/{% if vote and vote.is_downvote %}clear{% else %}down{% endif %}vote/" method="POST">
        <input type="image" id="linkdownarrow{{ link.id }}" src="{{ media_url }}img/adown{% if vote and vote.is_downvote %}mod{% else %}grey{% endif %}.png">
      </form>
```

We'll finish  off by displaying the Link itself and its score details:

```
    </td>
    <td class="item">
      <a href="{{ link.url }}">{{ link.title|escape }}</a></h2>
      <p class="details">
        <span class="score" id="linkscore{{ link.id }}"
              title="after {{ score.num_votes|default:0 }} vote{{ score.num_votes|default:0|pluralize }}">
         {{ score.score|default:0 }} point{{ score.score|default:0|pluralize }}
        </span>
        posted {{ link.created|timesince }} ago by
        <span class="user"><a href="../users/{{ link.user.id }}/">{{ link.user.get_full_name|escape }}</a></span>
        <span class="details"><a href="{{ link.get_absolute_url }}">details</a></span>
      </p>
    </td>
  </tr>{% endfor %}
</tbody>
</table>
```

## Progressive Enhancement ##

You may have noticed the `class` attribute on the voting forms and the `id` attributes on the voting forms, widgets and score details - these will allow us to [progressively enhance](http://domscripting.com/blog/display/41) the voting process using XMLHttpRequest.

Note that we've prefixed these `class` and `id` attributes with `"link"` - this is a bit of forward thinking so we can easily progressively enhance pages which allow you to vote on multiple types of content at once.

When you're using the `voting.views.vote_on_object` view, you can enable fall-_forward_ to processing and responding to the request in a manner suitable for an XMLHttpRequest by passing in `allow_xmlhttprequest=True` when setting up the view.

As we've configured the voting view this way in our sample urlconf, we can use JavaScript when the page loads to hijack the voting forms, cancelling the default form submission, sending the voting request via XMLHttpRequest and processing the JSON response we're given, which involves updating the voting forms, widgets and scores to reflect the vote which was made when it was successfully processed.

For this example, we'll use the [Prototype library](http://www.prototypejs.org/), version 1.5.1 RC1, purely because it already sets the `HTTP_X_REQUESTED_WITH` header the `voting.views.vote_on_object` view looks for to determine if a request was made using XMLHttpRequest (that and I want to see what's changed since version 1.3.1).

All that said, here's an example script for progressively enhancing voting - using the `VoteHijacker` object defined below, we can have all voting done via XMLHttpRequest, with voting arrows, form actions and scores updated appropriately after votes have been made.

This is where the `"link"` prefix we added to all of our important `class` and `id` attributes comes into play - the `VoteHijacker` constructor accepts a single argument for this prefix, then uses the prefix when searching for forms to hijack their `onsubmit` event, and again when it needs to update the state of the form actions, voting arrows and scores. For our example, the `VoteHijacker` would be used as follows:

```
<script type="text/javascript">
Event.observe(window, "load", function()
{
    new VoteHijacker("link");
});
</script>
```

And the script source:

```
var VoteHijacker = Class.create();
VoteHijacker.prototype =
{
    initialize: function(prefix)
    {
        this.prefix = prefix || "";
        this.registerEventHandlers();
    },

    registerEventHandlers: function()
    {
        $$("form." + this.prefix + "vote").each(function(form)
        {
            Event.observe(form, "submit", this.doVote.bindAsEventListener(this), false);
        }.bind(this));
    },

    doVote: function(e)
    {
        Event.stop(e);
        var form = Event.element(e);
        var id = /(\d+)$/.exec(form.id)[1];
        var action = /(up|down|clear)vote/.exec(form.action)[1];
        new Ajax.Request(form.action, {
            onComplete: VoteHijacker.processVoteResponse(this.prefix, id, action)
        });
    }
};

VoteHijacker.processVoteResponse = function(prefix, id, action)
{
    return function(transport)
    {
        var response = transport.responseText.evalJSON();
        if (response.success === true)
        {
            var upArrowType = "grey";
            var upFormAction = "up";
            var downArrowType = "grey";
            var downFormAction = "down";

            if (action == "up")
            {
                var upArrowType = "mod";
                var upFormAction = "clear";
            }
            else if (action == "down")
            {
                var downArrowType = "mod";
                var downFormAction = "clear";
            }

            VoteHijacker.updateArrow("up", prefix, id, upArrowType);
            VoteHijacker.updateArrow("down", prefix, id, downArrowType);
            VoteHijacker.updateFormAction("up", prefix, id, upFormAction);
            VoteHijacker.updateFormAction("down", prefix, id, downFormAction);
            VoteHijacker.updateScore(prefix, id, response.score);
        }
        else
        {
            alert("Error voting: " + response.error_message);
        }
    };
};

VoteHijacker.updateArrow = function(arrowType, prefix, id, state)
{
    var img = $(prefix + arrowType + "arrow" + id);
    var re = new RegExp("a" + arrowType + "(?:mod|grey)\\.png");
    img.src = img.src.replace(re, "a" + arrowType + state + ".png");
};

VoteHijacker.updateFormAction = function(formType, prefix, id, action)
{
    var form = $(prefix + formType + id);
    form.action = form.action.replace(/(?:up|down|clear)vote/, action + "vote");
};

VoteHijacker.updateScore = function(prefix, id, score)
{
    var scoreElement = $(prefix + "score" + id);
    scoreElement.innerHTML = score.score + " point" + VoteHijacker.pluralize(score.score);
    scoreElement.title = "after " + score.num_votes + " vote" + VoteHijacker.pluralize(score.num_votes);
};

VoteHijacker.pluralize = function(value)
{
    if (value != 1)
    {
        return "s";
    }
    return "";
};
```