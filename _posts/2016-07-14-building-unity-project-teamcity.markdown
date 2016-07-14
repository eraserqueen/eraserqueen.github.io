---
layout: post
title:  "Building a Unity3D VR project with Teamcity"
date:   2016-07-14 09:55:45 +1000
categories: unity ci
---

The project
---
A virtual reality game for the Samsung GearVR, using Unity3D, Oculus SDK, Android SDK.


Before we start
---
I have the project in a public repo on github, and I want Teamcity to build it on every check-in and run unit tests.

Step 1: Unit testing in Unity
---
_TODO: talk about Test project_


Step 2: Publishing the android APK file with a custom script
---
_TODO: talk about Runner.Start_


Step 3: Setting up the build server
---

I found out there's no easy way to avoid Windows for this: I first tried setting up a Linux machine, but Teamcity needs a local instance of Unity to build the project, and their Linux version is experimental. As for MacOS, although unity3d is supported, the Oculus SDK we use for the VR part of the project is not.

Intall Teamcity Server
===
* Run the [windows installer](https://www.jetbrains.com/teamcity/), you can keep all default settings. I like to hostfile my server to be something like http://teamcity.local
* Visit the Teamcity home, set up your admin account

Install Teamcity Agent
===
_For simplicity I am hosting the server and the build agent on the same machine but you can separate them of course._
* Click on "Agents" in the top navigation, then "Install Build Agents" in the top right corner, choose "MS Windows Installer"
* Run the installer, again all defaults is fine (make sure that the server url is correct if you chose a custom one in the previous step), we're only going to use one agent.

Create project and configuration
===
* In "Administration" click "Create Project" and give it a name
* In Administration > Root > MyProject
	* under "Build configurations" click "Create Build Configuration", call it something like "MyProject_CI"
	* In Administration > Root > MyProject > MyProject_CI
		* in the left-hand menu click "Version Control Settings", then "Attach VCS root" and fill in your repository credentials
		* in the left-hand menu click "Triggers", then "Add a new trigger". Choose "VCS trigger" and click "Trigger a build on each check-in" then save.
		* in the left-hand menu click "Build Step" then "Add a build Step". Choose "Command Line" and add the following:
		* ```
			"C:\Program Files\Unity\Editor\Unity" 
			-projectPath "%teamcity.build.checkoutDir%" 
			-logfile "%teamcity.build.checkoutDir%\unity.build.%build.number%.log" 
			-executeMethod Runner.Start 
			-batchmode 
			-nographics 
			-runEditorTests 
			-quit
		```
		_TODO: explain each argument_

Display Test Results
===
* In Administration > Root > MyProject > MyProject_CI, in the left-hand menu click "Build Features" then "Add build feature".
* Select "XML report processing"
* report type: NUnit
* Monitoring rules: ```%teamcity.build.checkoutDir%\EditorTestResults.xml```

List Artifacts
===
* Let your build run once for easier selecting
* In Administration > Root > MyProject > MyProject_CI, click "general settings"
* Click on the "folder tree" icon next to "Artifacts path" and select the APK file
* the APK will now show as an artifact of your future builds for easy downloading

Step 4: Use Teamcity's build number to version your APK
---
_TODO_