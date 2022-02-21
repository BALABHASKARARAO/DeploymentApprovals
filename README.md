# DeploymentApprovals
Github Actions For ChatOps
---------------------------


Basically in our Github repo settings we can' find environments. So if you want to  deploy your application on diffrent environments like Dev, Qa, test or Production. 

In build.yml
============
whenever you do a build on your github actions you would do all the steps like testing building and so no. Later we do publish an artifact of the build.

In this scenario normally we do another job here for the deployment using the environments. Since we not have envvironments is using somthing called issueops. 


when we have thea pproval we do right after the build and then approve the deployment to the next envirement. For automate the action on issues you acan search in marketplace has create an issue then will give some code for issue action like below.
```
- name: Create an issue
        uses: JasonEtco/create-an-issue@v2.6.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ENVIRONMENT: Dev
          RUNNUMBER: ${{ github.run_number  }}
        with:         
          filename: .github/deployment-approval.md
```          

Here we can set the env and the issue run number with a file name.


deployment-approval.md
======================

this is issue templet  here i gave title and label we can give whatever the lablel here. The title is for Deployment Approval Required for perticular env that what ever we want to deploy and it will evaluvate by github actions. The action will evaluate these in
the template so we will be able to pass variables basically to the issue creation through that github actions.
here the issues saying deployment approve are required for dev for prod for qa and so on. you can use the payload and the sender and login so basically if i'm gonna run this action or to create this issue this will be deployment approval requested from my user which is 'Username' and then in my case what i want to do is if a user comment on this issue saying approved for example i want the deployment to take place now there are many other ways that you can achieve that i've seen people doing it like if you close the issue then get the deployment going
if you you know add a label. for
example, the issue you know with a specific label then the deployment is approved there are multiple ways and if a user comment approved to the issue then make it deploy and then have probably the most important part. Because this is related on how we will consume this data but
you know think about what you're doing basically splitting the build and the deploy so you need basically a way to pass some metadata between the two workflows to be able to complete the job.
---
title: Deployment Approval Required for {{ env.ENVIRONMENT }}
labels: deployment-requested
---

Deployment Approval requested from {{ payload.sender.login }}.

Comment "Approved" to kick the deployment off.


=== DON'T CHANGE BELOW THIS LINE
```json target_payload
{
    "runNumber":  {{ env.RUNNUMBER }},
    "environment": "{{ env.ENVIRONMENT }}"
}
```



build.yml
===========
```
 env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ENVIRONMENT: Dev
          RUNNUMBER: ${{ github.run_number  }}
        with:         
          filename: .github/deployment-approval.md
          
```          
github token this is used by this action to be able to create the issue in my repo and then i have the environment variables `ENVIRONMENT: Dev` that i've used in my in my template remember i have the environment and i have the run number `RUNNUMBER: ${{ github.run_number  }}` and again i'll explain in a second why i use it i'm using those variables just uh bear with me with for a second on this so we said that we want to deploy to the next environment and the next environment in this case will be the dev environment so let's just put here the dev
and then i want the run number and the run number i can consume it from a variable that is in the context of the github action so it's github.run number there's a difference between run number and run id you can use either ones i prefer round number because it's it's
easier to understand and to to consume basically a round number is a is a progressive number that is that is assigned to a specific
github action workflow for example this is the ci github action workflow so the first time i execute this will be run number one then
the next one will be run number two and so on.

Add Approvals to Deploy with chatops (Deploy.yml)
=================================================


so the first thing i want to do is trigger these and in my case as i mentioned before i want to start the deploy when the user comment on that specific issue with the with the word 'approved' so i need to use is the issue comment type of trigger
```
on:
  issue_comment:
    types: [created]
```
but not any issue comment because there are different types of this event so i just want to use the created event because i want only to trigger this whenever a user types the 'approved' keyword and then i need to do few other things including checking that only the comment approved triggers this because if i just leave the trigger like this this will be trigger for any single comment. But i want only the approved one so let's let's create a job i'll call it 'parse' and
to be able to discriminate only on the comments i want which is the approved one i need to use a conditional so i'm gonna say and the conditionals work like this. 
if the event payload contains comment and the comment is approved then run this job otherwise the issue will still the the action workflow will still be triggered but if any other comment is found it'll basically do nothing we'll just evaluate this and and exit.
```
if: ${{ github.event.comment.body == 'Approved' }}
```

This one even though the trigger is issue comment and it can be triggered also for pull requests the issue comment event is also created whenever a comment is done on a pull request.
```
jobs:  
  parse:
    if: ${{ !github.event.issue.pull_request && github.event.comment.body == 'Approved' }}
    runs-on: ubuntu-latest
    outputs:
      deploy-environment: ${{ fromJSON(steps.issue_body_parser.outputs.payload).environment }}
      ci-run-number: ${{ fromJSON(steps.issue_body_parser.outputs.payload).runNumber }}
```

