**Personnel → Final Settlement -- Backend Logic Analysis and Technical
Documentation**

PRL, Retirement (Pay Commission, Wage Commission) and Contractual
settlement: eligibility, calculations, approvals, status changes and
database interaction

# Final Settlement Workflow

## What happens, in simple words

-   **Four menu items, one page.** Personnel → Final Settlement has PRL,
    Retirement (Pay Commission, Wage Commission) and Contractual. All of
    them open the same page, FinalSettlements/Index, with a number:
    typeId 1 = PRL, 3 = Pay Commission, 4 = Wage Commission, 5 =
    Contractual (there is no typeId 2). The files are in
    CddlErpApp.RCL.Admin/Pages/HumanResource/FinalSettlements/. The
    Admin Personnel menu (\_Admin.cshtml line 200) shows all four; the
    Managing Director menu shows PRL, Pay Commission and Wage Commission
    only. No page has an \[Authorize(Roles)\] attribute, so only "logged
    in" applies; the buttons check roles.

-   **There is no "settlement case" table row at the start.** The list
    is calculated every time it is opened by
    EmployeesRepository.GetLeaveReport(..., isFinalSettlement = true).
    Opening the list writes nothing. A row is created in
    AdminFinalSettlements or AdminClearanceForms only when HR first
    saves something on a note or on the Clearance Form for that
    employee.

-   **Initiation = opening an employee's action menu** (Index.cshtml
    lines 183--218): Final Settlement (Details), PRL Note & IDN (PRL
    only), Mental Preparation (PRL, Pay, Wage) and Clearance Form (all
    four lists). Contractual employees only get Clearance Form.

-   **The settlement note shows days and months, not money.** Details
    calculates service duration, gratuity months, earned leave remaining
    and how many days of leave encashment are payable. The rupee amounts
    (gratuity, leave amount, provident fund, deductions) are entered and
    approved in the Accounts module (Gratuity Payments and PF Final
    Settlement).

-   **Approving the settlement does not close the employee.** The only
    employee field the settlement pages write is
    Employees.EmployeeDisableTypeId (from the Clearance Form drop-down).
    Setting IsDisabled, DateOfDisable and DateOfRetirement is a separate
    manual Disable action on the Employees list, available to Admin HR
    and Super Admin.

-   **History is kept per employee, not per settlement type.**
    AdminFinalSettlements and AdminClearanceForms are looked up by
    EmployeeId only (FirstOrDefault), with no type or date column. PRL
    dates stay on the Employees row (PRLStartDate, PRLEndDate).

-   **PRL differs from the others** because officers and staff do not
    leave at once: they retire from active service at 59 and take up to
    12 months of post-retirement leave (PRL) before separation. Workers
    and wage-to-pay-scale employees retire at 60 with no PRL.
    Contractual staff are only listed; this module has no age rule,
    notes or calculation for them.

  ---------------------------------------------------------------------------------
  **Event**        **Database interaction**                 **Result on screen**
  ---------------- ---------------------------------------- -----------------------
  **HR opens a     SELECT only: Employees, Leaves (Status ≥ One row per active
  list (typeId 1,  8), LeaveOpenings, EmployeeSalaries      employee (default) with
  3, 4 or 5)**     (last non-zero BasicPay), FiscalYears.   dates, age, service
                   Nothing is written.                      duration, current basic
                                                            and gratuity months.
                                                            Ordered by days left to
                                                            Mental Preparation.

  **HR sets the    UPSERT                                   Printable notice to the
  Department Head  AdminFinalSettlements.DepartmentHeadId   employee, about 90 days
  (Mental          (new row if none exists).                before PRL start or
  Preparation)**                                            retirement.

  **HR fills the   UPSERT AdminFinalSettlements:            PRL Note with 12-month
  PRL Note and IDN PRLNoteReferences, four signer employee  PRL entitlement table;
  (PRL)**          ids and their name/designation text      IDN (death /
                   (PRLNoteInitiator, DGM, GM, DirFin); IDN resignation) note on
                   part: DeathRef, DeathDate, DeathReason.  the same page. No
                                                            approval status.

  **Admin          UPSERT AdminFinalSettlements:            Note shows saved text
  Personnel saves  BankClearance, PRLBasicInfo,             and attachments. Fields
  the settlement   IsBeforeNoon and up to 7 attachments     are editable only while
  note (Details,   (files in                                no row exists or Status
  Status 0)**      wwwroot/files/final_settlements).        = 0.

  **Forward on     UPDATE AdminFinalSettlements.Status 0 →  Button changes to the
  Details**        1 (Admin Personnel), 1 → 2 (Admin HR +   next role's label,
                   CPF Section Head), 2 → 3 (General        finally "Approved". No
                   Manager Admin); appends the signer's     send back.
                   HTML signature. INSERT Notifications +   
                   UserNotifications.                       

  **HR fills the   INSERT or UPDATE AdminClearanceForms:    Printable clearance
  Clearance Form   RefNo, ClearanceDate, RetirementDate,    letter. Header editable
  header**         OfficeOrderNo, OfficeOrderDate.          only while Status = 0.
                   Drop-down UPDATE                         
                   Employees.EmployeeDisableTypeId.         

  **Clearance Form UPDATE AdminClearanceForms: Status 0 → 1 Every section shows
  is forwarded and (to all sections, Admin Personnel), each Forward / Approve; form
  approved**       section's \*Status / \*Remarks /         shows "Approved" at
                   \*Signatures (24 blocks), Status 1 → 2   Status 3.
                   (to OIC Admin & HR, Admin Personnel), 2  
                   → 3 (Admin HR + CPF Section Head).       
                   Notifications at each step.              

  **Admin HR /     UPDATE Employees: IsDisabled = true,     Employee turns red,
  Super Admin      DateOfDisable, DateOfRetirement,         drops from active lists
  disables the     ReasonForDisable.                        and (by date) from
  employee                                                  salary. Final
  (Employees                                                Settlement lists show
  list)**                                                   the employee only under
                                                            status All / Deactive.

  **Accounts pays  INSERT GratuityPayments (own approval,   Gratuity and PF notes,
  the employee**   Status 0 → 6) and PFFinalSettlements     vouchers and
                   (Status 0 → 5). Reads Employees and      statements.
                   EmployeeSalaries.                        
  ---------------------------------------------------------------------------------

