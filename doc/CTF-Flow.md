# Northenbridge College CTF — Challenge Flow

## Overview

The Northenbridge College CTF is structured as a progressive investigation through the fictional college portal. The player begins as a normal student and gradually discovers weaknesses in the portal.

The intended progression is:

```
College Website
      │
      ▼
Student Registration
      │
      ▼
Student Login
      │
      ▼
Student Profile and marks
      │
      ▼
Directory Enumeration
      │
      ▼
Discover .env file
      │
      ▼
Discover hidden /admin Portal
      │
      ▼
Login as admin 
      │
      ▼
Admin Dashboard
      │
      ▼
Limited Student Records
      │
      ▼
Unsafe Search
      │
      ▼
SQL injection to break restriction
      │
      ▼
Player's Student Record
      │
      ▼
Dump full database using SQLi
      │
      ▼
Follow compliance notes clue to find final flag
```

The challenge is designed to teach one concept at a time.  
</br>

## Player Starting Point

1. The player starts at the public Northenbridge College homepage.

2. The website presents itself as a normal fictional college portal.

3. The primary student functionality includes:

    - Student registration
    - Student login
    - Student profile
    - Examination marks

The administration portal is deliberately absent from the normal student navigation.  
</br>  

## Stage 1 — Discover the Administration Portal

**Objective**

Discover the hidden administration area.

**Starting point**

- The player begins with the normal student portal.
- Then player registers a student account and receives their generated credentials.
- After logging in, the player can access their profile and examination results.
- The newly registered student's seeded marks include a failing Python result.
- The marks page provides a Reassessment option but displays error.
- On Directory enumeration, player can discover `.env` file and hidden `/admin` portal   
</br>  

**Discovery**

The player visits admin portal and discovers first flag.

> **Flag 01**
> The first flag is displayed on the administration login page.


**Lesson**

An unlinked or hidden application route is not a security control.  
A route that is not visible in normal navigation can still be discovered.  
</br>  

## Stage 2 — Utilize the Exposed Administrator Credentials

**Objective**

- Use the fictional administrator credentials discover under `.env` file to continue.
- After reaching the administrator portal, the player needs administrator credentials.
- Find Valid one credential pair under `.env` file.
- The file includes information that should not be treated as a real secret but is intentionally exposed for the CTF.

> **Flag 02**
> On successful login, player will discover second flag on admin dashboard


**Discovery**

- The player discovers the file and observes that it contains the fictional administrator login information.
- The player can then return to the administrator login page and authenticate.

**Lesson**

A `.env` file is not an access-control mechanism and must not be publicly accessible.  
Sensitive credentials and configuration secrets must be protected from direct web access and stored securely outside the publicly served web directory whenever possible.  
</br>  

## Stage 3 — Explore the Limited Student Records

**Objective**

- Understand the normal limitations of the administrator's student-record view.
- After successful administrator authentication, the player reaches the administration dashboard.
- The dashboard provides a limited overview of student information and an option to edit student marks.
- The marks management page provides a student search interface.
- Under normal use, the search results are restricted to a subset of fictional student records.
- The player can inspect the available records and observe that not every student record is visible through the normal search behavior.

**Intended observation**

The important point at this stage is not immediately exploiting the application.

The player should first understand:

- What records are normally visible
- What information is displayed
- How student searches behave
- That the record set is intentionally limited

> **Flag 03** 
> The third flag is associated with the limited-record exploration stage.


**Lesson**

Application-level restrictions must actually enforce the intended data boundary.

A user interface showing only a subset of records does not necessarily mean the underlying query is securely restricted.  
</br>  

## Stage 4 — Investigate the Unsafe Search

**Objective**

- Identify the weakness in the administrator student-search functionality and use it to locate the player's student record.
- The administrator marks page contains a search field.
- The normal search behavior restricts displayed records to student IDs belonging to the seeded administrator-visible group.
- The implementation intentionally constructs part of this search query using the supplied search input without proper parameterization.

- This creates the SQL injection learning stage.

**Intended progression**

The player should:

- Observe the normal search behavior.
- Identify that the search input affects the database query.
- Test how the search behaves with unexpected input.
- Determine that the intended record restriction can be bypassed.
- Locate the student account created during registration.
- Obtain the player's generated student ID.
- Open the corresponding marks record.
- Modify the player's marks.

The vulnerability is confined to the fictional SQLite database used by the CTF.  
</br>  

## Player Record Discovery

A newly registered student receives an ID in the following general format:

```
NC-<initials><year>-<random-suffix>
```
For example:
```
NC-AB26-1234
```

The exact value depends on the registration information and generated suffix.

The student's initial marks include a deliberately low Python score. Therefore, once the player's record is discovered through the vulnerable administrator search, the player can select the record for editing.  
</br>  

## Search field vulnerability

The administrator search bar is vulnerable to SQLi which can break the student record view restriction and dump entire database.

The goal is not to dump oOS files of the VM or obtain operating-system access. The intended outcome is simply to demonstrate that the application-level weakness allows the player to access or alter data through webapp directly into database.  
</br>  

## Final Stage 

Final flag isn't stored on database, but instead it is present inside the system files. Player can find clue in dumped database table to locate flag.txt and use Local File Inclusion vulnerability to retrieve the file.

> **Flag 04**
> Final flag will be visible once LFI vulnerability is exploited.

**Lesson**

