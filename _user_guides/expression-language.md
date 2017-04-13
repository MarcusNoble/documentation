---
layout: post
---

This guide should include:

- the power of the expression language
- examples
- troubleshooting tips


The expression language is a powerful tool that you can use to add logic and decision-making to your pipelines. While a lot of the time you will probably use it to evaluate variables, it can do a lot more. You can write straight Java/Groovy into it. This means you can do transformations, filters, maps, etc. You can use it to branch your pipeline into different directions.

Some of the most common uses include:
- Getting build information from Jenkins
- Passing image names from one stage to another
- Retrieving a user's manual judgement response


Before we go into examples and troubleshooting, check out the guide on spinnaker.io for an detailed overview: [http://www.spinnaker.io/docs/pipeline-expressions-guide](http://www.spinnaker.io/docs/pipeline-expressions-guide)


## Examples

You can find the expression language used in the examples within the [baking images](baking_images.md), [deploying](deploying.md), [working with Jenkins](working_with_jenkins.md) and [finding images](find_images.md) guides.

## Troubleshooting

Sometimes using the expression language doesn't go as anticipated. Here are some of the common issues:

### Autocomplete popup menu

Sometimes the UI doesn't display the autocomplete popup menu. This is because it doesn't have the context to do so. Try filling in as much of the details as you can for the stages in your pipeline, then run the pipeline. After it has ran (it's okay if it failed), go back to where you were trying to use the expression language and see if the autocomplete popup menu is displayed.

### Expression doesn't get evaluated

Sometimes your expression is just printed out plainly and not evaluated. Usually this happens when the expression is invalid and/or can not be resolved and does not mean that Spinnaker isn't trying to evaluate it. Try double checking that what you are referencing does exist. To check that it does exist, go to your pipeline's execution details and click the 'Source' link in the very bottom right hand corner. 

For example:
![](https://d1ax1i5f2y3x71.cloudfront.net/items/0O261w2H3N3H043r0l33/Image%202017-04-03%20at%203.37.57%20PM.png)

You should see a page full of JSON. It doesn't print in a very readable format, so you may want to copy and paste it into a text editor or another tool that will help you read it (I usually just curl this URL and pipe it to `jq`). You can navigate to the JSON field 'stages' for a list of stages in your pipeline. These stages are not necessarily in order. In the stage you'll see another field called 'context'. This is the information avaliable to the expression language. Make sure what you are referencing is in the context of the appropriate stage.