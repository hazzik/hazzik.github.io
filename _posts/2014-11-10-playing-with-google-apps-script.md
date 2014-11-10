---
title: Playing with Google scripts
date: 2014-11-10T22:40:49+13:00
tags: 
  - JavaScript
  - Google
  - Google Apps Script
---
Google has an Apps Script platform which allows you yo write scripts for most of Google products, such as Gmail, Drive, Spreadsheets, etc. The google script (extension .gs) is an JavaScript language subset. Google provides online editor for scripts, which is available at [script.google.com](https://script.google.com). The editor is an online IDE with basic autocompletion support, which is really nice. But I prefer to write Google Scripts in some mature desctop editors such as Visual Studio or at least Sublime Text.

I came across need of writing Google Script, when I wanted to sort my GitHub notification and remove some subscription emails.

Ok, lets start. First, open the [script.google.com](https://script.google.com) page in your favourite browser. The page will ask you to choose an project type, or open recent project.

![Step 1](/public/images/gs/step1.png)

I'm going to select the "Blank Project" option.

![Step 2](/public/images/gs/step2.png)

If you click on "Untitled project" you'll be able to give a sensible name for your project.

![Step 3](/public/images/gs/step3.png)

![Step 4](/public/images/gs/step4.png)

Then you can write a code. As manipulating emails means a lot of work with collection, so it would be nice to have something like [underscore.js](http://underscorejs.org/) or [lodash.js](https://lodash.com/) in place. I'll go with Lo-Dash. Just go to [lodash.com](https://lodash.com) and dowload [minified modern build](https://raw.github.com/lodash/lodash/2.4.1/dist/lodash.min.js). 

When you have Lo-Dash downloaded you need to import it to your Google Apps Script project. Press _File_->_New_->_Script File_

![Step 5](/public/images/gs/step5.png)

Enter the name of the file: LoDash

![Step 6](/public/images/gs/step6.png)

Then copy and paste content of lodash.min.js to LoDash.gs and save the file.

And finally there is my code to sort Github label in the Gmail.

```JavaScript
function split() {
  // I have a filter in Gmail, which will put all emails from notifications@github.com 
  // to "GitHub" label and skip inbox for them
  // Get the 'GitHub' label
  var label = GmailApp.getUserLabelByName('GitHub');

  // Group all threads by Project Name. All notification emails have Project Name 
  // in square brackets in the email title. 
  var groups = _.groupBy(label.getThreads(), function(t) {
    var subject = t.getFirstMessageSubject();

    var tag = /\[([^\]]+)\]/.exec(subject);
    if (tag) {
      return label.getName() + '/' + tag[1];
    }
    return label.getName();
  });

  _.forEach(groups, function(threads, l) {
    if (label.getName() != l) {

      // Get or create a label for a project 
      var new_label = GmailApp.getUserLabelByName(l) || GmailApp.createLabel(l);
      
      // Label.addToThreads / Label.removeFromThreads has an restriction of
      // 100 threads per call, so we need to page our threads.
      var pages = _.groupBy(threads, function(t, i) { return (i / 100) >> 0; });
      
      _.forEach(pages, function(page) {
        new_label.addToThreads(page);
        label.removeFromThreads(page);  
        
        // If you call API to often you can reach call limit and script will stop execution
        Utilities.sleep(100);
      });
    }
  });
}
```

When we have code in place we want to add a trigger to run the script regularly. Click _Resources_->_Current project's triggers_

![Step 9](/public/images/gs/step9.png)

Then click on "No triggers set up. Click here to add new now."

![Step 10](/public/images/gs/step10.png)

Select the frequence you want your trigger to run. I've chosen to run this script every five minutes. Click Save. The script will ask you for Authorisation.

![Step 11](/public/images/gs/step11.png)

Press "Continue". And then in the pop-up window review and Accept permissions for your script.

![Step 12](/public/images/gs/step12.png)

Happy scripting!