so i'm gonna copy this and what these action does basically reading the whole issue and going to consume this part which is json. the action can work with both json or yaml
```json target_payload
{
    "runNumber":  {{ env.RUNNUMBER }},
    "environment": "{{ env.ENVIRONMENT }}"
}
```
I just decided to use json but if you want to use yaml i think the default is json
and if you see here this is not just json but i basically tagged this section because you can have even multiple section this is something i don't have here but if you want to have multiple values or
multiple sections of metadata in your issue you can tag them and consume only the one you actually need in that specific moment.


i'm calling these get issues issue data for example
```
 - name: Get Issue Data
        uses: peter-murray/issue-body-parser-action@v1
        id: issue_body_parser
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          issue_id: ${{ github.event.issue.number }}         
          payload_marker: target_payload 
```

i do need to give the the github token because otherwise it doesn't have access to the issues part and i need to identify the issue that contains the metadata this is this issues id and
again we can rely on something that github provides for us which is this get up event issue number `issue_id: ${{ github.event.issue.number }}  ` because we we are triggering this by an issue comment so we have a payload for this event for this
trigger which is the issue itself and in the issue object we do have the number of the issues so this will allow this action to identify the issue that we need to consume again we can specify the type of payload which is yaml or json in my case is json so it's the default as i mentioned before this target payload `payload_marker: target_payload ` is not a keyword this is just whatever you want to call it as long as you have the same value here this works.


this one this this action will actually read this section and basically consume the json and those data to consume data in other jobs or steps i'm gonna give this step an id and let's call it issue body parsers `id: issue_body_parser`if you're not
familiar with these they are just unique names that you can specify and you can assign to a specific action. i'm gonna add
here another another action which is comment on issue
```
- name: Comment on Issue      
        uses: peter-evans/create-or-update-comment@v1
        with:
          issue-number: ${{ github.event.issue.number }}
          body: 'Deployment Initiated 🚀'
```
so this is the the peter evans create because it's create or update comment so gain here just pass the issue number and specify the comment body.
basically this is just a way to notify the users that something is going on so. whenever this will run i expect to have
this deployment initiated message as a comment in my issue.

The another job and i'm going to call it deploy dev for example
```
 deploy-dev:
    needs: [parse]
    if: ${{ needs.parse.outputs.deploy-environment == 'Dev' }}
    runs-on: ubuntu-latest
```
in which i will do the actual deployment now first thing i want to specify here is that this needs to run only after the parse job got the metadata right so this job that it needs the parse job so this means that this can run only after the parse has completed and will run only if the parts succeeded since have other deployment probably in the same issue like for example deploy to production deploy to qa and so on so a way to identify when need to run this and do have if you remember in my issue do have the environment so the environment is being read by this ` "environment": "{{ env.ENVIRONMENT }}" `
```
- name: Get Issue Data
        uses: peter-murray/issue-body-parser-action@v1
        id: issue_body_parser
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          issue_id: ${{ github.event.issue.number }}         
          payload_marker: target_payload 
```
get issued data that a way to pass those data from this step which is in another job to this job then you know execute this only if whatever which is the environment it's for example dev `if: ${{ needs.parse.outputs.deploy-environment == 'Dev' }}`

something like that now this whatever needs to be the the environment that comes from here so how can i consume the environment or the value of the variable that i read in here in the other job well i need to specify some output parameters or output data out of this job so this is something that has been introduced a while back and we now have these outputs section in the job definition basically so let's call it deploy environment

```
  outputs:
      deploy-environment: ${{ fromJSON(steps.issue_body_parser.outputs.payload).environment }}
      ci-run-number: ${{ fromJSON(steps.issue_body_parser.outputs.payload).runNumber }}
```
now this is something that will be consumed here as an output parameter of this job and we'll see in a second how to consume
it but now first thing i need to get the environment data out of this `issue body parser`  add also
the other parameter that we have i'm gonna call it ci run number and the syntax is exactly the same i have the from json the name of the output parameter and run number which once again is the same parameter we have over here ` ci-run-number: ${{ fromJSON(steps.issue_body_parser.outputs.payload).runNumber }} ` 

now with this syntax those two variables basically those two parameters will be made available to other jobs and in this case the way that is made available is through the needs because the only thing that basically you know putting connection this deployed dev job with parse job and it's like this is needs it comes from here `if: ${{ needs.parse.outputs.deploy-environment == 'Dev' }}` the name of the job that is underneath so parse and again outputs and the name of the variable or the output parameter so i have this deployed-environment which is the same i have here in this way i can evaluate and run this job only if the environment specified in the issue that is being uh used as a trigger for this is dev.


remember that we have the build done in the other github actions workflow right so and in there we do publish an artifact so we need a way to basically download the build artifact that we have created in the build section

```
- name: Download workflow artifact
        uses: dawidd6/action-download-artifact@v2.15.0
        with:
          workflow: build.yml
          repo: naveen-k-u-n/DeploymentApprovals
          run_number: ${{ needs.parse.outputs.ci-run-number }}
          name: your-artifact-name
          path: ${{ github.workspace }}
```

