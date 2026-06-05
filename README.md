#   One Click Release Workflow


##  Steps:

All these steps should be completed in a single click.

### Build Setup Steps

* Create ReleasePlanAdmission in gitlab/konflux-release-data Repo 
* If there is any new component being added then add pyxis  entry as well. 
* Generate the RPA/Pyxis from Hack Repo and copy them to konflux-release-data to avoid human error


* Create New Release/Patch from Hack Repo
  * This should include the updating the upstream branch
  *   Watch for PRs konflux config created for Hack
  * Merge All Hack PRs
  * Apply updated configuration on Konflux
  * Watch PRs generated in p12n Repos
  * Wait for  konflux build to finish
  * If  Build is green then merge the PR
  * If build is failed then  Notify RC group
  * Bump OLM version in operator
  * Currently in project.yaml but we can read this from dockerfile as docker file is updated with latest patch version
  * Olm script should read the  
  * current version from  bundle.dockerfile
  * Min and Max OCP version from olm/config.yaml


### Update Sources Workflow

* If Konflux builds fails then Notify Slack
* If Konflux builds are not triggered then add /retest comment (keep the gap of 30 mins between 2 retest comments)
* Once All workflows are successful then merge PRs
* Build All Components
* Component build is triggered as soon as PR is merged
* If any of the component  build is not triggered because of API issue then retrigger the build
* If  any of the build fails Notify Slack
* Once all the components are built then  update operator project.yaml with core-snapshot

##  Dev Release Steps
* **Release Bundle**
  * Update CSV  for dev registry
  * Build Bundle
  * Release

* **Release Index** 
<br/><b>_Release of index should be indendent of release version.
  Every index image contains all  the bundles supported on OCP version_
</b>
    * Update OLM with latest bundle snapshot
  * Render OLM
  * Build Index Images
  * Release Tests
  * Release
* **Slack Notification**

##  Stage/Production Release Steps

* Update CSV with latest core snapshot
* Build Bundle
* Update OLM with latest bundle snapshot
* Render OLM
* Build Index Images
* Release Tests
* Release
* Slack Notification