Sensitive information and challenge secrets may exist outside the database, within the server's filesystem. Information disclosed through database queries can help attackers discover the location of sensitive files, while an improperly implemented Local File Inclusion (LFI) vulnerability can allow unauthorized access to those files.  

The remediation is to validate and restrict file paths using allowlists, prevent user-controlled input from determining arbitrary filesystem paths, enforce least-privilege filesystem permissions, and keep sensitive files outside web-accessible directories.
 
</br>  
## Complete Challenge Chain

```
┌──────────────────────────────────────┐
│ 1. Visit Northenbridge College       │
│    Website                           │
└──────────────────┬───────────────────┘
                   ▼                    
┌──────────────────────────────────────┐
│ 2. Register Student Account          │
│    Receive generated credentials     │
└──────────────────┬───────────────────┘
                   ▼                    
┌──────────────────────────────────────┐
│ 3. Login and Open Profile            │
│                                      │
└──────────────────┬───────────────────┘
                   ▼                    
┌──────────────────────────────────────┐
│ 4. Perform Directory Brute-force     │
│    Discover /admin and .env file     │
└──────────────────┬───────────────────┘
                   ▼                    
┌──────────────────────────────────────┐
│ 5. Visit /admin web directory        │
│    Discover Admin Login Portal       │
└──────────────────┬───────────────────┘
                   │                    
              [ FLAG 1 ]                
                   │                    
                   ▼                    
┌──────────────────────────────────────┐
│ 5. Use credentials found inside .env │
│    to login as admin                 │
└──────────────────┬───────────────────┘
                   ▼                    
┌──────────────────────────────────────┐
│ 6. Obtain Fictional Admin Login and  │
│    Authenticate                      │
└──────────────────┬───────────────────┘
                   ▼                    
┌──────────────────────────────────────┐
│ 7. Explore Admin Dashboard           │
│    Inspect limited records           │
└──────────────────┬───────────────────┘
                   │                    
               [ FLAG 2 ]               
                   │                    
                   ▼                    
┌──────────────────────────────────────┐
│ 8. Inspect limited records           │
└──────────────────┬───────────────────┘               
                   │                    
                   ▼                    
┌──────────────────────────────────────┐
│ 9. Investigate Student Search        │
│    Identify unsafe query handling    │
└──────────────────┬───────────────────┘
                   ▼                    
┌──────────────────────────────────────┐
│ 10.Locate Player's NC-* Record       │
│     Through intended SQLi stage      |
|     and edit marks.                  │
└──────────────────┬───────────────────┘
                   │                    
              [ FLAG 3 ] 
                   ▼                             
┌──────────────────────────────────────┐
│ 11. Dump database and follow clue    │
│     present inside compliance notes  │
└──────────────────┬───────────────────┘
                   ▼                    
                                        
┌──────────────────────────────────────┐
│ 12. exploit LFI on ?id= parameter    │
│     and retrieve flag.txt from system│
│     files.                           │           
└──────────────────┬───────────────────┘
                   │                    
             [ FLAG 4 ]                 
                   │                    
                   ▼                    
             CTF COMPLETED              
```
</br>  

## 11. Flag Progression
Flag | Stage | Discovery
-|-|-
Flag 1 | Hidden administration route | Admin login portal
Flag 2 | After successful admin login | Admin Dasboard
Flag 3 | Limited records | Exploration of administrator record view
Flag 4 | SQL injection / Local File Inclusion |	Player locates file and retrieves it
</br>  

## Intended Learning Outcomes

By completing the challenge, a beginner should understand:

**Hidden routes**

- A URL being absent from the navigation does not make it inaccessible.

**Sensitive information exposure**

- Configuration Files on  web server can expose information if administrators place secrets inside them.

**Authorization and data boundaries**

- A restricted-looking interface is not sufficient protection if the backend query does not correctly enforce the restriction.

**SQL injection**

- Building SQL statements by directly concatenating user-controlled input can allow the application's intended query logic to be altered.

**Local File Inclusion**
- Improperly implemented Local File Inclusion (LFI) vulnerability can allow unauthorized access to those files.

**Secure remediation**

Students should be able to identify appropriate defensive measures, including:

- Parameterized SQL queries
- Proper authorization checks
- Avoiding secrets in web-accessible files
- Restricting sensitive routes
- Validating access to individual student records
- Separating user-facing functionality from administrative functionality  
</br>  

## Intended Scope

The challenge is deliberately limited to the fictional application and its seeded SQLite data.

The intended solution does not require:

- Host operating-system exploitation
- Vagrant exploitation
- VirtualBox exploitation
- SSH attacks
- Access to the host computer
- Internet-facing targets
- Real credentials
- Real student information
- Persistence
- Lateral movement

All data used by the challenge is fictional and intended only for the lab.  
</br>  
## Expected Completion

A player has completed the CTF when they have:

1. Discovered the hidden administration portal.
    - Obtained the first flag.

2. Discovered the intentionally exposed fictional administrator credentials.
    - Obtained the second flag.

3. Explored the limited administrator records.
4. Identified the unsafe student-search behavior.
5. Located their own registered student record.
      - Obtained the third flag.

6. Dump database and find clue for final flag
7. Follow clue and retrieve system file
    - Obtained the final flag.

The challenge is complete once all four flags have been recovered and the player can explain the basic security issue demonstrated by each stage.
