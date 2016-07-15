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


Step 2: Building from command line
---
According to the [Command line reference](https://docs.unity3d.com/Manual/CommandLineArguments.html) there's no option to build an Android player directly. Instead, we'll have to use a custom method to publish our APK, and call it with the ```-executeMethod``` argument.
Our build command looks like this:
```
"C:\Program Files\Unity\Editor\Unity" 
-projectPath "C:\Path\To\Project" 
-logfile "C:\Path\To\Project\build.log" 
-executeMethod AndroidBuilder.Build
-batchmode
-quit
```

In Unity, under the "Scripts" folder (create it if missing), add a new C# script called "AndroidBuilder" and open it with your preferred IDE.

- the class must be ```static```
- you don't need the default methods ```Start``` or ```Update``` because we'll only call this from the command line. In our case the starting point is a method called ```Build()```
- ~~~
public static void Build()
    {
        var outputPath = Application.dataPath + "/../myProject.apk";
        Debug.Log("Publishing Android Player in " + outputPath);
#if UNITY_EDITOR
        string[] scenes = EditorBuildSettings.scenes.Select(scene => scene.path).ToArray();
        BuildPipeline.BuildPlayer(scenes, outputPath, BuildTarget.Android, BuildOptions.Development);
#endif
    }
~~~
- note the ```#if UNITY_EDITOR``` clause: this is because the Unity API is separated into UnityEditor and UnityEngine, and the compiler only knows how to deal with UnityEngine. We can still use classes from the UnityEditor assembly, as long as we wrap them in that clause. [http://unityready.com/use-unity_editor-write-editor-code-game-scripts/](reference)


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
		* ```-projectPath``` is where your project files are checked out by Teamcity
		* ```-logfile``` will generated a log for each build that we can later add to the artifacts list
		* ```-executeMethod``` will run the custom "publish to android" script
		* ```-batchmode``` makes the build silent instead of launching the editor
		* ```-nographics``` allows the build to run on servers without GPU
		* ```-runEditorTests``` runs unit tests
		* ```-quit``` ensures unity exits after building the project (it hangs by default, because... reasons?)
		
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
Command line builder accepts arbitrary arguments: it will just ignore them. This lets us add a ```-buildNumber``` argument to the command line that we will retrieve in our custom publication script to define the APK version number.
* In Administration > Root > MyProject > MyProject_CI > Build Step  edit the current command line step to add the following argument: ```- "%build.number%" ```
* In the Unity project, we will add to the AndroidBuilder class
* retrieve the build number from command line:
* ~~~
private static string GetArgumentFromCommandLine(string parameterName, string defaultValue = null)
    {
        var arguments = Environment.GetCommandLineArgs();
        var index = Array.IndexOf(arguments, "-"+parameterName);
        if (arguments.Length == 0 || index == -1)
        {
            if (defaultValue != null) return defaultValue;
            throw new ArgumentException("Could not find argument '"+ parameterName + "' in command line.");
        }
        return arguments[index + 1];
    }
~~~
* modify the Build() method:
* ```var outputPath = Application.dataPath + "/../myProject."+GetArgumentFromCommandLine("buildNumber")+".apk";```