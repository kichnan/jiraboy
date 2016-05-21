---
layout: post
title:  "Creating daily status updates email from JIRA"
date:   2016-02-09 21:11:30 +0530
categories: Reporting
author: krishnan
---
## Context
My client has always preferred reading an email at the start of his day that summarizes all the work his dev team was up to the night before. We call it **the status mail**. It usually contains the list of tasks we have worked on, containing its full description along with our comments/status on that day. Such reports are favorites among clients who are non-technical or who cannot afford to spend time in JIRA, as they get all updates in one go. But, of course, this is beneficial only for a team size of 5-20 members. Any more than that, the mail itself becomes too long to read.

## Background
We adopted JIRA only recently, slightly more than a year ago. During the pre-JIRA dark ages, my predecessors had to perform a laborious task of _compiling_ status mails from tasks, bugs and feature requests sent by the clients over emails. It used to easily take up 30-60 mins on a rainy day, copy pasting updates provided by the team, then mix-matching the formats to make it look good, etc. Moreover, it was too much manual labor to love this part of the job.

Then came JIRA, and life became much more efficient.


## Creating Status Mail from JIRA
So, let us get down to the core part of this post. The steps to create a status mail are pretty straightforward. But before we proceed further, go through the checklist below that you may need (to know).

#### The Secret Ingredients List
1.  [JQL][] (JIRA Query Language) to build your query
1.  [JIRA API][] to fetch the query results programmatically
1.  An application[<sup>+</sup>](#language-note) to fetch your results using JIRA API
1.  JavaScript and [JQuery][][<sup>+</sup>](#language-note) to process the result (final JSON)
1.  [JQuery templates][jqt][<sup>+</sup>](#language-note) to build your email content (HTML) from your processed _final JSON_.

<a name="language-note">\[<sup>+</sup>\]</a>: Choose language or tool of your choice.


### JQL
First and foremost, you need to fetch only relevant issues that have been updated that day to be processed and put into the status mail. JQL (JIRA Query Language) is the best way to get that done. A typical example of the query I use is as follows.
{% highlight sql %}
project in ("My Project") AND comment ~ "\\[statuscomment\\]"
    AND updated > startOfDay() AND updated < endOfDay()
ORDER BY key DESC
{% endhighlight %}

#### NOTE
*   In our implementation, a comment that needs to go into the status mail must be prefixed with `[statuscomment]`, so that only the relevant comments are grabbed when processing results (later below).
*   Of course, you can use any other keyword instead -- just change it in the [processing function](#processor-fn). If you don't want any "keyword processing" at all, such that all comments from today should show up, that's fine by me. Just, again, make sure to change the [processing function](#processor-fn) accordingly.
*   You may save the JQL query as a **JIRA Filter**, so that you can reuse or reference it later.
*   The JQL above is not perfect, because it may fetch issues that have not been actually updated with a `statuscomment` _on that day_. A typical example is, if you added a status comment for an issue yesterday (and not today), but you did _update_ it today (like marked it to done or edited some fields), the task still qualifies for the JQL above. This is because [JQL][] does not support querying individual comments' `updated` dates. However, this should not be an issue for us, as we will be filtering out such comments, as you will see later.


### JIRA API
To fetch issues from JIRA in to your application, you need to use [`/search`][search] API. You can use either `GET` or `POST` method to do so. The JIRA documentation explains all necessary steps clearly, so I'm not covering that. However, here are the key parameters that you should add to fetch all necessary details in one go.
{% highlight json %}
{
    "expand": "names,schema,renderedFields",
    "jql": "our JQL query mentioned above",
    "fields": "summary,description,comment"
}
{% endhighlight %}

Here is a sample JQL [result-set][].
{% gist kichnan/43959db34e0bb26717b7d66d94555c55 03-jira-api-result-sample.json %}

There are a few key things to note in this result set:

*   It brought only the fields mentioned in `fields` parameter above.
*   The `startAt` and `maxResults` properties are used for pagination. By default, JIRA API brings only 50 items per query. So, if you have more than 50 issues to process daily, you should tweak your API call accordingly.
*   There is a `renderedFields` property which contains the actual _rendered_ HTML of the description fields, and not the JIRA wiki syntax as in descriptions in the original `fields`. The JIRA wiki syntax is useless for us.


### Process Result
<a name="processor-fn"></a>The following JavaScript code processes the above JSON and builds me a cleaner JSON that I can use to build my final status mail. Refering to the code below, calling the `processStatus(yourJSONFromAPI)` function will give you the full and final status mail HTML built using [JQuery Templates][jqt]. You may use that HTML output however you want to send your status email to your client/boss.
_NOTE:_ You may need to modify below code for your use.
{% gist kichnan/43959db34e0bb26717b7d66d94555c55 01-jira-task-status-processor.js %}

And here are the jQuery templates used to generate the HTML.
{% gist kichnan/43959db34e0bb26717b7d66d94555c55 02-jira-status-html-jqtemplate.html %}


## Ending Note
The output `finalHTML` generated out of this exercise can be either sent as email (through whichever means you like), or maintained as web pages. The choice is yours.

And that's how you save time.



[jqt]:      https://github.com/BorisMoore/jquery-tmpl
[JQuery]:   http://jquery.com
[JIRA API]: https://docs.atlassian.com/jira/REST/latest/
[JQL]:      https://confluence.atlassian.com/jira/advanced-searching-179442050.html
[search]:   https://docs.atlassian.com/jira/REST/latest/#api/2/search
[result-set]:   https://gist.github.com/kichnan/43959db34e0bb26717b7d66d94555c55#file-03-jira-api-result-sample-json
