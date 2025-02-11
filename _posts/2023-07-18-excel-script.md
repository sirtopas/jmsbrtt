--- 

layout: post 

title: "Getting to grips with ExcelScript" 

--- 

## Introduction 

Within the [BMA]() team, we have a requirement to store certain information about our users, includihng their email address. We make use of [Windows Authentication]() in all of our applications, which means that our users don't need to provide a username and password and instead their credentials are passed on with the requests they make. 

There are a host of benefits in using Windows Authentication; some might argue that chief among them is that we no longer need to store passwords, which presents umpteen issues in and of itself: hashing, salting, rotating, encrypting and various other verbs that can land you in hot water if not implemented perfectly. 

Passing that username through with requests is especially helpful in the context of our applications. With that username, we can perform checks for a multitude of things: 

- Does this user have the correct role to access this resource? 

- Is this user still active? 

Whilst the Username variable of the Windows Authentication request remains consistent, other fields do not. There are reasons for this, they could have been setup in a different way on the domain or they could be on domain that differs from the one our servers sit on. To combat this, we capture the information we require from the user; this includes: 

- Email Address 

- Phone Number 

- First and Last name 

This information may be accurate at the time of submitting a request, but equally the user could have entered the wrong email address, missed a 0 from their phone number or, as we've had in some cases, misspelled their own name. As part of trying to keep that information up to date and confirm the details we have, the post below details how we can leverage [ExcelScript/Office Scripts](https://learn.microsoft.com/en-us/javascript/api/office-scripts/excelscript) to call an internal API which attempts to get user details from the domain and compare it to what we have stored. 

The main reason for using Excel is ease of use; although we could open the API up to the users that need it, an easy to use spreadsheet which automates the work make the process simple enough for anyone to use. 

## Getting started with ExcelScript 

When trying to load or transform data in Excel, I have historically made use of the in-built VB.NET editor. Whilst useful, Visual Basic is not a language I keep up to date on, some of the syntax can feel clunky and I will be constantly looking up the names of methods I need. Similarly, having used Visual Studio, VS Code and the Chrome Developer tools, the debugger and editor in Excel leave a bit to be desired. 

I started by investigating the best way to call an API from VB.NET and found some examples that looked more like Typescript than Visual Basic! That got me searching and I soon stumbled on ExcelScipt. 

There are a couple of ways to make use of Excel's Office Scripts: Action Recorder and the Code Editor: 

![Action Recorder](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-recorder.png){:style="display:block; margin-left:auto; margin-right:auto"} 

Action Recorder works in a similar way to recording Macros - start recording, complete a set of actions and top. Once you're happy, you can replay that action as many times as you want. 

![Code Editor](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-editor.png){:style="display:block; margin-left:auto; margin-right:auto"} 

es-editor.png 

The Code Editor is a bit more interesting and that's what we'll dive into today. 

### Running Code in Excel 

To get started, open the 'Automate' tab and use the 'New Script' option: 

![Dev Tab](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-dev-tab.png){:style="display:block; margin-left:auto; margin-right:auto"} 

Once pressed, we get an editor! 

![Editor](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-new-editor.png){:style="display:block; margin-left:auto; margin-right:auto"} 

Not only can we code away using this editor, we get some really neat features: intellisense to help complete our code, a console output and a way to save these scripts so that they can be shared and pushed to source control if needed. 

The documentation for ExcelScript has been helpful so far and consistent with what we'd expect from their .NET documentation. In particular, the [API Reference](https://learn.microsoft.com/en-us/javascript/api/office-scripts/overview?view=office-scripts) helped massively with exploring the methods and objects to use. 

There is a list of common classes that we can make use of when developing our script: 

- Workbook 

- A Workbook, as you may already know, is in most instances, the spreadsheet you have open 

- It consists of one or more Worksheets 

- Worksheet 

- A Worksheet is a a set of cells, most often shown as the tabs at the bottom of your Workbook 

- Range 

- A Range usually represents a series of Cells 

To demonstrate this, let's take a look at a very simple script: 

```typescript 

function main(workbook: ExcelScript.Workbook) { 

// Set fill color to FFC000 for range Sheet1!A2:C2 

let selectedSheet = workbook.getActiveWorksheet(); 

selectedSheet.getRange("A2:C2").getFormat().getFill().setColor("FFC000"); 

selectedSheet.getRange("A3:C3").getFormat().getFill().setColor("yellow"); 

} 

``` 

In the example above, we're finding the worksheet, selecting a range and setting the colour of the cells. Simple and readable. As mentioned above, we even get some nice intellisense when coding: 

![Intellisense](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-intelli.png){:style="display:block; margin-left:auto; margin-right:auto"} 

Not only will the editor suggest methods we can use but we'll also get the description of that method and what it returns. 

## Searching the Domain 

Now that we've got ExcelScript up and running, it's time to implement a method to compare our user data to the data stored on our domain. 

