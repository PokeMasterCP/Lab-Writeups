## Lab Info

| Lab | Platform | Difficulty | Focus |
| --- | --- | --- | --- |
| Operation Blackout 2025: Phantom Check | Hack The Box | Very Easy | Windows event log analysis and virtualization detection |

## Scenario

Talion suspects that the threat actor carried out anti-virtualization checks to avoid detection in sandboxed environments. Your task is to analyze the event logs and identify the specific techniques used for virtualization detection. Byte Doctor requires evidence of the registry checks or processes the attacker executed to perform these checks.

Tools used:

- Hayabusa
- Elastic
- CyberChef

## Investigation Actions Taken

I started by creating JSON files from the provided `.evtx` files using Hayabusa:

```bash
./hayabusa -f Windows-Powershell-Operational.evtx -t jsonl -o operational.json -w
./hayabusa -f Microsoft-Windows-Operational.evtx -t jsonl -o operational.json -w
```

I loaded the JSON files into Elastic to query the logs. The statistics showed that all logs belonged to `DESKTOP-M3AKJSD`.

1. **Which WMI class did the attacker use to retrieve model and manufacturer information for virtualization detection?**

    I searched for "model" and confirmed that the threat actor used the `Win32_ComputerSystem` class.

    Query: `"model"`

    Payload:

    ```text
    "$Model = Get-WmiObject -Class Win32_ComputerSystem | select-object -expandproperty ""Model"""
    ```

2. **Which WMI query did the attacker execute to retrieve the current temperature value of the machine?**

    I searched for invocations of `Get-WmiObject` and confirmed that the threat actor used `MSAcpi_ThermalZoneTemperature` to get the current temperature.

    Query: `Details.ScriptBlock:"Get-WmiObject*"`

    Payload:

    ```text
    "Get-WmiObject -Query ""SELECT * FROM MSAcpi_ThermalZoneTemperature"" -ErrorAction SilentlyContinue"
    ```

3. **The attacker loaded a PowerShell script to detect virtualization. What is the function name of the script?**

    I searched for PowerShell and found:

    Query: `"ps*"`

    Payload:

    ```powershell
    function Check-VM {
    ```

4. **Which registry key did the above script query to retrieve service details for virtualization detection?**

    I found this snippet querying the registry for signs of Hyper-V services:

    ```powershell
    $hyperv = Get-ChildItem HKLM:\SYSTEM\ControlSet001\Services
    if (($hyperv -match "vmicheartbeat") -or ($hyperv -match "vmicvss") -or ($hyperv -match "vmicshutdown") -or ($hyperv -match "vmiexchange")) {
        hypervm = $true
     }
    }
    ```

5. **The VM detection script can also identify VirtualBox. Which processes is it comparing to determine if the system is running VirtualBox?**

    Further into the script, I found the VirtualBox detection section:

    ```powershell
    #Virtual Box

     $vb = Get-Process
     if (($vb -eq "vboxservice.exe") -or ($vb -match "vboxtray.exe"))
     {

     $vbvm = $true

     }
    ```

6. **The VM detection script prints any detection with the prefix 'This is a'. Which two virtualization platforms did the script detect?**

    I used a wildcard query to search for the script's "This is a" output.

    Query: `"This is a*"`

    Payload:

    ```text
    CommandInvocation(Out-Default): "Out-Default"ParameterBinding(Out-Default): name="InputObject"; value="This is a Hyper-V machine."ParameterBinding(Out-Default): name="InputObject"; value="This is a VMWare machine."
    ```
