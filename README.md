
## WinRM 사용 정리 

### WinRM 초기 설정 
- PowerShell (Run as Administrator) 
```
# Service Enable 
winrm qc 

# 인증 및 AllowUnencrypted 설정 
winrm set winrm/config/service/Auth '@{Basic="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
winrm set winrm/config/winrs '@{MaxMemoryPerShellMB="1024"}'
```
- 참고 : https://github.com/xebialabs/overthere/#winrm-winrm_internal-and-winrm_native

#### 네트워크 연결 형식 에러 
- 아래아 같은 에러의 경우 강제로 네트워크 연결 형식을 변경 필요 

```
# 초기화중 네트워크 연결 형식 오류 
>> winrm qc
WinRM 서비스가 이미 이 컴퓨터에서 실행 중입니다.
WSManFault
    Message
        ProviderFault
            WSManFault
                Message = 이 컴퓨터의 네트워크 연결 형식 중 하나가 공용으로 설정되어 WinRM 방화벽 예외가 작동하지 않습니다. 네트워크 연결 형식을 도메인 또는 개인으로 변경한 후 다시 시도하십시오.

오류 번호:  -2144108183 0x80338169
이 컴퓨터의 네트워크 연결 형식 중 하나가 공용으로 설정되어 WinRM 방화벽 예외가 작동하지 않습니다. 네트워크 연결 형식을  도메인 또는 개인으로 변경한 후 다시 시도하십시오.

```
- 네트워크 연결 타입을 변경으로 해결 
```
Set-NetConnectionProfile -NetworkCategory Private
```
- 참고 : https://4sysops.com/archives/enabling-powershell-remoting-fails-due-to-public-network-connection-type/?fbclid=IwAR0A2fUBF4G6Plv92mxcAIbpRwuAHflo-uLX1KIFYnfdb7GK4u-xu7-_6os

### Winrs 를 통한 원격 명령 
- Winrs 사용하려면 아래와 같이 TrustedHosts 를 추가 하여야함 
    - 서버, 클라이언트 모두 작업 필요
    - 모든 Hosts 허용 : `Set-Item wsman:\localhost\Client\TrustedHosts -value * `

```
# TrustedHosts 목록 보기 
Get-Item wsman:\localhost\Client\TrustedHosts

# TrustedHosts에 모든 컴퓨터를 추가 
Set-Item wsman:\localhost\Client\TrustedHosts -value * 

# TrustedHosts에 특정 도메인/IP 컴퓨터를 추가 
Set-Item wsman:\localhost\Client\TrustedHosts test.domain.com 
Set-Item wsman:\localhost\Client\TrustedHosts *.domain.com 
Set-Item wsman:\localhost\Client\TrustedHosts -value 1.1.1.1 

# 기존 TrustedHosts에 추가 
$curValue = (Get-Item wsman:\localhost\Client\TrustedHosts).value 
    Set-Item wsman:\localhost\Client\TrustedHosts -value "$curValue, test.domain.com" 

```
- 참고 : http://reid.tistory.com/29

#### Winrs 명령 예제
```
>> winrs -?

예:                                                                                          
winrs -r:https://myserver.com command
winrs -r:myserver.com -usessl command
winrs -r:myserver command
winrs -r:http://127.0.0.1 command
winrs -r:http://169.51.2.101:80 -unencrypted command
winrs -r:https://[::FFFF:129.144.52.38] command
winrs -r:http://[1080:0:0:0:8:800:200C:417A]:80 command
winrs -r:https://myserver.com -t:600 -u:administrator -p:$%fgh7 ipconfig
winrs -r:myserver -env:PATH=^%PATH^%;c:\tools -env:TEMP=d:\temp config.cmd
winrs -r:myserver netdom join myserver /domain:testdomain /userd:johns /passwordd:$%fgh789
winrs -r:myserver -ad -u:administrator -p:$%fgh7 dir \\anotherserver\share                    
```

- AD 설정이 없더라고 같은 계정/패스워스 인증으로 아래와 같이 가능 
```
winrs -r:ServerHost ipconfig all 

# 계정지정
winrs -r:ServerHost -u:admin_user -p:password ipconfig all 
```

- Administrators 계정이 아닌 경우 사용 
```
# 계정 설정 창이 실행되어 계정 추가 
winrm configSDDL default
```
- 참고 : https://serverfault.com/questions/590515/how-to-allow-access-to-winrs-for-non-admin-user

---

### PowerShell 원격 명령 및 파일 복사 
- Enable-PSRemoting  
This command starts the WinRM service, sets it to start automatically with your system, and creates a firewall rule that allows incoming connections. The -Force part of the cmdlet tells PowerShell to perform these actions without prompting you for each step.

```
Enable-PSRemoting -Force
```


- 원격 명령 
```
Invoke-Command -ComputerName ServerHost -ScriptBlock { dir d:\ }
```

- 파일 복사 
```
# 로컬 → 원격 
Copy-Item -Path D:\util\WinDump.exe -Destination d:\ -ToSession (New-PSSession -ComputerName ServerHost)
# 원격 → 로컬
Copy-Item -Path D:\serversocket.exe -Destination d:\ -FromSession (New-PSSession -ComputerName ServerHost)

```
- 참고 : https://www.howtogeek.com/117192/how-to-run-powershell-commands-on-remote-computers/
- 참고 : https://blog.ipswitch.com/use-powershell-copy-item-cmdlet-transfer-files-winrm

---

### Python Library 
- https://github.com/diyan/pywinrm
- Linux, Mac OS X, Windows 지원 

```python
import winrm

s = winrm.Session('ServerHost', auth=('user', 'password'))
r = s.run_cmd('ipconfig', ['/all'])
>>> r.status_code
0
>>> r.std_out

Windows IP Configuration
...
...

```

- Endpoint URL 유추 
```
windows-host -> http://windows-host:5985/wsman
windows-host:1111 -> http://windows-host:1111/wsman
http://windows-host -> http://windows-host:5985/wsman
http://windows-host:1111 -> http://windows-host:1111/wsman
http://windows-host:1111/wsman -> http://windows-host:1111/wsman
```