Thankfully, most of this work is already complete; when developing the new UAM application, we added a new method which does something similar. When a user is being registered or changed on UAM, an option is available to 'check the domain' - this functionality simply fires off a request to the GLOBAL.Lloydstsb.com domain with the user's username and brings back the fields we're interested in. In turn, the UAM UI will compare the response to what is already stored and offer an easy way for the user to automatically updates those fields if they differ. 

The code to call the domain is quite simple: 

```csharp 

using (PrincipalContext ctx = new PrincipalContext(ContextType.Domain, "GLOBAL.lloydstsb.com")) 

{ 

var user = UserPrincipal.FindByIdentity(ctx, username); 

if (user is null) 

{ 

return null; 

} 

UserPrincipal userPrincipal = new UserPrincipal(ctx); 

var entry = (DirectoryEntry)user.GetUnderlyingObject(); 

userDetails.UserName = username; 

userDetails.FirstName = user.GivenName; 

userDetails.LastName = user.Surname; 

userDetails.Email = user.EmailAddress; 

userDetails.PhoneNumber = user.VoiceTelephoneNumber; 

userDetails.HRNumber = user.Name; 

} 

``` 

It searches the domain for that user using the [FindByIdentity](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.accountmanagement.userprincipal.findbyidentity?view=dotnet-plat-ext-7.0) method and returns the information we're interested in. There is a lot of information available in this [`UserPrincipal`](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.accountmanagement.userprincipal?view=dotnet-plat-ext-7.0) object but so far, we're just returning the basics: name and contact information. 

There is also the possibility that this user is not found on the domain - there's a number of reason that this might be the case: the user is on another domain, they've been setup differently or they could have even left the business and no longer exist. To handle this, we'll return a [`204 (No Content)` Status Code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/204). By returning the 204 response, we know that although nothing failed, there was nothing to return. This way we can ascertain whether or not the user's information was found on the domain. 

## Calling the API from ExcelScript 

Now onto the interesting part: using the endpoint we created above to get information about each user we have and show their details in Excel. 

The first step is to insert the 'base data' we have for these users; a simple SELECT statement will do here that returns the username and emailAddress for all active users: 

![Base](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-excel-base.png){:style="display:block; margin-left:auto; margin-right:auto"} 

With that data in place and using the knowledge we gained from the Microsoft documentation, we can access the data in those cells to get the values: 

```ts 

async function main(workbook: ExcelScript.Workbook) { 

// Access the worksheet 

const sheet = workbook.getActiveWorksheet(); 

// Get 'used' range which contains username and email 

const users = sheet.getUsedRange().getValues(); 

users.forEach(async user => { 

console.log(user) 

}); 

} 

``` 

The script so far is very simple: 

1. Get the active worksheet (the one with user details in) 

1. Get the ['Used Range'](https://learn.microsoft.com/en-us/javascript/api/office-scripts/excelscript/excelscript.range?view=office-scripts#excelscript-excelscript-range-getusedrange-member(1)) - the used range contains the values for all cells that have contents (in our case, all the usernames and email addresses) 

1. For each record found, log them in the console 

The last step here is simply for debugging, to ensure we're getting the data we expect. Excel gives us a helpful Output windows which not only shows us this output but also parses it into JSON so that we can view the whole object: 

![Output](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-excel-output.png){:style="display:block; margin-left:auto; margin-right:auto"} 

With the help of the tooling, we can see that ExcelScript has found the correct range we were looking for. Now we need to call our new endpoint! 

To call the API from ExcelScript, we'll make use of [Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API). Fetch provides an interface for fetching resources and will allow us to call our API straight from Excel. There is one caveat here: authentication. As mentioned above, we make use of Windows Authentication to guard our APIs against unauthorised users. Fetch won't send credentials by default so we'll need to send those by modifying a header. Let's dive into the code: 

We already have the array of users we entered by using the getUsedRange method so we'll iterate over each of them and call the API: 

```ts 

users.forEach(async user => 

{ 

const username = user[0]; 

const response = await fetch(`https://localhost:44342/api/v1/user/details?username=${username}`, { credentials: 'include' }); 

} 

``` 

For each user row that is found, call our API endpoint passing in the username as the parameter - simple! We are able to send through our credentials using the credentials header - this will authenticate the user running the script against the API and only execute if they have the correct rights. 

Once again, we'll log the response and make sure it's working as expected: 

```json 

fullName: "James Brett" 

managerName: "0000000" 

approver: "Permanent Employee" 

userName: "9438985" 

firstName: "James" 

lastName: "Brett" 

email: "James.Brett@lloydsbanking.com" 

phoneNumber: "00000000" 

``` 

We get a well formatted JSON response that gives us the fields we wanted! With that information to hand, we can do some simple comparisons to compare what we have stored to that stored on the domain. 

To make this easier, we'll create an [interface](https://www.typescriptlang.org/docs/handbook/interfaces.html) to map the response to. Once mapped, we can then use the properties of that object to do our comparisons: 

```ts 

