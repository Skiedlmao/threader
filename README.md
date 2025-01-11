**copy paste it into a admin powershell **

Function Find-SuspendedThreadsInServices {
    $hasSuspended = $false
    $runningServices = Get-WmiObject Win32_Service |
        Where-Object {
            $_.State -eq 'Running' -and
            $_.ProcessId -ne 0
        }
    foreach ($service in $runningServices) {
        $procId = $service.ProcessId
        try {
            $process = Get-Process -Id $procId -ErrorAction Stop
        } catch {
            continue
        }
        # Filter for threads in a Wait state with WaitReason = Suspended
        $suspendedThreads = $process.Threads | Where-Object {
            $_.ThreadState -eq [System.Diagnostics.ThreadState]::Wait -and
            $_.WaitReason  -eq [System.Diagnostics.ThreadWaitReason]::Suspended
        }
        if ($suspendedThreads.Count -gt 0) {
            $hasSuspended = $true
            Write-Host "Service '$($service.Name)' (PID $procId) has suspended thread(s):" -ForegroundColor Yellow

            foreach ($thread in $suspendedThreads) {
                Write-Host "  - Thread ID: $($thread.Id) | State: $($thread.ThreadState) | WaitReason: $($thread.WaitReason)"
            }
            Write-Host
        }
    }
    if (-not $hasSuspended) {
        Write-Host "No suspended threads detected in any running services." -ForegroundColor Cyan
    }
}
Find-SuspendedThreadsInServices
