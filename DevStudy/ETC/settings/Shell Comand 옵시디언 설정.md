
1. Shell Command 플러그인 설치
2. 현재 내 wsl.exe 경로 찾기 in PowerShell
	```terminal
	Get-Command wsl.exe | Select-Object -Expand Source
	```