# Diagram: Final Settlement Workflow

![Final settlement
workflow](./image1.png "Final settlement flow"){width="9.791666666666666in"
height="5.572916666666667in"}

# Differences Between PRL, Retirement and Contractual

Employee Type: 1 = Officer, 2 = Staff, 3 = Worker. Employment Type: 1 =
Permanent, 2 = Contractual, 3 = Muster Roll. These meanings come from
the code comments in EmployeesRepository (lines 377--383, 519--529, 582)
because the lookup tables are not seeded in the project. Every list
defaults to active employees (IsDisabled = false); the Status filter can
show All or Deactive.

  ----------------------------------------------------------------------------------------------------------------------------
  **Aspect**                         **PRL (typeId 1)**      **Retirement -- Pay   **Retirement -- Wage  **Contractual (typeId
                                                             Commission (typeId    Commission (typeId    5)**
                                                             3)**                  4)**                  
  ---------------------------------- ----------------------- --------------------- --------------------- ---------------------
  **Who is listed (Index.cshtml.cs                                                                       
  lines 25--42,                                                                                          
  EmployeesRepository.GetEmployees                                                                       
  line 189)**                                                                                            

  **Employment Type**                Permanent (1)           Permanent (1)         Permanent (1)         Contractual (2)

  **Employee Type**                  Officer and Staff (1,   Any type (no filter)  Worker (3)            Officer, Staff,
                                     2)                                                                  Worker (1, 2, 3)

  **Wages-to-pay flag**              WagesToPayScaleDate is  WagesToPayScaleDate   WagesToPayScaleDate   WagesToPayScaleDate
                                     empty                   is filled (worker     is empty              is empty
                                                             moved to pay scale)                         

  **When service ends**                                                                                  

  **Rule**                           Age 59 ends active      Age 60                Age 60                No age rule in this
                                     service; PRL follows                                                module

  **Dates used**                     PRLStartDate (stored,   DateOfRetirement      Same as Pay           None read by these
                                     else birth date + 59    (stored, else birth   Commission            pages
                                     years); PRLEndDate      date + 60 years − 1                         
                                     (stored). Active        day); DateOfDisable                         
                                     service ends the day    if earlier                                  
                                     before PRL start.                                                   

  **Mental Preparation date**        PRL start − 90 days     Earlier of            Same as Pay           Not calculated
                                                             DateOfDisable /       Commission            
                                                             DateOfRetirement − 90                       
                                                             days                                        

  **Pages offered in the row menu                                                                        
  (Index.cshtml lines 187--216)**                                                                        

  **Final Settlement (Details)**     Yes                     Yes                   Yes                   No

  **PRL Note & IDN**                 Yes                     No                    No                    No

  **Mental Preparation**             Yes (notice says retire Yes (notice says      Yes (notice says      No
                                     at 59)                  retire at 60)         retire at 60)         

  **Clearance Form**                 Yes                     Yes                   Yes                   Yes (only page)

  **What the settlement note                                                                             
  calculates**                                                                                           

  **Leave basis**                    Earned leave =          Wage-period leave ÷   Annual leave =        None
                                     remaining working days  18 plus pay-period    remaining working     
                                     ÷ 11 (HAPL ÷ 12)        leave ÷ 11            days ÷ 18, plus       
                                                             (WagesClosingFS       opening ALClosing     
                                                             carried in)                                 

  **Unenjoyed leave wording**        "Paid" (already paid at "Payable"             "Payable"             --
                                     PRL start)                                                          

  **Lump encashment line**           "Paid as per rules"     "Not entitled"        "Not entitled"        --

  **Leave encashment cap**           540 days (18 months)    540 days              540 days              --
                                     over the whole career                                               

  **Service and gratuity months**    Joining → PRL end (+1   Joining → last        Joining → last        Not calculated
                                     day), so PRL counts as  working day           working day           (Employment Type 2 is
                                     service                                                             skipped)

  **PRL entitlement**                12 months: full average None                  None                  None
                                     pay for remaining                                                   
                                     earned-leave days (up                                               
                                     to 365), the rest on                                                
                                     half average pay                                                    

  **Approval, status and record                                                                          
  changes**                                                                                              

  **Approval of note**               Status 0 → 3 (Personnel Same                  Same                  No note
                                     → HR OIC → GM Admin)                                                

  **Clearance Form**                 Status 0 → 3 plus 24    Same                  Same                  Same
                                     section approvals                                                   

  **Employee record changed by these EmployeeDisableTypeId   Same                  Same                  Same
  pages**                            only                                                                

  **Employee closed by**             Manual Disable          Manual Disable        Manual Disable        Manual Disable;
                                     (Employees list); PRL                                               contract dates are in
                                     dates entered on the                                                ContractRenewals but
                                     Employee form                                                       not read here

  **Money side**                     Gratuity payment        Gratuity and PF       Gratuity and PF       Gratuity Payments
                                     (PRLBasic used if                                                   list only offers
                                     present) and PF final                                               Employment Type 1; no
                                     settlement                                                          contractual gratuity
                                                                                                         found
  ----------------------------------------------------------------------------------------------------------------------------

## PRL: initiation, processing and maintenance

-   **Eligibility and initiation:** a Permanent Officer or Staff who is
    active and not on wages-to-pay appears in the PRL list. PRLStartDate
    is the stored value or birth date + 59 years (GetLeaveReport line
    104). Employees Create fills the retirement date as birth date + 59
    years − 1 day (Create.cshtml.cs line 173). HR watches PRL Pending
    (within 120 days of PRL start or end) and Reminders ("Waiting For
    PRL", "PRL n days", and "End of Service, Disable the employee" after
    PRLEndDate).