we need to specify the workflow `workflow: build.yml` i want to consume my artifact from and i can use like in this case the name of the file so build.yaml or you can use the id of that of that workflow that in my case was ci and here we know now why i need the run number ` run_number: ${{ needs.parse.outputs.ci-run-number }}`  i need a way to identify the specific action or the specific workflow that generated that artifact that i'm going to download.

ci workflow that produce more artifacts you probably want to specify the name over here `name: your-artifact-name` and then you can specify the path where you want this artifact to be to be downloaded

Deploy to production scenario
=============================
in a scenario when we use the environments we would have you know deployment to dev and then we approve and we deploy to qa and when we approve and we deploy to production and so on. let's see how we can do this but basically with everything we've seen we should be able to do again this specific workflow.
for example when i finish deploying in dev and i know everything is successful i should create another issue for the next environment right so to do so we can use the exact same approach i had in the previous one when i created the first issue for the dev environment so i use exactly the same action so let's say i want to after dev i want to deploy to production so i'm gonna specify the environment production

```
- uses: JasonEtco/create-an-issue@v2
        name: Create approval Issue for Prod
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ENVIRONMENT: Prod
          RUNNUMBER: ${{ needs.parse.outputs.ci-run-number }}
        with:
          filename: .github/deployment-approval.md
```
exact same action the json etc one and i'm gonna just change the environment `ENVIRONMENT: Prod ` to production and of course i need to have the run number the run number `RUNNUMBER: ${{ needs.parse.outputs.ci-run-number }}` in the one
we've seen before it was easier to get because uh it came from the build right so we added in the context in here we can use again the crm number variable that we have and this is something that is very important and i want to stress on and this is usually the case whenever you do ci cd you want to build once and then deploy multiple times the same exact artifact maybe you can change the configuration because maybe you when you deploy to the to dev you have a db connection that is one thing when you deploy to another environment that the credentials or connection are different but the artifact you want to deploy is exactly the same and this is why i'm going to use the same run number that i had before when i deployed in dev for example also for deploying to production and this is why i'm consuming it from the needs variable because it's the one that it's parsed from the issue and i'm gonna use the exact same template for doing so now when i have that i can proceed and basically duplicate these to deploy in production
```
deploy-prod:
    needs: [parse]
    if: ${{ needs.parse.outputs.deploy-environment == 'Prod' }}
    runs-on: ubuntu-latest    

```
right so i'm gonna have my deploy prod for example that once again needs the parse job to execute because it needs to
consume the data from it and of course in this case i need to have it as prod. once again we have our steps

```
steps:
               
      - name: Download workflow artifact
        uses: dawidd6/action-download-artifact@v2.15.0
        with:
          workflow: build.yml
          repo: naveen-k-u-n/DeploymentApprovals
          run_number: ${{ needs.parse.outputs.ci-run-number }}
          name: your-artifact-name
          path: ${{ github.workspace }}
          
```
and we can do exactly the same thing we can download our artifact from you know from the build we had before and we can simulate the deployment to production right so basically we have the exact same thing so in this case we deployed to dev if we are coming from dev uh in the issue in this case with the we deployed to production if we're coming to production the last thing we need to do is whenever we finish with the deployment and of course sorry in this case since production is ideally my last environment i'm not going to create another issue like i do in development because there's nothing to deploy after production right in theory so i'm not going to do that last thing we have to do and we have to add here is as i mentioned before closing the issue

```
  close-issue:
    needs: [deploy-dev, deploy-prod]
    if: ${{ always() }}
    runs-on: ubuntu-latest  
```
if you remember i had to close the i had to close the issue um manually because after deployment nothing was happening so what i want to do is closing the issue automatically and again this is a personal preference you may want to have for example another another uh you know another comment like this one saying deployment uh completed and in fact let's just add here another one that says deployment completed with perhaps uh star there we go and same we could do in production um
but then if we start you know we take these we put it in production we have duplication of code and then we can we should have another task over here as i mentioned before to close the issue um and again we should do it here and we should do it in production and this will be duplicated for each and every you know each and every environment we want to deploy so i wanna make it a little bit a little bit more elegant so this duplicated code for commenting on an issue or closing the issue i want to move it to another job i'm gonna create a job that i'm close i don't know close issue again it runs let me copy this runs on whatever and now we want to execute these and follow follow follow my reasoning this needs to be executed
at the end of every execution right if it's successful so whatever whether we are deploying to dev to test qa or production we want the issue to that specific environment to be closed if everything is successful

```
steps:
      - name: Close Issue
        if: ${{ needs.deploy-dev.result == 'success' || needs.deploy-prod.result == 'success' }}
        uses: peter-evans/close-issue@v1.0.3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          comment: 'Deployment Completed 🌟'
    
```
