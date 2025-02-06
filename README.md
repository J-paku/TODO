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


경계선
-----------------------------------------------------------------------------------------------------------------------------------------

# Power BI 보고서 자동 다운로드 (CLI 기반)

Power BI 서비스에 **서비스 주체(Service Principal)**로 로그인하여 보고서 파일을 저장하는 자동화된 배치 파일을 만들기 위해 **Azure AD 권한 설정 및 PowerShell 코드 작성 방법**을 설명합니다.

---

## 1. **Azure AD에서 필요한 권한 설정**
Power BI REST API를 사용하려면 **Azure Active Directory(AAD)에서 서비스 주체를 생성하고, Power BI에 권한을 부여해야 합니다.**

### (1) **Azure AD에서 앱 등록**
1. [Azure Portal](https://portal.azure.com)에서 **Azure Active Directory**로 이동합니다.
2. **앱 등록** 메뉴에서 **새로운 애플리케이션 등록**을 클릭합니다.
3. **앱 이름 입력** → **지원하는 계정 유형 선택** (조직 전용 계정) → **리디렉션 URI 없음으로 설정** → **등록** 클릭
4. **앱 등록이 완료되면 다음 정보를 복사하여 저장합니다.**
   - **테넌트 ID (Tenant ID)**
   - **클라이언트 ID (Application ID)**
   - **클라이언트 암호 (Client Secret)**
     - `인증서 및 비밀키` → `새 클라이언트 암호 추가` → 생성된 `값(Value)`을 복사하여 저장

---

### (2) **API 권한 추가**
Power BI REST API를 사용하려면 아래의 **애플리케이션 권한(Application permissions)**을 추가해야 합니다.

1. **Azure AD > 앱 등록 > API 권한 > 권한 추가** 클릭
2. **Microsoft API > Power BI Service** 선택
3. **애플리케이션 권한(Application permissions)**에서 다음 권한 추가:
   - `Dataset.ReadWrite.All` → **모든 데이터셋 읽기 및 수정**
   - `Report.Read.All` → **모든 보고서 읽기**
   - `Workspace.ReadWrite.All` → **모든 작업 영역 읽기 및 수정**
   - `Capacity.Read.All` → **모든 용량 읽기**
   - `Dashboard.Read.All` → **모든 대시보드 읽기**
   - `Gateway.Read.All` → **모든 게이트웨이 읽기**
   - `Group.ReadWrite.All` → **모든 그룹(워크스페이스) 읽기 및 수정**

4. **관리자 동의 부여(Admin consent)**
   - `API 권한` 페이지에서 `관리자 동의 부여(Admin consent for <테넌트 이름>)` 버튼 클릭

---

### (3) **Power BI 관리자 포털에서 서비스 주체 활성화**
Power BI 서비스에서 서비스 주체(Service Principal)를 활성화해야 합니다.

1. [Power BI 관리 포털](https://app.powerbi.com/admin-portal) 이동
2. **테넌트 설정** > **서비스 주체(서비스 계정) 사용 허용** 활성화
3. **승인된 보안 그룹**에 해당 애플리케이션 추가

---

## 2. **PowerShell 스크립트 작성**
Power BI에 **로그인 후 보고서 파일을 다운로드**하는 PowerShell 스크립트입니다.

```powershell
# Power BI 서비스 계정 로그인 (서비스 주체 방식)
$tenantId = "YOUR_TENANT_ID"
$clientId = "YOUR_CLIENT_ID"
$clientSecret = "YOUR_CLIENT_SECRET"

# 보안 문자열 생성
$securePassword = ConvertTo-SecureString $clientSecret -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ($clientId, $securePassword)

# Power BI 로그인
Connect-PowerBIServiceAccount -ServicePrincipal -TenantId $tenantId -ClientId $clientId -Credential $credential

# 다운로드할 Power BI 보고서 정보
$workspaceId = "YOUR_WORKSPACE_ID"
$reportId = "YOUR_REPORT_ID"
$outputFilePath = "C:\Reports\PowerBI_Report.pbix"

# Power BI REST API를 사용하여 보고서 다운로드
$headers = @{
    "Content-Type"  = "application/json"
    "Authorization" = "Bearer $(Get-PowerBIAccessToken)"
}

# API 요청 URL
$exportUrl = "https://api.powerbi.com/v1.0/myorg/groups/$workspaceId/reports/$reportId/Export"

# 보고서 다운로드
Invoke-RestMethod -Uri $exportUrl -Headers $headers -Method Get -OutFile $outputFilePath

Write-Host "Power BI 보고서 다운로드 완료: $outputFilePath"

경계선
-------------------------
Power BI 테넌트 설정에서 Service Principal 사용 허용 여부 확인
Power BI에서는 Service Principal을 사용하려면 관리자 설정을 변경해야 합니다.
Power BI 관리자(Admin Portal)에서 설정이 되어 있는지 확인하세요.

🔹 확인 방법 (관리자 계정 필요)

Power BI Admin Portal 접속 (링크 → 'Admin Portal'로 이동)
"Tenant Settings" → "Developer Settings" → "Allow service principals to use Power BI APIs" 확인
Enable(허용)로 설정되어 있는지 확인 후 Save changes
❗ 만약 꺼져 있다면, Power BI 관리자에게 요청하여 활성화해야 합니다.
서비스 프린시펄(Service Principal)은 기본적으로 Power BI 테넌트에서 막혀 있을 수 있습니다.