-   **Processing:** Mental Preparation notice (90 days ahead) → PRL Note
    (references, four signers, 12-month PRL table) → Details note with
    bank clearance and attachments → Clearance Form → Accounts payments.
    The employee keeps working status and salary through PRL: Salaries
    skip an employee only after PRLEndDate is before the month start
    (Salaries/Index.cshtml.cs line 188).

-   **PRL length (PRLNote.cshtml lines 243--300):** remaining earned
    leave = earned leave − (EL + RL + EDL + LE taken). PRL days = the
    smaller of that figure and 365, starting at birth date + 59 years.
    If the leave is less than 365 days, the balance of the 365 days is
    granted on half average pay. Retirement takes effect at birth date +
    60 years − 1 day, at noon or evening according to "Before Noon".

-   **Maintenance:** PRLStartDate and PRLEndDate are typed on the
    Employee Create / Edit forms; PRL data is not written back by the
    PRL Note. After PRLEndDate HR must disable the employee unless
    PermanentToContractualDate is set (Reminders.cshtml.cs line 192).

## Retirement -- Pay Commission and Wage Commission

-   **Pay Commission** is the list of employees who were workers and
    were moved to the pay scale (WagesToPayScaleDate filled). **Wage
    Commission** is the list of workers still on wages. Both retire at
    60: DateOfRetirement is the stored value or birth date + 60 years −
    1 day (GetLeaveReport line 108); the Retirement Pending page lists
    them within 120 days of that date.

-   **No PRL.** The note title is "last working day", lump encashment is
    "not entitled", and the unenjoyed earned (or annual) leave days are
    shown as payable.

-   **Pay Commission leave:** the note recalculates only the pay-scale
    period (from WagesToPayScaleDate): EL = PayClosing (days ÷ 11), then
    adds the earned leave carried from the wage period (WagesClosingFS =
    wage-period days ÷ 18 + ALClosing − ALTaken, rounded). **Wage
    Commission leave:** annual leave (AL) = working days ÷ 18 counted
    from the gazette date (default 1 Jan 2016, editable on the page),
    plus the opening ALClosing balance.

-   **End of service:** DateOfDisable, when set and earlier than
    DateOfRetirement, replaces DateOfRetirement as the last working day.
    "Before Noon" on the Details form makes the last day not count;
    evening counts it.

## Contractual: how it is handled

-   **Listing:** Employment Type 2 staff of any employee type, active by
    default. Columns shown are only File No, Name, Designation, Date of
    Birth, Age and Joining; the Mental Preparation, PRL, retirement,
    service duration, basic and gratuity columns are hidden
    (Index.cshtml lines 150--173).

-   **Process:** the only row action is Clearance Form (approval by 24
    sections, then OIC Admin & HR). GetLeaveReport skips the whole
    service / gratuity / mental-preparation block for Employment Type 2
    (line 118), so no settlement note, leave encashment or gratuity is
    produced.

-   **Contract data lives elsewhere:** ContractRenewals (FromDate,
    ToDate, salary and designation snapshots) is used for contractual
    Casual Leave periods and in the Reminders list (days left to
    contract end). These pages do not read it, and nothing in the module
    ends a contract or disables the employee automatically.

-   **Record changes:** same as the others:
    Employees.EmployeeDisableTypeId from the Clearance Form, and a
    manual Disable. Gratuity Payments (Accounts) only offers Employment
    Type 1 employees, so contractual employees have no gratuity payment
    path in the code reviewed.

# Settlement Calculations, Benefits and Deductions

## HR side: days and months (no money)

  -------------------------------------------------------------------------------
  **Item**             **Formula**                         **Where**
  -------------------- ----------------------------------- ----------------------
  **Last working day   PRL: PRLEndDate + 1 day if stored;  EmployeesRepository
  (RetirementDate)**   otherwise the earlier of            271--274
                       DateOfDisable and DateOfRetirement. 
                       Never later than the report "To     
                       Date".                              

  **Leave period end** PRL start − 1 day when To Date is   Lines 276--298
                       after PRL start; else               
                       DateOfDisable-based date (minus 1   
                       day if "Before Noon"); for workers  
                       the year end of the previous year   
                       when To Date is earlier.            

  **Total working      (end date − start date) + 1. Start  Lines 265--312, 358
  days**               = joining date; workers who joined  
                       before the gazette date start at    
                       the gazette date (default 1 Jan     
                       2016); wage-to-pay employees start  
                       at the earlier of the wages-to-pay  
                       date and the gazette date.          

  **Earned leave       (Total days − EL, RL, HAPL, AL, LE, Lines 508--529
  (EL)**               LWP taken) ÷ 11 for Officer /       
                       Staff; HAPL uses ÷ 12; workers AL ÷ 
                       18.                                 

  **Wage-to-pay        EL = wage-period days ÷ 18 +        Lines 532--555
  split**              pay-period days ÷ 11;               
                       WagesClosingFS = (wage part +       
                       ALClosing − ALTaken) rounded.       

  **Unenjoyed leave    Types 1 and 3: remaining =          Details.cshtml
  days**               WagesClosingFS + (EL − (EL + RL +   163--178
                       EDL + LE taken)). Type 4: remaining 
                       = ALClosing + (AL − (AL + LE        
                       taken)). Payable = remaining, or    
                       540 − leave already encashed if     
                       remaining + encashed ≥ 540 (never   
                       below 0).                           

  **Leave encashed so  Sum over all approved category-15   Details.cshtml
  far**                leaves of (days ÷ 2). Split in the  161--199
                       note into before / after the        
                       wages-to-pay date, or before /      
                       after 1 Jan 2016.                   

  **Service duration** Joining date to last working day +  GetLeaveReport
                       1 day (GetMonthAndDays adds one     123--128
                       more day). Shown in years, months,  
                       days.                               

  **Gratuity months    2 months for each completed year,   GetLeaveReport 125,
  (NoOfGratuity)**     plus 2 more months if the extra     NoOfGratuity()
                       period is more than 6 months (6     
                       months and at least 1 day counts).  
                       Example: 30 years 8 months = 62.    

  **Current basic**    Latest non-zero                     GetLeaveReport 79--97
                       EmployeeSalaries.BasicPay with      
                       Month.FiscalYearId up to the fiscal 
                       year of To Date (July--December:    
                       that year; January--June: previous  
                       year).                              

  **Mental Preparation PRL start (type 1) or earlier of    GetLeaveReport
  date**               DateOfDisable / DateOfRetirement    140--145
                       (types 3, 4) minus 90 days. Days    
                       left = max(0, date − To Date).      

  **PRL entitlement**  min(remaining earned leave, 365)    PRLNote.cshtml
                       days from birth date + 59 years on  243--300
                       full average pay; 365 − that on     
                       half average pay.                   
  -------------------------------------------------------------------------------

