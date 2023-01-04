# PowerShell-DynamicParam
Working with the DynamicParam options in PowerShell

Working with Dynamic Paramters in PowerShell is an incredibly useful improvement.  If you are reading this, I am assuming you already know about basic paramters, but just in case, I will start there. This ability applies to scripts and functions.  But here, I am going to use it at the script level.

A basic funtion looks like this
```PowerShell
Param (
    [Parameter(Mandatory = $false,
        HelpMessage = "Do not udpate R4 with device",
        Position = 1)]
    [switch]$NoR4,
    
    [Parameter(Mandatory = $false,
        HelpMessage = "Update Via Fastboot",
        Position = 2)]
    [switch]$Fastboot
)
 ```
 But let's say I want to make some enhancements that are more complex then a ParameterSetName can do.  For example some if...else or even multiple parameters using a function.
 
 The first thing to know is that after the DynamicParam {} you have to either have a Begin, Process, or End statement. If you just end the block and just sdtart entering code, it will not work.
 
 Immediatly after my Param block I star the DynamicParam
 
 ```PowerShell
 DynamicParam {
 ```
 I then need to define the Parameter Dictionary. I do this here because I am adding multiple entries, and if I used this inside my function, each time I call the function it resets. I like using this as Script, I don't need to, but it just helps me keep things clear.
 
 ```PowerShell
 $script:paramDictionary = New-Object System.Management.Automation.RuntimeDefinedParameterDictionary
 ```
 
 I need to build the function that will create the Dynamic Parameters.  This will make more sense later.  BUt I will explain each part as I go
 I start with some basic Parameters
 
 ```PowerShell
     function AddParam {
        param (
            [Parameter(Mandatory = $True,
                Position = 1)]
            [String]$ParamName,
            [Parameter(Mandatory = $True,
                Position = 3)]
            [String]$HelpMessage,
            [Parameter(Mandatory = $false,
                Position = 4)]
            [array]$ValidateSet
        )
```
None of that should be anything new, basic Param stuff.
Now we need to start building the Dynamic Parameters
Unlike the Dicationary, this needs to be cleared and reset for each paramter.
These options should look familiar

```PowerShell
        #Create the Attribute Parameters
        $AttributeSet = New-Object System.Management.Automation.ParameterAttribute
        $AttributeSet.Mandatory = $True
        $AttributeSet.HelpMessage = $HelpMessage
        #Create a Collection for the Attributes
        $AttributeCollection = new-object System.Collections.ObjectModel.Collection[System.Attribute]
```

You can even add validation sets.  But not all of my paramters require a validate set, I have to see if one was passed.  Notice that the ValdateSet param is an array, not a string.

```PowerShel
       if ($ValidateSet) {
            #Create an object to hold the Validation Set Attributes
            $paramOptions = New-Object System.Management.Automation.ValidateSetAttribute -ArgumentList $ValidateSet
            #We will add these Options to that Attribute Collection create above.
            $AttributeCollection.Add($paramOptions)
        }
```

Now we can add the Attribute Set we created above to the collection

```PowerShell
         $AttributeCollection.Add($AttributeSet)
```

Last we define the paramter (remember this has to be created for each one, so you cannot create it just once). and add the Defined Parameter to the Dication
And the close out the function
```PowerShell
        $ParamDefined = New-Object System.Management.Automation.RuntimeDefinedParameter($ParamName, [string], $AttributeCollection)
        $Script:paramDictionary.Add($ParamName, $ParamDefined)
    }
```

Now we start calling that function to create the dynamic parameters.  You can see, I am just calling the function like I would with any other function.  Notice how the Validate Set is done.

```PowerShell
    if (-Not $NoR4) {
        AddParam -ParamName "DeviceOwner"  -HelpMessage "Owner of the Device The ticket number from Jira"  
    }

    if ($Fastboot) {
        if (-Not $NoR4) {
            AddParam -ParamName "SerialNo"  -HelpMessage "Serial Number, ending in 2"
        }
        AddParam -ParamName "DeviceProductName" -HelpMessage "Product Model, RallyBar, RallyBarMini, TapIP, TapScheduler, RoomMate" -ValidateSet "RallyBar", "RallyBarMini", "TapIP", "TapScheduler", "RoomMate"
        AddParam -ParamName "DeviceBuildTags" -HelpMessage "Build Tags, Test-key or release-key" -ValidateSet "test-keys", "release-keys"
        
    }
    ```
Finally, we return the Param Dictionary from the Dynamic Param, and close out the DynamicParam

```PowerShell
    Return $Script:paramDictionary
}
```
Now we need to pull the variables out of the dynamic parameter. Because of how we built them, they are not just identified with a $. So now create some new variables.

```PowerShell
Begin {
    $paramDictionary.Values| ForEach-Object {New-Variable -Name $_.Name -Value $_.Value}

}
```

Then we start our the script process.  I could do this better with begin, process and end, but I would have to re-write a lot for that.

```PowerShell
Process {
    Write-Host $DeviceOwner
    Write-Host $SerialNo
    Write-Host $DeviceProductName
    Write-Host $DeviceBuildTags
}
```
So Putting it all together you get this:
```PowerShell
DynamicParam {
    $script:paramDictionary = New-Object System.Management.Automation.RuntimeDefinedParameterDictionary

    function AddParam {
        param (
            [Parameter(Mandatory = $True,
                Position = 1)]
            [String]$ParamName,
            [Parameter(Mandatory = $True,
                Position = 3)]
            [String]$HelpMessage,
            [Parameter(Mandatory = $false,
                Position = 4)]
            [array]$ValidateSet
        )

        $AttributeSet = New-Object System.Management.Automation.ParameterAttribute
        $AttributeSet.Position = $Position
        $AttributeSet.Mandatory = $True
        $AttributeSet.HelpMessage = $HelpMessage
    
        $AttributeCollection = new-object System.Collections.ObjectModel.Collection[System.Attribute]
        
        if ($ValidateSet) {

            $paramOptions = New-Object System.Management.Automation.ValidateSetAttribute -ArgumentList $ValidateSet
            $AttributeCollection.Add($paramOptions)
        }
        $AttributeCollection.Add($AttributeSet)
        $ParamDefined = New-Object System.Management.Automation.RuntimeDefinedParameter($ParamName, [string], $AttributeCollection)
        $Script:paramDictionary.Add($ParamName, $ParamDefined)
    }
    
    if (-Not $NoR4) {
        AddParam -ParamName "DeviceOwner"  -HelpMessage "Owner of the Device The ticket number from Jira"  
    }

    if ($Fastboot) {
        if (-Not $NoR4) {
            AddParam -ParamName "SerialNo"  -HelpMessage "Serial Number, ending in 2"
        }
        AddParam -ParamName "DeviceProductName" -HelpMessage "Product Model, RallyBar, RallyBarMini, TapIP, TapScheduler, RoomMate" -ValidateSet "RallyBar", "RallyBarMini", "TapIP", "TapScheduler", "RoomMate"
        AddParam -ParamName "DeviceBuildTags" -HelpMessage "Build Tags, Test-key or release-key" -ValidateSet "test-keys", "release-keys"
        
    }
    Return $Script:paramDictionary
}

Begin {
    $paramDictionary.Values| ForEach-Object {New-Variable -Name $_.Name -Value $_.Value}

}

Process {
    Write-Host $DeviceOwner
    Write-Host $SerialNo
    Write-Host $DeviceProductName
    Write-Host $DeviceBuildTags
}
```
