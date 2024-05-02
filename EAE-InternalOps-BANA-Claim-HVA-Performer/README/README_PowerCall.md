# PowerCall.xaml
## **Introduction**
This xaml files goal is to send claim data to Power and create a Power record through API calls. It does this through Power's SOAP web service. This workflow reads in a template SOAP envelope XML file, fills in required tags with the arguments passed in to the workflow, and creates the new record in Power with the `put_record` method of the API.

This README will walkthrough step by step everything with this workflow. Please contact Josh Sample for any questions.

## **Template**
There are two SOAP XML templates that this workflow can use. One is `testPowerPutRecord.xml` and the other is `prodPowerPutRecord.xml`. I tried to get it working with just one, that can swap a value depending on the enviornment, but this caused an issue with the SOAP envelope as a whole. So for times sake I opted to go the route of having two templates that change per environment. This should be changed by the **PowerPutRecordTemplate** setting in the configuration sheet. You should also update the **PowerEnvironment** setting in the configuration as well as this will determine a variable later on in the workflow, specifically the *Create XML Body* sequence.

These templates are identical with one exception, the Header tag has a different argument for a URL since this does change per the Power production URL and the Power test URL. 

For any non production environment, you'll want **PowerPutRecordTemplate** to have the value of `Config\testPowerPutRecord.xml` and **PowerEnvironment** to be `Test`.

For production, **PowerPutRecordTemplate** should be `Config\prodPowerPutRecord.xml` and **PowerEnvironment** should be `Production`. 

## **Arguments**
All arguments passed in are required. These arguments are the data that gets sent to Power.

1) **`in_RecieveDate`** - For the ***Recieved*** field in the Power record. Information is obtained from the query and should be a part of the transaction data obtained from the Orchestrator queue. The 'RECIEVE_DATE' key. 
2)  **`in_CorrectedCLM`** - For the ***Claim Number*** field in the Power record. Information is obtained from the query and should be a part of the transaction data obtained from the Orchestrator queue. The 'CORR_CLM' key.
3) **`in_CorrectedSCCF`** - For the ***SCCF*** field in the Power record. Information is obtained from the query and should be a part of the transaction data obtained from the Orchestrator queue. The 'CORR_SCCF' key.
4) **`in_Prefix`** - For the ***Alpha Prefix*** field in the Power record. Information is obtained from the query and should be a part of the transaction data obtained from the Orchestrator queue. The 'PREFIX' key.
5) **`in_ReasonCode`** - For the ***Reason Code*** field in the Power record. Information is obtained from the query and should be a part of the transaction data obtained from the Orchestrator queue. The 'RSN_CD' key.
6) **`in_BotErrorMessage`** - This is for the ***B2 Comments*** field and for the ***Action Comments*** field in the Power record. Since these claims are being sent to Power because the bot can't process them, this message will typically be some sort of error or exception message.
7) **`in_PowerQueueName`** - For the ***Queue*** field in the Power record. Queue name varies depending on what queue you want to send the claim to in Power. This is determined by pend code.
8) **`in_Config`** - The configuration file. The configuration spreadsheet contains settings important to this workflow, such as *PowerURL*, *PowerCredentials*, *PowerPutRecordTemplate*, and *PowerEnvironment*.

## **Global Variables**
These variables will persist throughout the runtime of this workflow. These are used between various sequences.
1) **`xmlString`** - The string representation of the template `PowerPuRecord.xml` file. 
2) **`xmlDeserialize`** - The deserialized SOAP XML. Initially this is the object representation of the `PowerPutRecord.xml` template, but we replace template dummy data with the argument data. This is also what gets sent to the API.
3) **`powerURL`** - Obtained from the **in_Config** argument. The URL for the Power web service.
4) **`password`** - Obtained from the **in_Config** argument, corresponds to the *PowerCredentials* asset in Orchestrator. The secure string password that is needed for authentication to Power.
5) **`username`** - Obtained from the **in_Config** argument, corresponds to the *PowerCredentials* asset in Orchestrator. The string username that is needed for authentication to Power.
6) **`powerTemplate`** - Obtained from the **in_Config** argument. This is the name of the SOAP XML file that will be read in.
7) **`powerEnvironment`** - Obtained from the **in_Config** argument. This is used to determine a variable in the *Create XML Body* section of the workflow.

## **Flowchart**
For this next section I will break down each sequence in the flowchart, step by step to explain what each sequence does.
### **Initialization - Read Config**
First sequence in the flowchart, this sequences purpose is to read in the configuration sheet and get specific settings from it.

This is wrapped in a try/catch in the event an exception occurs trying to get any of the variables/getting the credentials. It assigns values to variables to get the data from the sheet. Some of these are global variables, but *powerCredential* is scoped to just this sequence as we look for it's string value to assign the `username` and `password` variables.

`PowerCredentials` is a robot value credential asset in Orchestrator, whichever user account is running this process needs to be sure they are a part of the Power web service Active Directory group. You may need to reach out to a Power business analyst for assistance.
### **XML Deserializing**
This sequence reads in the SOAP envelope XML template and deserializes it to an object.

