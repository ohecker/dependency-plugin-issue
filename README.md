## Issue with dependency:copy-dependencies excludeTransitive flag under certain circumstances

This demonstrates an issue where under certain circumstances setting `excludeTransitive` to true within dependency:copy-dependencies might result in even direct dependencies not being copyied
(thus resulting in no dependencies being copyied at all).

The issue is only triggered under certain circumstances and depends on overall configuration of the project setup and history (e.g. whether artifacts are already existing in Maven cache 
or in the remote repository).

The demo code is a modified version of https://github.com/jimonthebarn/maven-license-plugin-issue which initially demonstrated an issue in dependency resolution when using 
the license maven plugin (which was fixed in v1.17 of that plugin; see issue https://github.com/mojohaus/license-maven-plugin/issues/145)

Even though the given setup uses the license-maven-plugin to trigger the issue, the issue seems to be lying within the dependency plugin (or used general maven libraries).

HINT: Further debugging indicates that within AbstractDependencyFilterMojo.getDependencySets() the used ProjectTransitivityFilter does not work as expected. This is caused because under the given
circumstances the version info of checked (SNAPHOT) artifacts contains timestamp information which makes the equals check fail.

Even though I could up to now only reproduce the issue under very specific conditions given below, it seems that this issue is the root cause for failing builds of several components on our
side. Possibly the problem can be reproduced in a wider range of situations but I did not yet find a way to do so.

## Synopsis

The goal `copy-dependencies` with flag `excludeTransitive` set to true might even not copy any direct dependencies.
The problem could be reproduced with versions up to 3.3.0 of the plugin.

## General setup to demonstrate the issue

I have a multi-project setup with a parent child hierarchy. There are two subprojects:

- submodule1 (this is a java module having a dependency on submodule 2; the dependency plugin is used to copy the artifact of submodule2)
- submodule2 (this is a dummy java module not containing any dependencies)

## Steps to reproduce the issue

### Setup environment
- Clone https://github.com/ohecker/dependency-plugin-issue (checkout to master)
- You need to have a maven repo where you can deploy to. So possibly setup some test repo (see e.g. https://reposilite.com/)
- Configure `repositories` and `dependencyManagement` in the main pom accordingly; possibly configure credentials in your settings.xml (at least for deploying)

### Testing the behaviour
The following table describes the steps to observe the (mis)behavour

| Step | Action | Result | OK/NOK | Remark |
|------|--------|--------|--------|--------|
| 1 | execute `mvn clean package` on command line | project is built; *dependency was copied* (see below how to check this) | OK | This is how it should work! |
| 2 | execute `mvn clean install` on command line | project is built; *dependency was copied*; artifacts get deployed to local cache | OK | |
| 3 | execute `mvn clean deploy` on command line | project is built; *dependency was copied*; artifacts get deployed to repo | OK | |
| 4 | execute `mvn clean package` on command line | project is built; *dependency was copied* | OK | still OK here |
| 5 | remove the artifacts from the local maven cache (possibly delete ~/.m2/repository/de/jotb) | ||
| 6 | execute `mvn clean package` on command line | project is built; *dependency NOT copied* | NOK | THIS IS THE MAIN ERROR. Note that at the beginning of the build submodule2-0.0.2-SNAPSHOT.jar is downloaded from the repo. All below is just to demonstrate what seems to be of influence here  |
| 7 | execute `mvn clean` then `mvn package` on command line |project is built; *dependency was copied* | OK | If `clean` and `package` is done separately it works ... |
| 8 | execute `mvn clean package -Dlicense.skipAggregateAddThirdParty=true` on command line | project is built; *dependency NOT copied* | NOK | Even when the main processing of the license plugin is skipped the problem still persists! |
| 9 | comment maven-license-plugin definition out in main pom (lines 28-31) | | |
| 10 | execute `mvn clean package` on command line | project is built; *dependency copied* | OK | So without the license plugin no problem!  |
| 11 | undo step 9 | | |
| 12 | comment out line `<excludeTransitive>true</excludeTransitive>` in submodule1/pom.xml | | |
| 13 | execute `mvn clean package` on command line | project is built; *dependency copied* | OK | without the flag it works  |
| 14 | undo step 12 | | |
| 14 | execute `mvn clean install` on command line | project is built; *dependency NOT copied* | NOK | Problem still there with initial setup |
| 15 | execute `mvn clean install` on command line | project is built; *dependency copied* | OK | The update of the repo cache in step 14 made the problem disappear (you might restart loop at step 3) |

#### Check *dependency was copied*

The expected behaviour is:
- maven logs
>  [INFO] --- maven-dependency-plugin:3.3.0:copy-dependencies (copy-dependencies) @ submodule1 ---
>  
>  [INFO] Copying submodule2-0.0.2-SNAPSHOT.jar to SOMELOCATION\dependency-plugin-issue\submodule1\target\submodule2-0.0.2-SNAPSHOT.jar

- the file `submodule2-0.0.2-SNAPSHOT.jar` exists in `submodule1\target`

