---
layout: post
title:  "Creating daily status updates email from JIRA"
date:   2016-02-09 21:11:30 +0530
categories: JIRA Tricks
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

Here is a sample JQL [result-set][]. There are a few key things to note in that result set:

*   It brought only the fields mentioned in `fields` parameter above.
*   The `startAt` and `maxResults` properties are used for pagination. By default, JIRA API brings only 50 items per query. So, if you have more than 50 issues to process daily, you should tweak your API call accordingly.
*   There is a `renderedFields` property which contains the actual _rendered_ HTML of the description fields, and not the JIRA wiki syntax as in descriptions in the original `fields`. The JIRA wiki syntax is useless for us.


### Process Result
<a name="processor-fn"></a>The following JavaScript code processes the above JSON and builds me a cleaner JSON that I can use to build my final status mail. Refering to the code below, calling the `processStatus(yourJSONFromAPI)` function will give you the full and final status mail HTML built using [JQuery Templates][jqt]. You may use that HTML output however you want to send your status email to your client/boss.
_NOTE:_ You may need to modify below code for your use.
{% highlight js linenos %}
(function(yourJSONFromAPI) {
    /**
     * This is the API to be exposed which creates the status HTML from given `status` json.
     * @param {json} status - Your json from JIRA API
     * @param {string} checkDate - The date for which the status has to be created,
     *  like if you want to create status for comments from 2 days ago, etc.
     *  Format: yyyy-MM-dd. Default: today's date
     * @return {string} Final status HTML
     */
    function processStatus(status, checkDate) {
        if (!status) return;
        checkDate = checkDate ? new Date(checkDate) : new Date();
        checkDate.setHours(0, 0, 0, 0);

        //initialize final status object after processing `status` and cleaning it up
        var finalStatus = {
            issues: []
        };
        for (var i = 0; i < status.issues.length; i++) {
            //loop through each JIRA issue that came up in my JQL
            var processedIssueDetails = _processIssue(status.issues[i], checkDate);
            if (processedIssueDetails) finalStatus.issues.push(processedIssueDetails);
        }
        
        //generate HTML for all processed JIRA issues, including issue description and comments
        var statusDiv = $('<div></div>');
        $('#jira-status').tmpl(finalStatus).appendTo(statusDiv);
        return statusDiv.html();
    }

    function _processIssue(item, checkDate) {
        //initialize and get basic details required for your status
        var processedIssueDetails = {
            title: item.fields.summary,
            link: getMyIssueLink(item.key),
            description: item.renderedFields.description,
            comments: [] //array to accomodate multiple comments on same issue on the same day
        };
        
        for (var j = 0; j < item.renderedFields.comment.comments.length; j++) {
            //loop through each JIRA issue's comments
            var processedComment = _processIssueComment(item.fields.comment.comments[j], item.renderedFields.comment.comments[j], checkDate);
            //add to comments if valid
            if (processedComment) {
                processedComment.mailToSubject = escape(processedIssueDetails.title);
                processedIssueDetails.comments.push(processedComment);
            }
        }
        //if no comments at all, do not send anything
        return processedIssueDetails.comments.length ? processedIssueDetails : null;
    }

    function _processIssueComment(issueComment, renderedIssueComment, checkDate) {
        //collect relevant issue properties together
        var processedComment = {
            html: renderedIssueComment.body,
            author: issueComment.author && issueComment.author.displayName,
            date: issueComment.created,
            mailToSubject: '', //we'll fill this in parent function
            mailToBody: ''
        };

        //your status prefix to be detected in comments
        var regexStatus = /\&#91;statuscomment\&#93;/i;

        //comments validations
        if (!regexStatus.test(processedComment.html))
            return null; //comment should start with "[statuscomment]"
        var createdDate = new Date(processedComment.date);
        if (createdDate < checkDate) return null; //comment should be of today

        //regex to detect and remove the `statuscomment` to remove it from final HTML
        var regexCommentCleanup = /<span class=\"error\">\&#91;statuscomment\&#93;<\/span>(<br\/>)?/ig;
        //comment cleanup and add to data
        processedComment.html = processedComment.html.replace(regexCommentCleanup, '');
        processedComment.date = ''; //not required anymore
        
        //additional reply-to feature for quick reply to issues in status mail
        processedComment.mailToBody = $(processedComment.html).text();
        if (processedComment.mailToBody) {
            processedComment.mailToBody = ' \n\n____________________\n\n'
                + processedComment.author + ' comment: '
                + processedComment.mailToBody;
            processedComment.mailToBody = escape(processedComment.mailToBody);
        }
        return processedComment;
    }

    function getMyIssueLink(key) {
        return "https://myproject.atlassian.cloud/browse/" + key;
    }

    function htmlEncode(value) {
        /// Explanation: Create an in-memory div, set it's inner text (which jQuery automatically encodes)
        /// then grab the encoded contents back out. The div never exists on the page.
        return $('<div/>').text(value).html();
    }

    return processStatus(yourJSONFromAPI, "2016-05-19");
    //You may use the returned HTML as an email using whichever technique you prefer
})(yourJSONFromAPI);
{% endhighlight %}

And here are the jQuery templates used to generate the HTML.
{% raw html %}
    <script id="jira-comments" type="x-jquery-tmpl">
        <table border="0" cellpadding="1" cellspacing="1" class="issue-article" style="width: 100%;">
            <tr>
                <td>{{html description}}</td>
            </tr>
            {{each comments}}
            <tr>
                <td class="comment">
                    {{tmpl($value) "#jira-comment"}}
                </td>
            </tr>
            {{/each}}
        </table>
    </script>
    <script id="jira-comment" type="x-jquery-tmpl">
        <table class="comment-head" width="100%">
            <tr>
                <td align="left"><span class="title" style="font-weight: 600; float:left;">Comment:</span></td>
                <td align="right">
                    <span class="author"> (By ${author})</span>&nbsp;
                    <a class="reply-to" href="mailto:[ReplyToEmail]?Subject=RE:%20${mailToSubject}&Body=${mailToBody}">Reply &gt;</a>
                </td>
            </tr>
        </table>
        <div>{{html html}}</div>
    </script>
    <script id="jira-status" type="x-jquery-tmpl">
        <html>
            <body>
            {{each issues}}
                <a href="${link}">${title}</a><br />
                {{tmpl($value) "#jira-comments"}}
            {{/each}}
            </body>
        </html>
    </script>
{% endraw %}


## Ending Note
The output `finalHTML` generated out of this exercise can be either sent as email (through whichever means you like), or maintain it as a web-page. Your choice.

And that's how you save time!


[jqt]:      https://github.com/BorisMoore/jquery-tmpl
[JQuery]:   http://jquery.com
[JIRA API]: https://docs.atlassian.com/jira/REST/latest/
[JQL]:      https://confluence.atlassian.com/jira/advanced-searching-179442050.html
[search]:   https://docs.atlassian.com/jira/REST/latest/#api/2/search
[result-set]:   {{ site.baseurl }}/assets/jira-status-sample.json