// An interface matching the returned JSON for a User. 

interface User { 

userName: string; 

firstName: string; 

lastName: string; 

fullName: string; 

email: string; 

phoneNumber: string; 

managerName: string; 

} 

``` 

With that in place, we'll add some code to: 

1. Check that the response contained content (a response code 200 has data and a 204 is empty) 

1. Map the user to our interface 

1. Compare the email part of the response to what we have stored 

1. Add the result to an adjacent cell 

```ts 

// Check if the response had content 

if (response.status === 204) { 

const cell = sheet.getCell(row, 2); 

cell.setValue('User not found on domain'); 

cell.getFormat().getFill().setColor("yellow"); 

row++; 

} else { 

// Map the user to the interface 

const domainUser: User = await response.json(); 

// Get the next empty cell 

const cell = sheet.getCell(row, 2); 

// Check if the email is not null 

if (domainUser.email != null) { 

// Set the email value  

cell.setValue(domainUser.email); 

const currentEmail = sheet.getCell(row, 1).getValue(); 

// Compare the response to what we have stored 

if (domainUser.email.toLocaleLowerCase() != currentEmail.toString().toLocaleLowerCase()) { 

// If it differs, make it RED 

cell.getFormat().getFill().setColor("red"); 

} else { 

// If it's the same, make it GREEN 

cell.getFormat().getFill().setColor("green"); 

} 

} else { 

// If there was no email in the response, make it ORANGE 

cell.setValue('Email not found on domain response'); 

cell.getFormat().getFill().setColor("orange"); 

} 

// Iterate to the next row 

row++; 

} 

``` 

Let's give the code a run and see how it looks! 

![Output](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-api-results.png){:style="display:block; margin-left:auto; margin-right:auto"} 

The data has been maked but the results are taken from a real run of this script. You can see the yellow boxes show those that could not be found, green where the response matched what we already hold for the user and red for where the responses differ. The last response highlights a common pattern we saw: old domains to new ones. It could be that the user was initially setup many years ago so a lot of the differences were around the domain (Lloydstsb to Lloydsbanking, etc.) 

Once this had run through the thousands of users we have, we got a good picture of how accurate our data was. 

### Interesting findings 

Before we dive into the findings, it's worth noting that we're just scratching the surface here with the data we're using from the domain. The links provided above give a good overview of the interfaces, methods and fields available but there was one that piqued our interest: Employee Type. There are parts of the bank that require you to say what type of employee you are; Permanent and Contractor being the most used. As with email, we can't always rely on this data being supplied as things change and a contractor might turn permanent or the other way around. Once again, getting this information from the 'golden source' is always preferable. 

Let's alter the API endpoint to also return the 'Employee Type' field: 

```csharp 

using (PrincipalContext ctx = new PrincipalContext(ContextType.Domain, "GLOBAL.lloydstsb.com")) 

{ 

var user = UserPrincipal.FindByIdentity(ctx, username); 

if (user is null) 

{ 

return null; 

} 

UserPrincipal userPrincipal = new UserPrincipal(ctx); 

var entry = (DirectoryEntry)user.GetUnderlyingObject(); 

userDetails.UserName = username; 

userDetails.FirstName = user.GivenName; 

userDetails.LastName = user.Surname; 

userDetails.Email = user.EmailAddress; 

userDetails.PhoneNumber = user.VoiceTelephoneNumber; 

userDetails.HRNumber = user.Name; 

userDetails.EmployeeType = user.EmployeeType; 

} 

``` 

We'll modify our Typescript interface to include the new field and change our ExcelScript code to output that new field against the user: 

```ts 

const employeeTypeCell = sheet.getCell(row, 3); 

typeCell.setValue(domainUser.employeeType); 

``` 

When we run this script, we get some interesting results: 

![Types](https://pages.ghe.service.group/James-Brett/blog/assets/images/excel-script/es-types.png){:style="display:block; margin-left:auto; margin-right:auto"} 

The results showed that most of the users were permanent, followed by contractors but there were a fair few Robot Identities, too. Interestingly, users of type Robot had a much higher percentage of having the wrong email. It could be that the human who set these robots up didn't know the email at the time or thought that the email would be useless as they were a robot identity. 

## Conclusion 

With a simple script made in Excel using ExceScript, we can achieve a lot. Calling an API endpoint was trivial, as was mapping the response to an object we can manipulate. This post only scratches the surface of what ExcelScript can do; charting, drawing, pivots, validation - it's all possible with ExcelScript. 

Searching the domain however felt a bit more sluggish. Domains are complex by their nature and you won't always get the same result for every search due to how the users are setup which made this exercise tricky. This presented a few issues with formatting and accessing the data as well as finding commonality between users. 

We did however identify some useful metrics about our user-base as well as being able to update our stored data without needing to reach out to each user individually. 