## Accounts side: money (separate processes)

  -------------------------------------------------------------------------
  **Component**    **Calculation**                   **Where**
  ---------------- --------------------------------- ----------------------
  **Gratuity**     Basic × NoOfGratuity. Basic =     Payments.cshtml.cs
                   PRLBasic when it is not 0, else   371--402
                   LastBasic. NoOfGratuity is        
                   suggested from the gratuity       
                   statement and can be edited.      

  **Leave amount** Basic ÷ 30 × NoOfLeave (days      Line 386
                   entered by Accounts).             

  **Provident      PFOwn + PFCompany entered on the  Line 388;
  fund**           gratuity note; the PF Final       PFFinalSettlement.cs
                   Settlement itself holds           
                   SelfContribution,                 
                   CompanyContribution,              
                   TotalPayable, LoanAdjustment,     
                   NetPayment.                       

  **Gross          Gratuity + leave amount + PFOwn + Line 391
  payment**        PFCompany.                        

  **Deductions**   HouseRent, ElectricBill, GasBill, Lines 393--401
                   WelfareLoan, NoticePay,           
                   OtherDeduction (named),           
                   RevenueStamp (0 if gross \< 400,  
                   else 10).                         

  **Net payment**  Gross payment − total deductions. Line 402

  **Gratuity       Disabled employee: PRLEndDate + 1 Statement.cshtml.cs
  provision date** if PRL exists, else               348--372
                   DateOfDisable + 1. Service months 
                   from joining to that date.        
  -------------------------------------------------------------------------

# Approval Workflows and Status Codes

  ------------------------------------------------------------------------------------------------------------
  **Record**                       **Status**   **Meaning and who    **Notes**
                                                moves it**           
  -------------------------------- ------------ -------------------- -----------------------------------------
  **Settlement note                                                  
  (AdminFinalSettlements.Status,                                     
  Details.cshtml.cs line 115)**                                      

  **Settlement note**              0            Draft with Admin     Admin Personnel can edit and save; button
                                                Personnel            "Forward to Section Head".

                                   1            With OIC Admin & HR  Button "Forward to Department Head".
                                                (needs both roles    
                                                Admin HR and CPF     
                                                Section Head)        

                                   2            With General Manager Button "Approve". GM Admin sets Status 3.
                                                Admin                

                                   3            Approved             Button shows "Approved". Signature column
                                                                     holds three HTML signatures split by
                                                                     \<@#\$/\>. No send back.

  **Clearance Form                                                   
  (AdminClearanceForms.Status,                                       
  ClearanceForm.cshtml.cs line                                       
  87)**                                                              

  **Clearance Form**               0            Header being         Admin Personnel: "Forward to All
                                                prepared             Sections".

                                   1            Sections reviewing   Sections can act only in this status.
                                                                     Admin Personnel: "Forward to OIC Admin &
                                                                     HR".

                                   2            With OIC             Admin HR + CPF Section Head: "Approve".

                                   3            Approved             Notification to Admin Personnel.

  **Each section (24 blocks)**     0 / 1 / 2    0 none, 1 forwarded  Remarks default to "নাই" (none). Legal,
                                                by a section user, 2 Berthing and Common sections also accept
                                                approved by the CPF  three named user ids as approvers
                                                Section Head of that (hard-coded).
                                                section              

  **Related Accounts approvals                                       
  (not HR)**                                                         

  **Gratuity payment               0 to 6       0 Gratuity Editor →  Send back at statuses 1 to 5
  (GratuityPayments.Status)**                   1 Pay Section OIC →  (Gratuities/FinalSettlement.cshtml.cs
                                                2 Gratuity           122--205).
                                                Accounts + Section   
                                                Head → 3 DGM Finance 
                                                (if used) → 4 GM     
                                                Finance → 5 Managing 
                                                Director → 6         
                                                approved             

  **PF final settlement            0 to 5       0 → 1 Pay Section    Send back returns to 0
  (PFFinalSettlements.Status)**                 Viewer → 2 DGM       (FinalSettlementNotes/Index.cshtml.cs).
                                                Finance (if used) →  
                                                3 GM Finance → 4 MD  
                                                → 5 approved         
  ------------------------------------------------------------------------------------------------------------

Notifications: each step creates one Notifications row and a
UserNotifications row for every user holding the next role (found by
looping over all users and calling IsInRole). Settlement note: Status 1
→ HR OIC, 2 → GM Admin, 3 → Admin Personnel. Clearance Form: 1 → role
"User In-Charge", 2 → HR OIC, 3 → Admin Personnel; section steps go to
CPF Section Heads of that section.

