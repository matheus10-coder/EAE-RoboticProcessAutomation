### Documentation is included in the Documentation folder ###

[REFrameWork Documentation](https://github.com/UiPath/ReFrameWork/blob/master/Documentation/REFramework%20documentation.pdf)

### HVA Modernization Project ###
**Process Name: BANA_HVA_Adjustments**

* Built on top of *REFramework* template
* Uses *State Machine* layout for the phases of automation project
* Offers high level logging, exception handling and recovery
* Keeps external settings in *Config.xlsx* file and Orchestrator assets
* Pulls credentials from Orchestrator assets per robot basis
* Gets transaction data from Orchestrator queue and updates back status
* Takes screenshots in case of system exceptions
* Error handling for Business and Application exceptions follow the HVA modernization plan, described in details later.

### How It Works ###

1. **INITIALIZE PROCESS**
 + ./Framework/InitiAllSettings* - Load configuration data from Config.xlsx file and from assets
 + ./Framework/*GetAppCredential* - Retrieve credentials from Orchestrator assets or local Windows Credential Manager
 + ./Framework/*ABCBS_InitiAllApplications* - HVA Initializer module, responsible to set up directories and login to Amisys Advance


2. **GET TRANSACTION DATA**
 + ./Framework/*GetTransactionData* - Fetches transactions from an Orchestrator queue defined by Config("OrchestratorQueueName") or any other configured data source

3. **PROCESS TRANSACTION**
 + ./Framework/ABCBS_Process* - Process trasaction encapsulating pre-Claims Processing, ClaimsProcessing and Pos-Claims Processing. In case of BE this process will attempt to make API calls to Power and send that transaction over. 
    Pre Claims Processing: Flowchart to initalize variables used in Amisys, check for successful claims criteria and navigate to medical claims in Amisys Advance
    Claims Processing: Core of claims adjustment. It involves Medical Adjustment and Readjudication procedures.
    Pos Claims Processing: After successfully claims adjustment it will check HPA Manual value, clean pages and write successful records to output
 + ./Framework/*SetTransactionStatus* - Updates the status of the processed transaction (Orchestrator transactions by default): Success, Business Rule Exception or System Exception
 + ./AmisysModules/AmisysNavigateToHome** - Amisys module responsible to trigger activities to safely navigate the process to Amisys home page
    **Exit** 
    + ./AmisysModules/RestoreAmisysSession* - After 5 consecutives BE process will restore amisys session and try again
    + Terminate Process - close the application and terminate the process after 8 consecutive BE

4. **END PROCESS**
 + ./Framework/*CloseAllApplications* - Logs out and closes applications used throughout the process

 ### Process Overview ###
  HVA Query runs and dispatcher service will load the queue. 3AM performer is triggered. Bots send emails via Emailer Service once the process is completed by success or exceptios. Only successful transactions will be written to Output file.

 1. **OBJECTIVE**
  Main goal is to reverse original claim and readjudicate the corrected claim, check for claim status and HPA manual. Hospital claims are excluded from this process.
 2. **HVA MODERNIZATION**
  Project created to replace the current hva process in order to make it more robust and efficient to maximize the error handling procedures, such as business exceptions and application exceptions before terminate the process. New features are in place such as AmisysModules, which is Amisys process pieces being encapsulated to reusability opportunities; configuration sheet created to store variables and constants associated with the process; web service to replace the dispatcher; new directory map and multiple Amisys session allowed. The main goal is to have the process running 24/6 minimizing the chances of having a faulted job and maximizing the adjustment of claims. Three main features have been developed: HVA Initializer; ABCBS Claims Process and Exception Handler
 3. **HVA INITIALIZER**
  Module ABCBS_InitAllApplications created to initialize the process. The goal is to handle the initialization part through 3 maximum attempts to succeed through background procedures and Amisys login. The background procedures contain three stages: 1- Retrieve amisys info; 2- Map directories; and 3- Power queue initialization. The Amisys login is invoked and login center procedure is triggered. HVA initializer will have 3 attempts to complete successfully.
 4. **ABCBS CLAIMS PROCESS**
  Module ABCBS_Process.xaml created to handle the core functionality for Claims Processing in Amisys. This module involves three components: 1- Pre ClaimProcessing, responsible to initilize local variables, execute claim check, exclude flags check, and navigate to medical adjustment; 2- Claims Processing, responsible to execute claims search, reverse claim, check claim status, readjudicate correcte claim; 3- Pos Claims Processing, responsible to check HPA manual value, clean pages and write to Output file claims information. If any claims has been throw a business exception during this module it invokes Power call and send this claim to the correspondent power queue and create a record ticket.
 5. **EXCEPTION HANDLER**
   Exception handler logic has been applied throughout the process in order to handle  business and applications exceptions. The goal is to have the minimum chances of failure and maximize the successful rate. The process will be considered successful in 4 different scenarios: SC 1 -> No trasaction in the queue; SC 2 -> Job stopped in Orchestrator; SC 3 -> 10 Consecutive application exceptions (process will stop in order to business area to review, emails will be sent with the specific information) and SC 4 -> HVA initializer failed in 3 attemtps (process will stop in order to business area to review, emails will be sent with the specific information). The process will be considered faulted in 2 different scenarios: FA 1 -> 8 Consecutive business exceptions; FA 2 -> Fail due package dependecies. If constant "ShouldMarkJobAsFaulted" is changed to TRUE in the config sheet, SC 3 and SC 4 will be changed to faulted status as well.

### Handling Exceptions ###
 1. **APPICATION EXCEPTION**
   Configuration sheet has the value of max application exception allowed in the process. Once the limit of this consecutive application exceptions is reached, the process will trigger the finish the job safely, and send out emails to the business area. Every application exception will be counted. For 5 consecutive application exception the process will trigger the restore amisys session. After 5 we will keep counting until we reach a total of 10 AE. If 10 application exception is reached in the process we will gently finish the job and send the results for review. Application exceptions are all but not limited by selector issue, amisys crash, page or element not opened and others.
 2. **BUSINESS EXCEPTION**
  Configuration sheet has the value of max business exception allowed in the process. Once the limit of this consecutive business exceptions is reached, the process will trigger the terminate the job, and will send out emails to the business area. Every business exception will be counted. For 5 consecutive business exception the process will trigger the restore amisys session. After 3 now, we will keep counting until we reach a total of 8 BE. If 10 business exception is reached in the process we will gently finish the job and send the results for review. Application exceptions are all but not limited by selector issue, amisys crash, page or element not opened and others. To be considere BE consecutive exception, the process will keep track of the older message and compare with the new one if they are a match we keep looping until we reach 5 or 8. 

### ABCBS Modules ###
   1. **AMISYS MODULES**
   - CheckProcedureCode: Check for the procedure codes at corrected claim level (readjudication screen) with specific a prefix defined at the config sheet (ExcludePrefix). This will exclude the claim from the adjustment
   - Chrome PopUp_ChangesMade: Check for chrome popup for changes made in the page
   - ClaimSearch: Claims search for Medical Adjustment (Original Claim) and Medical Readjudiction (Corrected Claim). In order to work properly, you either provide the original claim OR the corrected claim 
   - ClaimStatus: Return back the status of the current claim
   - ClaimsThirtyPartyAnalisysScreen: Check if CL4440 is opened and perform a safe close page
   - ClearPage_F4: Clear the page in amisys
   - CloseTab: Receive the browser object to be closed
   - DynamicTimer: Delay dynamic time to be used during readj phase
   - GetAllProcedureCodes: Identify and retrieve all the procedure code in the service line
   - GetMemberNumber: Retrieve the member number in amisys
   - Login: Perform a safe login in amisys
   - Logout: Perform a safe logout from amisys
   - MedicalAdjustment_InterPlan_HPAManualProcess: Insert 'K' in HPA Manual at the ITS Inter Plan screen
   - MedicalReadjudication_InterPlan_FundCd: Insert 4 at the FundCd field in ITS Inter Plan screen
   - MedicalReadjudication_InterPlan_HPAManualValue: Retrieve the last value of HPA field at ITS Inter Plan screen
   - MedicalReadjudication_CheckExCodes: Retrieve the first service line EX Codes
   - MedicalReadjudication_GetAllEx1Codes: New module created to promove efficiency when retrieve code at EX1 field. As soon as we identify f0 has been found we won't proceed as we want to override f0 on the HVA process
   - MedicalReadjudication_ClaimReadjudicate: Perform the readjudication procedure in the claim
   - MedicalReadjudication_EXOverrides: Open ExOverride screen and depending on the claim it will override f0 with Y0 and LM with Y0
   - MedicalReadjudication_PageUnresponsiveHandler: Check for the chrome popup page unresponsive and close it safely
   - NavigateToClaimProcessing: Naviagte to the claims processing tab
   - NavigateToHome: Naviagte to Amisys Advance home page
   - NavigateToITSInterPlanWindow: Open ITS Inter plan screen
   - NavigateToMedicalAdjustment: Open Medical Adjustment screen
   - NavigateToMedicalReadjudication: Open Medical Readjudication screen 
   - AmisysReverseClaim: Type MX into the Backout ex code field 
   - AmisysServiceLineRetrieval: Identify how many service line the current claim contains
   - RestoreAmisysSession: After 5 consecutive exceptions we should restore the page in amisys wait 20s and closing out the application and perform the login again
   - ServiceLineRetrival: Retrieve in amisys the service line total value 
   2. **FRAMEWORK ABCBS MODULES**
   - ABCBS_BusinessExceptionHandler: checks for consecutive business exceptions through 1-5 then trigger restore amisys session, then keep counting for more 3 consecutive BE to terminate the process. 
   - ABCBS_CloseAllApplications: close amisys application through a logout flow, if fails Kill All Process will be trigger
   - ABCBS_DateCalculation: calculate the difference between received date and current date in order to exclude or not a claim from bot adjustment. It works with date format "MM/dd/yyyy"
   - ABCBS_EmailNotification: send out emails from the process
   - ABCBS_InitAllApplications: HVA initializer following the 3 attempts method
   - ABCBS_Process: Core functionality for Amisys Claims Processing
   - ABCBS_WriteToOutput: write successful records to an output spreadsheet.
   3. **POWER ABCBS MODULES**
   - PowerCall: makes call to Power API that will pass the claim information to its specific queue
   - PowerQueueSelection: selects the specific queue mapping the claim prefix indicated during the Amisys variables intialization part
 **All this modules should have built-in documentation in their own files

### For New Project ###

1. Check the Config.xlsx file and add/customize any required fields and values
2. Implement InitiAllApplications.xaml and CloseAllApplicatoins.xaml workflows, linking them in the Config.xlsx fields
3. Implement GetTransactionData.xaml and SetTransactionStatus.xaml according to the transaction type being used (Orchestrator queues by default)
4. Implement Process.xaml workflow and invoke other workflows related to the process being automated