Wrapped in a try/catch in case there is an error reading in the file, it reads the XML template file and stores it in the `xmlString` global variable, then uses a deserialization activity to translate the string to an object that can be parsed more easily. This is deserialized to the `xmlDeserialize` global variable.
### **Create XML Body**
This sequence parses the `xmlDeserialize` object and breaks it down to the `<web>` tag of the SOAP XML so it can change those dummy values from template with the data that is passed in through arguments.

The first two assignment variables are **`soapenv`** and **`web`**. This is required in order to get the specific nested tags so we can change out data.
`soapenv` comes from the XML template. If you were to look at the template, notice how each of the main tags start with `<soapenv>`? For example, on the SOAP XML the *Envelope* tag looks like this: `<soapenv:Envelop>`. Then you have the *Header* and *Body* tags also starting with this. To parse out the values within these tags, and to get specifically to the `<web:put_record>` tag which has all the values we need, we need to have these two variables. And their value is a URL since if you look at the template, the attributes of the `<soapenv:Envelop>` tag has both this `xmlns:web` and `xmlns:soapenv` attribute. And the values these attributes have are URLs, which is why these two variables are URL strings. *The whole purpose of these variables is so we can parse out the XML down to the values we actually need to change*.

The **`web`** variable is determined by an if statement. If we're are in production, determined by the PowerEnvironment setting in the configuration sheet, then we want to assign **`web`** the production URL. Else, assign it the test URL. I had to do it this way because otherwise the SOAP XML would break. 

**`baseXMLParse`** is our next assignment activity. This is the base variable for changing attributes. Changing our template to having real values that need to go to Power queues all happens because of this variable. Updates to this variable does in fact update our `xmlDeserialize` variable so no need to worry about our grander XML that we send to the web service. `baseXMLParse` gets its value from actually breaking down the `xmlDeserialize` variable into an object where it is easier for us to modify. 
`xmlDeserialize.Element(soapenv + "Envelope").Element(soapenv + "Body").Element(web + "put_record")` translates to: from the template, we need to go in the Envelope tag and get the Body tag, and then we need to get the put_record tag that's inside the Body tag. A little confusing I know, but the entire reason for this assignment is to get to the tags we need to change.

The next few assignments all leading up to a logging activity are all changing out the dummy values of specific tags with the arguments passed in to the workflow. Note we do not have anything for the `<rec_ID>` tag because we aren't updating a record, we're creating a new one. The tags updated include:
- `<queue>` - updated with **in_PowerQueueName** argument.
- `<fields>` - this tag is a complex type, which in SOAP basically means it's an object. In this case theres an even extra layer of confusion since this is an array of *fieldValue*. Please refer to the Power documentation linked at the bottom for more information on the *fieldValue* datatype. Since this tag is an array of *fieldValue* objects, we have several tags underneath which we change out for argument values. Each *field* tag in order is as follows:
  - <`field`> - This first one is for the **in_ReceiveDate** argument.
  - <`field`> - The second one is for the **in_CorrectedCLM** argument.
  - <`field`> - Third is for the **in_CorrectedSCCF** arguement.
  - <`field`> - Fourth is for the **in_Prefix** arguement.
  - <`field`> - Fifth is for the **in_ReasonCode** arguement.
  - <`field`> - Sixth is for the **in_BotErrorMessage** arguement.
- `<comment>` - Another tag that **in_BotErrorMessage** fills in. Since this is for a required field on a Power record we use **in_BotErrorMessage** twice.

### **Power Call - put_record** 
Our last sequence calls the Power SOAP web service to create a new record.
Wrapped in a try/catch in case there is an error when making the web service call, we use an HTTP Request activity instead of a SOAP Request because our SOAP envelope has complex types in it. So we couldn't use UiPath's activity as a result, since the SOAP Request activity only handles simple name value pair tags. To break down our HTTP Request:
- Endpoint is the `powerURL` variable
- Method is POST since we are sending data
- Body is our `xmlDeserialize` object
- BodyFormat is text/xml (not application/xml due to web service needs)
- Result is a new variable, `ticketNumberXML`
- StatusCode has a new variable, statusCode. Mainly for debugging purposes we can see the HTTP response code from the web service call
- SecurePassword is `password` variable
- Username is `username` variable

We store the output from the HTTP request in the `ticketNumberXML` variable. This is a string so we do another deserialize XML activity on it to a `ticketNumberDeserialize`. This variable is our response from the web service and so we need to parse out the info that matters to us, the *ticket number*.

The next to activities are logging statements, but I do want to mention that we get the number directly by this piece of code: `ticketNumberDeserialize.Root.Value`. This parses the XML to give us the ticket number in Power.

## **Final**
This should about cover everything about this workflow in detail. If you still have any questions please reach out to Josh Sample or the Enterprise Automation Engineering team. Should this workflow be updated in the future, please update this document to keep the information here accurate. Below are some resources I recommend checking out. The YouTube video linked helped me create this workflow, especially since we were dealing with complex types, so I highly recommend checking it out for yourself.

## **Resources**
[Power web service documentation](https://abcbspower/power/webservices/record.php)

[UiPath Tutorial for SOAP and XML (Create and Deserialize XML) | UiPath Soap Request](https://www.youtube.com/watch?v=3nUtbExwP6k)