# Employee Status Changes and Historical Records

  --------------------------------------------------------------------------------------------------------
  **Change**                      **Where it is done**                  **Notes**
  ------------------------------- ------------------------------------- ----------------------------------
  **EmployeeDisableTypeId**       Clearance Form drop-down →            Only employee column written by a
                                  ClearanceForm.OnPostSaveDisableType   settlement page. No status or role
                                  (line 76)                             check on the server. Id 12 is
                                                                        treated as death (PRLNote.cshtml
                                                                        line 803); Ids 1 and 2 are left
                                                                        out of the generic wording on the
                                                                        form.

  **IsDisabled, DateOfDisable,    Employees list → Disable / Disable    The real "end of service" action.
  DateOfRetirement,               Many (Employees/Index.cshtml.cs       Salaries skip a disabled employee
  ReasonForDisable**              183--218). Button shown to Admin HR   whose DateOfDisable is before the
                                  and Super Admin.                      month start, and one with no
                                                                        DateOfDisable and no PRLEndDate
                                                                        immediately
                                                                        (Salaries/Index.cshtml.cs
                                                                        188--196).

  **PRLStartDate, PRLEndDate**    Employees Create / Edit forms         Typed by HR. Shown on Employee
                                                                        Details and used by salary, bonus,
                                                                        income tax, insurance and gratuity
                                                                        code.

  **PermanentToContractualDate,   Employees Create / Edit forms         After PRL end, a filled
  ContractualToPermanentDate**                                          PermanentToContractualDate stops
                                                                        the "End of Service, Disable the
                                                                        employee" reminder (re-engagement
                                                                        as contract staff).

  **New employee row after        Employees list → Recruit Existing     Clones the old Employees row (new
  retirement (RecruitExisting)**  (Index.cshtml.cs 220--276)            Id, IsDisabled = false,
                                                                        DateOfDisable cleared, new joining
                                                                        date, retirement = birth date + 59
                                                                        or 60 years − 1 day), copies
                                                                        attachments, sets
                                                                        old.ExtendedEmployeeId = new Id
                                                                        and re-points user accounts to the
                                                                        new row.

  **Settlement documents**        AdminFinalSettlements,                One row per employee, no type.
                                  AdminClearanceForms                   Attachments (7) stay in
                                                                        wwwroot/files/final_settlements.
                                                                        Signatures are stored as HTML
                                                                        text. Rows are never deleted by
                                                                        the pages.

  **Historical data kept**        Leaves, LeaveOpenings,                Nothing is archived or copied at
                                  EmployeeSalaries, GratuityPayments,   settlement time; every figure is
                                  PFFinalSettlements, ContractRenewals  rebuilt from live tables each time
                                                                        a page opens.
  --------------------------------------------------------------------------------------------------------

