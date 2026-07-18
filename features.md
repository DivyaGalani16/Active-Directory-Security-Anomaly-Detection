
### Feature Engineering

Raw Windows Security Event Log entries are transformed into structured features
organized across six functional categories, each targeting a distinct dimension of 
authentication behaviour relevant to attack detection.

**1. Event Identification**
- **EventID** — Retains the Windows Security Event identifier as a categorical numeric 
  feature (4624, 4625, 4648, 4672, 4768, 4769, 4776). Different attack techniques produce 
  characteristic EventID sequences, making this a foundational detection feature.
- **LogonType** — Records how authentication was initiated: Type 2 (interactive), 
  Type 3 (network access), Type 9 (NewCredentials, alternative credential context), and 
  Type 10 (RemoteInteractive/RDP). Credential theft tools such as Mimikatz characteristically 
  produce Type 9 sessions, while lateral movement via RDP manifests as repeated Type 10 
  events across multiple hosts.

**2. Authentication Outcome**
- **Is_Failure** — Binary flag (1 = failure, 0 = success) derived from EventID 4625. 
  Repeated failures from the same source are a well-established indicator of 
  credential-based attacks (MITRE ATT&CK T1110 — Brute Force), most informative when 
  analysed alongside Auth_Burst_Ratio.

**3. Privilege and Access Level**
- **Is_Special_Logon** — Set to 1 when EventID 4672 is recorded, indicating sensitive 
  Windows privileges assigned at logon (e.g., SeDebugPrivilege, SeTcbPrivilege, 
  SeImpersonatePrivilege, SeBackupPrivilege). Associated with MITRE ATT&CK T1548 
  (Abuse Elevation Control) and T1003 (OS Credential Dumping).
- **Is_Elevated** — Flags sessions carrying a full administrative access token rather 
  than a restricted standard-user token, targeting MITRE T1548.
- **Is_First_Time_Admin** — Engineered behavioural feature set to 1 when an account 
  exercises administrative privileges for the first time across the full event history, 
  encoding the state transition that characterises privilege escalation without requiring 
  labelled training data.

**4. Protocol and Credential Type**
- **Is_NTLM** — Flags use of the legacy NTLM protocol rather than Kerberos, the target 
  protocol for Pass-the-Hash attacks (MITRE T1550.002).
- **Is_Type9** — Flags LogonType 9 (NewCredentials) sessions, the precise mechanism used 
  by credential theft tools to inject a captured hash into a new session.
- **Process_NtLmSsp** — Confirms at the process level that NTLM Security Support Provider 
  handled the authentication request, offering a complementary NTLM signal alongside 
  Is_NTLM.

**5. Behavioural and Temporal**
- **Auth_Burst_Ratio** — Ratio of authentication events from a source within a short time 
  window relative to its historical baseline rate, exploiting the natural frequency limits 
  of human authentication versus automated tools.
- **Inter_Arrival_Time** — Time in seconds between consecutive authentication events from 
  the same source. Human users generate irregular gaps; scripted tools fire at near-zero 
  intervals.
- **Unique_Target_Count** — Number of distinct workstations/servers accessed by a source 
  account within the observation window, targeting lateral movement (MITRE TA0008).
- **Hour_Of_Day** — Hour extracted from the event timestamp (0–23), capturing deviations 
  from an account's established behavioural baseline.
- **Is_Weekend** — Binary flag (1 = Saturday/Sunday) supplementing Hour_Of_Day, 
  preventing low-volume weekend activity from being misclassified as anomalous.

**6. Session and Target Identity**
- **LogonID_Enc** — Label-encoded Windows Logon Session ID, where −1 denotes a session 
  identifier absent from historical records, targeting credential relay attacks 
  (MITRE T1550).
- **Target_Work_Enc** — Label-encoded identifier of the destination workstation/server, 
  where −1 represents a previously unseen target machine, complementing 
  Unique_Target_Count by encoding target novelty (MITRE TA0008).

 Note: Values of −1 in encoded categorical fields (LogonID_Enc, Target_Work_Enc) 
 represent entities absent from the training domain history. MITRE ATT&CK technique 
 identifiers are referenced throughout to situate each feature within the established 
 threat taxonomy.
