flowchart TD
  Start[Landing Page]
  Start --> Decision1{Existing user}
  Decision1 -->|No| SignUp[Sign Up Form]
  Decision1 -->|Yes| Sign In Form
  SignUp --> VerifyEmail[Email Verification]
  VerifyEmail --> Dashboard[Dashboard]
  Sign In Form --> ValidateCreds{Credentials valid}
  ValidateCreds -->|Yes| Dashboard
  ValidateCreds -->|No| SignInError[Error Message]
  SignInError --> Sign In Form
  Dashboard --> Search[Search Companies]
  Search --> Filter[Filter Panel]
  Filter -->|Apply| Results[Results Grid]
  Results --> Profile[Company Profile]
  Profile --> Export{Export or push CRM}
  Export -->|Export file| Download[Download File]
  Export -->|Push CRM| PushCRM[Push to CRM]
  Dashboard --> Analytics[Dashboard Analytics]
  Dashboard --> Settings[Settings]
  Settings --> ProfileSettings[Profile Settings]
  Settings --> Security[Security Settings]
  Settings --> Billing[Billing Settings]
  Dashboard --> AdminCheck{Admin user}
  AdminCheck -->|Yes| AdminPanel[Admin Panel]
  AdminPanel --> UserMgmt[User Management]
  AdminPanel --> AuditLogs[Audit Logs]