# Files and Methods

  ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  **File**                                                                            **Method**                      **What it does**      **Database table(s)**      **Read /
                                                                                                                                                                       Write**
  ----------------------------------------------------------------------------------- ------------------------------- --------------------- -------------------------- ---------
  **Pages in CddlErpApp.RCL.Admin/Pages/HumanResource/FinalSettlements/**                                                                                              

  **Index.cshtml(.cs)**                                                               OnGet (line 23)                 Maps typeId to        Employees, Leaves,         Read
                                                                                                                      Employment / Employee LeaveOpenings,             
                                                                                                                      Type and wages-to-pay EmployeeSalaries,          
                                                                                                                      filter; calls         FiscalYears                
                                                                                                                      GetLeaveReport in                                
                                                                                                                      final-settlement                                 
                                                                                                                      mode. View builds the                            
                                                                                                                      table and row menu.                              

  **Details.cshtml(.cs)**                                                             OnGet (38),                     Settlement note:      AdminFinalSettlements,     Read /
                                                                                      OnPostSaveFinalSettlement (89), general information   Employees,                 Write
                                                                                      OnPostForward (115)             and unenjoyed-leave   EmployeeDisableTypes,      
                                                                                                                      tables; save bank     Notifications              
                                                                                                                      clearance, PRL basic                             
                                                                                                                      info, Before Noon, 7                             
                                                                                                                      attachments; forward                             
                                                                                                                      0 → 3 with signatures                            
                                                                                                                      and notifications.                               

  **PRLNote.cshtml(.cs)**                                                             OnGet (27),                     PRL Note (12-month    AdminFinalSettlements,     Read /
                                                                                      OnPostSaveFinalSettlement (81), PRL table, signers,   Employees                  Write
                                                                                      OnPostSaveIDN (136)             references) and IDN                              
                                                                                                                      (death /                                         
                                                                                                                      resignation). Gazette                            
                                                                                                                      date default 1 Jul                               
                                                                                                                      2015.                                            

  **MentalPreparation.cshtml(.cs)**                                                   OnGet (26), OnPostSaveDeptHead  Advance notice about  AdminFinalSettlements,     Read /
                                                                                      (60)                            90 days before PRL    Users, UserRoles           Write
                                                                                                                      start or retirement;                             
                                                                                                                      Department Head                                  
                                                                                                                      chosen from active                               
                                                                                                                      users with role CPF                              
                                                                                                                      Department Head.                                 

  **ClearanceForm.cshtml(.cs) and Partial/\_ClearanceForm.cshtml**                    OnGet (33), SaveClearanceForm   Clearance letter with AdminClearanceForms,       Read /
                                                                                      (44), SaveDisableType (76),     24 section blocks;    Employees,                 Write
                                                                                      Forward (87), SaveSectionRemark header, disable type, EmployeeDisableTypes,      
                                                                                      (168), ApproveSection (186),    section remarks and   Notifications              
                                                                                      GetHistory (354)                approvals; remark                                
                                                                                                                      auto-complete from                               
                                                                                                                      earlier forms.                                   

  **ClearanceFormPending.cshtml(.cs)**                                                OnGet (24)                      List of forms at      AdminClearanceForms,       Read
                                                                                                                      Status ≥ 1 by         FiscalYears                
                                                                                                                      clearance date, for                              
                                                                                                                      sections (linked from                            
                                                                                                                      \_CommonAndSubStore                              
                                                                                                                      menu).                                           

  **Shared code**                                                                                                                                                      

  **CddlErpApp.Data/Repository/Common/EmployeesRepository.cs**                        GetLeaveReport (40),            Builds the list and   Employees, Leaves,         Read
                                                                                      GetEmployees (189), GetLeaves   the figures used by   LeaveOpenings,             
                                                                                      (255)                           every page above (see EmployeeSalaries,          
                                                                                                                      Calculations).        FiscalYears,               
                                                                                                                                            ContractRenewals           

  **CddlErpApp.Data/Repository/Common/EmployeesRepository.cs**                        GetClearanceForm (913),         Loads the clearance   AdminClearanceForms,       Read
                                                                                      GetMonthAndDays, NoOfGratuity   form, sections,       DepartmentWiseSections,    
                                                                                                                      signer map; service   Department                 
                                                                                                                      duration and gratuity                            
                                                                                                                      months.                                          

  **Related HR code**                                                                                                                                                  

  **CddlErpApp.RCL.Admin/Pages/HumanResource/Employees/Index.cshtml.cs**              OnPostDisable (183),            Closes the employee;  Employees,                 Write
                                                                                      OnPostDisableMany (198),        clones the employee   EmployeeWiseAttachments,   
                                                                                      OnPostRecruitExisting (220)     row when re-engaged.  EmployeeFiles, Users       

  **Employees/PRLPending.cshtml.cs, RetirementPending.cshtml.cs,                      OnGet                           Early-warning lists:  Employees,                 Read
  Reminders.cshtml.cs**                                                                                               PRL within 120 days,  ContractRenewals           
                                                                                                                      retirement (60)                                  
                                                                                                                      within 120 days,                                 
                                                                                                                      contract and PRL                                 
                                                                                                                      reminders.                                       

  **Accounts code that continues the settlement**                                                                                                                      

  **CddlErpApp.RCL.Accounts/Pages/GratuityAccounts/Gratuities/Payments.cshtml.cs**    OnPostSavePayment (134),        Gratuity payment rows GratuityPayments,          Read /
                                                                                      OnPostGetEmployeeGratuityInfo   and net calculation.  Employees,                 Write
                                                                                      (282),                          Blocked when the      EmployeeSalaries,          
                                                                                      OnPostCalculateNetPayment (371) fiscal year is        GratuityStatements         
                                                                                                                      finalized.                                       

  **.../Gratuities/FinalSettlement.cshtml.cs**                                        OnPostForward (122),            Gratuity approval 0 → GratuityPayments           Write
                                                                                      OnPostSendBack (167)            6.                                               

  **CddlErpApp.RCL.Accounts/Pages/PFAccounts/FinalSettlementNotes/Index.cshtml.cs**   OnPostForward (27),             PF final settlement   PFFinalSettlements         Write
                                                                                      OnPostSendBack (166)            note approval 0 → 5.                             
  ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Database Tables Used

  ----------------------------------------------------------------------------------------------------------------------------------
  **Table**                   **Model file (CddlErpApp.Model/\...)**                          **Used   **Purpose**
                                                                                              when**   
  --------------------------- --------------------------------------------------------------- -------- -----------------------------
  **AdminFinalSettlements**   Admin/HumanResource/FinalSettlements/AdminFinalSettlement.cs    Read /   Settlement note data for one
                                                                                              Write    employee: bank clearance, PRL
                                                                                                       basic text, Before Noon,
                                                                                                       death data, Department Head
                                                                                                       id, PRL Note signers and
                                                                                                       references, Status,
                                                                                                       signatures, 7 attachments.

  **AdminClearanceForms**     Admin/HumanResource/FinalSettlements/AdminClearanceForm.cs      Read /   Clearance letter header,
                                                                                              Write    Status, signatures and, for
                                                                                                       each of 24 sections, Remarks,
                                                                                                       Status and Signatures.

  **EmployeeDisableTypes**    Admin/HumanResource/Setup/EmployeeDisableType.cs                Read     Reason of service end (Name,
                                                                                                       NameBangla). Rows are not
                                                                                                       seeded in code.

  **Employees**               Admin/HumanResource/Employees/Employee.cs                       Read /   DateOfBirth, JoiningDate,
                                                                                              Write    DateOfRetirement,
                                                                                                       PRLStartDate, PRLEndDate,
                                                                                                       WagesToPayScaleDate,
                                                                                                       PermanentToContractualDate,
                                                                                                       IsDisabled, DateOfDisable,
                                                                                                       ReasonForDisable,
                                                                                                       EmployeeDisableTypeId,
                                                                                                       ExtendedEmployeeId,
                                                                                                       FileNumber, EmployeeTypeId,
                                                                                                       EmploymentTypeId.

  **Leaves, LeaveOpenings**   Common/CommonTasks/Leave, Other/Common/Leave                    Read     Approved leaves (Status ≥ 8)
                                                                                                       and opening balances behind
                                                                                                       every leave figure. See the
                                                                                                       Leave Entry document.

  **EmployeeSalaries,         Accounts/PaySection, Accounts/YearAndMonths                     Read     Last non-zero BasicPay used
  FiscalYears, Months**                                                                                as Current Basic.

  **ContractRenewals**        Admin/HumanResource/Employees/ContractRenewal.cs                Read     Contract periods (not read by
                                                                                                       Final Settlement pages).

  **Department,               Admin/HumanResource/Setup                                       Read     Section list and ordering for
  DepartmentWiseSections**                                                                             the Clearance Form (21
                                                                                                       sections plus three virtual
                                                                                                       Accounts sub-sections with
                                                                                                       Ids 51, 52, 53).

  **Notifications,            Other/Common/Notifications                                      Write    Alerts for each forward step.
  UserNotifications**                                                                                  

  **GratuityPayments**        Accounts/GratuityAccounts/GratuityPayments/GratuityPayment.cs   Read /   Gratuity amount, leave
                                                                                              Write    amount, PF shares,
                                                                                                       deductions, net payment,
                                                                                                       nominees, Status.

  **PFFinalSettlements**      Accounts/PFAccounts/PFFinalSettlement.cs                        Read /   Provident fund payout, loan
                                                                                              Write    adjustment, IDN number and
                                                                                                       date, Status.
  ----------------------------------------------------------------------------------------------------------------------------------

# Model Fields

## AdminFinalSettlement (AdminFinalSettlements)

  ----------------------------------------------------------------------------------
  **Field**               **Type**    **Notes**
  ----------------------- ----------- ----------------------------------------------
  **Id, EmployeeId /      int / nav   Primary key; required foreign key to Employees
  Employee**                          (restrict delete). Lookup is by EmployeeId
                                      only.

  **BankClearance,        string,     Text printed in the note; Before Noon decides
  PRLBasicInfo,           string,     whether the last day counts.
  IsBeforeNoon**          bool        

  **DeathRef, DeathDate,  string,     IDN section: reference, death or resignation
  DeathReason**           DateTime,   date, reason (death only).
                          string      

  **DepartmentHeadId**    int         Employee id chosen on Mental Preparation. 0 =
                                      none.

  **PRLNoteReferences**   string      One reference per line.

  **PRLNoteInitiator /    int?,       Chosen signers; signature = Bangla name +
  DGM / GM / DirFin       string      designation text.
  EmployeeId and                      
  Signature**                         

  **Status, Signatures**  int, string Note approval 0 to 3; signature HTML joined by
                                      \<@#\$/\>.

  **Attachment1--7        string      Uploaded file names: service/leave
  (File1--7, IsDelete1--7             information, section clearances, bank
  not mapped)**                       certificate, retirement order, joining letter,
                                      pay-commission promotion order, others.
  ----------------------------------------------------------------------------------

## AdminClearanceForm (AdminClearanceForms)

  -----------------------------------------------------------------------------------
  **Field**                 **Type**   **Notes**
  ------------------------- ---------- ----------------------------------------------
  **Id, EmployeeId /        int / nav  Required foreign key to Employees.
  Employee**                           

  **RefNo, ClearanceDate,   string,    Header of the clearance letter.
  RetirementDate,           DateTime   
  OfficeOrderNo,                       
  OfficeOrderDate**                    

  **Status, Signatures**    int,       Form approval 0 to 3.
                            string     

  **\<Section\>Remarks,     string,    24 blocks: Administration, HR, Berthing,
  \<Section\>Status,        int,       Security, Medical, Safety, MT, Legal, Common,
  \<Section\>Signatures**   string     Commerce, Planning, Project, Store,
                                       Fabrication, Engineering, Hull, Maintenance,
                                       Civil, Electrical, IT, Finance, WPPF, WF, Pay.
  -----------------------------------------------------------------------------------

# Model Relationship Fields

  ------------------------------------------------------------------------------------------------------------------------------------------
  **Model**                  **Link column (FK)**    **Link property**     **Related model**     **Type /     **Note**
                                                                                                 Required**   
  -------------------------- ----------------------- --------------------- --------------------- ------------ ------------------------------
  **AdminFinalSettlement**   EmployeeId              Employee              Employee              Many→1 /     Index on EmployeeId only, not
                                                                                                 Required     unique. Delete restricted.

  **AdminClearanceForm**     EmployeeId              Employee              Employee              Many→1 /     Index only, not unique. Delete
                                                                                                 Required     restricted.

  **AdminFinalSettlement**   DepartmentHeadId,       none                  Employee              Plain        Not enforced by the database.
                             PRLNote...EmployeeId                                                numbers      

  **Employee**               EmployeeDisableTypeId   EmployeeDisableType   EmployeeDisableType   Many→1 /     Set from the Clearance Form.
                                                                                                 Optional     

  **Employee**               ExtendedEmployeeId      none                  Employee (new row)    Plain number Points from the old row to the
                                                                                                              re-engaged row. 0 = none.

  **GratuityPayment**        EmployeeId              Employee              Employee              Many→1 /     Employee.GratuityPayments.
                                                                                                 Required     Delete restricted.

  **PFFinalSettlement**      EmployeeId,             Employee, FiscalYear, Employee, FiscalYear, Many→1 /     Employee.PFFinalSettlements.
                             FiscalYearId,           PFPaymentType         PFPaymentType         Required     
                             PFPaymentTypeId                                                                  

  **ContractRenewal**        EmployeeId              Employee              Employee              Many→1       Employee.ContractRenewals;
                                                                                                              loaded by GetEmployees.
  ------------------------------------------------------------------------------------------------------------------------------------------

# Dependencies on Other HR and Financial Processes

  -----------------------------------------------------------------------------
  **Process**           **How it depends on or is     **What changes if it is
                        used by Final Settlement**    wrong**
  --------------------- ----------------------------- -------------------------
  **Employee master     Birth date, joining date,     Wrong dates move the
  data**                type, PRL / retirement /      employee between lists
                        disable / wages-to-pay dates  and change leave and
                        decide who is listed, every   gratuity months.
                        date and every month count.   

  **Leave Entry         GetLeaves supplies EL, HAPL,  Wrong opening balances
  (Leaves, Leave        AL, taken counters, wages     change payable leave
  Opening)**            closing; approved leave of    days. See the Leave Entry
                        category 15 gives encashment  document.
                        already used.                 

  **Payroll (Salaries,  Current Basic = last non-zero An early Disable without
  EmployeeSalaries)**   BasicPay. Salaries stop after DateOfDisable stops
                        PRLEndDate or DateOfDisable.  salary at once.
                        Salary print has a PRL        
                        employee filter.              

  **Gratuity Accounts** Provision and payment use     HR and Accounts can
                        PRLEndDate / DateOfDisable    differ by one day for PRL
                        and the same NoOfGratuity     (see Things to Know).
                        rule; payment approved        
                        separately (0 to 6).          

  **PF Accounts**       PF final settlement (own +    No automatic link from
                        company share, loan           the HR note to the PF
                        adjustment) approved 0 to 5;  record.
                        the HR note only states "paid 
                        as per Trust rules".          

  **5% profit,          Use PRLStartDate / PRLEndDate PRL dates must be right
  incentive bonus,      / IsDisabled to decide        before those runs.
  income tax, group     eligibility or service end.   
  insurance**                                         

  **Reminders and       PRL Pending, Retirement       --
  pending lists**       Pending and Reminders drive   
                        when HR starts a settlement.  

  **Organisation        24 clearance sections tied to A renamed or new section
  structure and roles** fixed section Ids; roles      does not appear on the
                        Admin Personnel, Admin HR +   form without code change.
                        CPF Section Head, General     
                        Manager Admin, CPF Section    
                        Head, User In-Charge.         
  -----------------------------------------------------------------------------

# Things to Know

  ---------------------------------------------------------------------------------
  **No.**   **Observation**   **Detail and location**
  --------- ----------------- -----------------------------------------------------
  **1**     Contractual       typeId 5 gets no note, mental preparation, leave or
            settlement is     gratuity figures. GetLeaveReport skips the block for
            only a Clearance  Employment Type 2 (line 118), so AllLeaves stays
            Form              null; opening Details for a contractual employee by
                              URL would fail at Details.cshtml line 161. The pages
                              never read ContractRenewals, and no code ends a
                              contract or disables the employee.

  **2**     Pay Commission    Pay Commission = WagesToPayScaleDate filled; Wage
            and Wage          Commission = Worker without it. A pay-scale officer
            Commission are    or staff who never was a worker appears in PRL, not
            defined by one    in Retirement.
            flag              

  **3**     No role attribute None of the five pages has \[Authorize(Roles)\].
            and no            Buttons are hidden by role, but SaveFinalSettlement,
            server-side guard SaveDeptHead, PRLNote saves, SaveClearanceForm and
            on saves          SaveDisableType do not check role or Status.
                              ClearanceForm.OnPostSaveSectionRemark sets any
                              AdminClearanceForm property by name through
                              reflection, including Status and the Signatures
                              fields. Employees.OnPostDisable also has no role
                              check.

  **4**     One row per       AdminFinalSettlements and AdminClearanceForms have a
            employee, not     normal index on EmployeeId. Pages read
            unique            FirstOrDefault. Saves from PRL Note, IDN or Mental
                              Preparation create a new row when no Id is posted, so
                              two tabs opened before any row exists can create two
                              rows for one employee. PRL and final retirement of
                              the same employee share the row; once Status ≥ 1 the
                              note can no longer be edited.

  **5**     No reject path    Status moves 0 → 3 only. There is no Send Back for
            for the           Details (Clearance Form has none either), unlike the
            settlement note   Gratuity and PF approvals.

  **6**     One-day           HR note: last day = PRLEndDate + 1, then +1 again,
            difference        then GetMonthAndDays adds a day. Gratuity Statement:
            between HR and    PRLEndDate + 1 then GetMonthAndDays adds a day.
            Accounts for PRL  Service months can differ by one day at the 6-month
                              rounding boundary. PRL Pending uses birth date + 59
                              years + 1 day for PRL start; GetLeaveReport uses + 59
                              years.

  **7**     PRL Note text and PRLNote prints "day before retirement" as birth
            calculation       date + 59 years − 2 days (line 172) while the
            differ by a day   calculation ends at PRL start − 1 (birth date + 59
                              years − 1 day).

  **8**     Hard-coded values 540 encashment days; 365 PRL days; ages 59 and 60; 90
                              days notice; gazette date 1 Jan 2016 on Details but 1
                              Jul 2015 on PRL Note; Employee Ids 94 (Details
                              signature name) and 866 (PRL Note); Ids 1 and 5
                              excluded from lists; user ids for Legal, Berthing and
                              Common approvers; Mental Preparation Department Head
                              limited to Department 3 and Employment Type 7; Janata
                              Bank Dry Dock branch wording.

  **9**     Current Basic     EmployeeSalaries last non-zero BasicPay up to the To
            comes from paid   Date fiscal year. The list throws if no FiscalYears
            salary            row exists for that year (fiscalYear.Id, line 83).
                              Gratuity Payments suggests the basic from
                              EmployeeSalaryInfo instead, so the two can differ.

  **10**    Disable timing    Setting IsDisabled without DateOfDisable and without
            affects salary    PRLEndDate removes the employee from salary at once
            and gratuity      (Salaries/Index line 192). Gratuity provision stops
                              only when DateOfDisable is set.

  **11**    Lifetime          Encashed days include every approved category-15
            encashment vs     leave, but leave taken in GetLeaves counts only
            leave history     leaves starting on or after 1 Sep 2025 (plus Leave
                              Opening). Payable days are right only if Leave
                              Opening holds the earlier history.

  **12**    Default list      Status defaults to Active. After Disable the employee
            hides retired     disappears from the settlement list unless Status is
            employees         set to All or Deactive. The employee-type filter on
                              PRL and Contractual lists is present but hidden
                              (class d-none).

  **13**    Mental            The list is sorted by days left to Mental
            Preparation order Preparation, where overdue rows all show 0, so they
                              are not ordered among themselves.

  **14**    Heavy             Each forward loads all users and calls IsInRole per
            notification code user; the Clearance Form also checks every user's
                              section for the 24 blocks.

  **15**    Lookup data not   EmployeeDisableTypes, EmployeeTypes and
            in code           EmploymentTypes rows are not seeded in the project.
                              Meanings used here (for example Id 12 = death) come
                              from code and should be confirmed in the database.
                              "IDN" is used as a note name (PRL Note & IDN, PF
                              IDNNo) without an expansion in the code.
  ---------------------------------------------------------------------------------
