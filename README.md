PSDepend
========

This is a simple PowerShell dependency handler.  You might compare it to `bundle install` in the Ruby world or `pip install -r requirements.txt` in the Python world.

## Defining Dependencies

Store dependencies in a PowerShell data file, and use *.depend.psd1 to allow Invoke-PSDepend to find your files for you.

What does a dependency file look like?

### Simple syntax

Here's the simplest syntax.  If this meets your needs, you can stop here:

```powershell
@{
    psake        = 'latest'
    Pester       = 'latest'
    BuildHelpers = '0.0.20'  # I don't trust this Warren guy...
    PSDeploy     = '0.1.21'  # Maybe pin the version in case he breaks this...
}
```

And what PSDepend sees:

```
DependencyName DependencyType  Version Tags
-------------- --------------  ------- ----
psake          PSGalleryModule latest
BuildHelpers   PSGalleryModule 0.0.20
Pester         PSGalleryModule
PSDeploy       PSGalleryModule 0.1.21
```

There's a bit more behind the scenes - we assume you want PSGalleryModule's unless you specify otherwise, and we hide a few dependency properties.

### Flexible syntax

What else can we put in a dependency?  Here's an example using a more flexible syntax.  You can mix and match.

```powershell
@{
    psdeploy = 'latest'
    
    buildhelpers_0_0_20 = @{
        Name = 'buildhelpers'
        DependencyType = 'PSGalleryModule'
        Parameters = @{
            Repository = 'PSGallery'
        }
        Version = '0.0.20'
        Tags = 'prod', 'test'
        PreScripts = 'C:\RunThisFirst.ps1'
        DependsOn = 'some_task'
    }

    some_task = @{
        DependencyType = 'task'
        Target = 'C:\RunThisFirst.ps1'
        DependsOn = 'nuget'
    }
    
    nuget = @{
        DependencyType = 'FileDownload'
        Source = 'https://dist.nuget.org/win-x86-commandline/latest/nuget.exe'
        Target = 'C:\nuget.exe'
    }
}
```

This example illustrates using a few different dependency types, using DependsOn to sort things (e.g. some_task runs after nuget), tags, and other options.

You can inspect the full output as needed.  For example:

```powershell
# List the dependencies, get the third item, show all props
$Dependency = Get-Dependency \\Path\To\complex.depend.ps1
$Dependency[2] | Select *
```

```
DependencyFile : \\Path\To\complex.depend.psd1
DependencyName : buildhelpers_0_0_20
Name           : buildhelpers
Version        : 0.0.20
DependencyType : PSGalleryModule
Parameters     : {Repository}
Source         : 
Target         : 
AddToPath      : 
Tags           : {prod, test}
DependsOn      : some_task
PreScripts     : C:\RunThisFirst.ps1
PostScripts    : 
Raw            : {Version, Name, Tags, DependsOn...}
```

## Exploring and Getting Help

Each DependencyType - PSGalleryModule, FileDownload, Task, etc. - might treat these standard properties differently, and may include their own Parameters.  For example, in the BuildHelpers node above, we specified a Repository parameter.

How do we find out what these mean?  First things first, let's look at what DependencyTypes we have available:

```powershell
Get-PSDependType
```

```
DependencyType  Description                                                                             DependencyScript                                    
--------------  -----------                                                                             ----------------                                    
PSGalleryModule Install a PowerShell module from the PowerShell Gallery.                                C:\...\PSDepend\PSDepen...
Task            Support dependencies by handling simple tasks.                                          C:\...\PSDepend\PSDepen...
Noop            Display parameters that a depends script would receive. Use for testing and validation. C:\...\PSDepend\PSDepen...
FileDownload    Download a file                                                                         C:\...\PSDepend\PSDepen...
Module          Install a PowerShell module from the PowerShell Gallery.                                C:\...\PSDepend\PSDepen...
```

Now that we know what types are available, we can read the comment-based help.  Hopefully the author took their time to write this:

```PowerShell
Get-PSDependType -DependencyType PSGalleryModule -ShowHelp
```

```
...
DESCRIPTION
    Installs a module from a PowerShell repository like the PowerShell Gallery.
    
    Relevant Dependency metadata:
        Name: The name for this module
        Version: Used to identify existing installs meeting this criteria, and as RequiredVersion for installation.  Defaults to 'latest'
        Target: Used as 'Scope' for Install-Module.  If this is a path, we use Save-Module with this path.  Defaults to 'AllUsers'

PARAMETERS
...
    -Repository <String>
        PSRepository to download from.  Defaults to PSGallery
        
        Required?                    false
        Position?                    2
        Default value                PSGallery
        Accept pipeline input?       false
        Accept wildcard characters?  false
```

In this example, we see how PSModulePath treats the Name, Version, and Target in the psd1, and we see a Parameter specific to this DependencyType, 'Repository'
