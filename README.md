Power BI PowerShell 로그인 방법

Power BI에 PowerShell 명령어만으로 로그인하는 방법은 Azure AD 인증을 사용하여 Power BI REST API에 로그인하는 방식으로 가능합니다. 주로 Connect-PowerBIServiceAccount 명령어를 사용하며, 경우에 따라 서비스 주체(Service Principal)나 인증서 기반 인증을 활용할 수도 있습니다.

1. PowerShell을 사용한 Power BI 로그인 방법

(1) Microsoft 계정(MSA) 또는 조직 계정(Azure AD)으로 로그인

기본적으로 Azure AD 계정을 사용하여 Power BI에 로그인하려면 다음 명령어를 실행합니다.

Install-Module -Name MicrosoftPowerBIMgmt
Import-Module MicrosoftPowerBIMgmt

# Power BI 서비스 계정에 로그인
Connect-PowerBIServiceAccount

실행하면 로그인 창이 팝업되어 자격 증명을 입력해야 합니다.

(2) 서비스 주체(Service Principal)를 사용한 로그인 (클라이언트 ID + 비밀키)

대화형 로그인 없이 PowerShell에서 인증하려면 **서비스 주체(앱 등록)**을 사용해야 합니다.

1. Azure Portal에서 앱 등록 (App Registration)

Azure Portal에서 Azure AD > 앱 등록에서 새 애플리케이션을 등록

Application (client) ID와 Tenant ID를 확보

클라이언트 비밀(Client Secret) 생성 후 저장

Power BI 관리 포털에서 앱을 Power BI 서비스에 관리자로 추가해야 함

2. PowerShell에서 로그인
```
$tenantId = "YOUR_TENANT_ID"
$clientId = "YOUR_CLIENT_ID"
$clientSecret = "YOUR_CLIENT_SECRET"

# 서비스 주체로 Power BI 로그인
Connect-PowerBIServiceAccount -ServicePrincipal -TenantId $tenantId -ClientId $clientId -Credential (New-Object System.Management.Automation.PSCredential ($clientId, (ConvertTo-SecureString $clientSecret -AsPlainText -Force)))
```
주의: 서비스 주체는 Power BI 관리 포털에서 적절한 권한을 설정해야 사용할 수 있습니다.

(3) Windows 인증 기반 로그인 (자동화 시)

Azure AD 계정이 Windows 인증을 지원할 경우, -Credential 옵션을 사용하여 PowerShell에서 로그인할 수도 있습니다.

$UserCredential = Get-Credential  # 팝업 창에서 사용자 입력 가능
Connect-PowerBIServiceAccount -Credential $UserCredential

2. 로그인 후 실행 가능한 작업

(1) 현재 로그인된 사용자 정보 확인

Get-PowerBIUser

(2) 현재 워크스페이스 목록 가져오기

Get-PowerBIWorkspace

(3) 특정 데이터셋 새로 고침

$datasetId = "YOUR_DATASET_ID"
Invoke-PowerBIRestMethod -Url "groups/{workspaceId}/datasets/$datasetId/refreshes" -Method Post

3. 로그인 자동화 가능 여부

대화형 로그인 없이 실행 가능: 서비스 주체(Service Principal)를 사용하면 암호 없이 로그인 가능

보안 문제 고려: 클라이언트 비밀 또는 인증서를 안전한 저장소에 보관해야 함

Windows 인증 가능: Azure AD와 Windows 인증이 연동된 경우 자동 로그인 가능

PowerShell 명령어만으로 Power BI 로그인은 가능하지만, 대화형 로그인 없이 자동화하려면 서비스 주체 인증을 활용하는 것이 일반적입니다.
