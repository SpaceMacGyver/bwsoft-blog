---
title: QB Desktop in PowerShell
date: 2020-03-24T12:01:11.000Z
description: 'A quick start to using the QB Desktop API in PowerShell'
categories:
    - Development
tags:
    - PowerShell
    - QB
    - COM Interop
---
One of my more recent engagements has seen me employing PowerShell scripting to accomplish several back-office functions, typically kicked off with scheduled tasks.  Among those functions, a recurring theme has been accessing data contained within the QuickBooks Enterprise installation the client maintains.

I write this post because, when I started this project, I wasn't able to find any useful examples of other people doing similar work.  Thankfully, there are aspects of this effort that were documented in assorted places, to which I will be referring to as appropriate, and it is my hope that somebody encounters this post and is saved a great deal of grief.

## Topics of Interest

Getting into this, know that we'll be encountering the following technologies/toolsets, and while I hope to present everything needed to get started, you may find yourself having to brush up on some topics:

* [COM](https://docs.microsoft.com/en-us/powershell/scripting/samples/creating-.net-and-com-objects--new-object-?view=powershell-5.1#creating-com-objects-with-new-object)
* [.NET XML DOM](https://docs.microsoft.com/en-us/dotnet/standard/data/xml/xml-document-object-model-dom)
* [Working with QB sessions](https://developer.intuit.com/app/developer/qbdesktop/docs/develop/connections-sessions-and-authorizations)
* [QB Desktop API](https://developer.intuit.com/app/developer/qbdesktop/docs/api-reference)

## Prerequisites

First, and perhaps most obvious, to do this you will need QuickBooks.  If this is not already something you have, it is possible to obtain a copy of the QB software using their [Not For Resale (NFR) licensing](https://developer.intuit.com/nfr/) for development/testing purposes.

In addition, you will want the [Desktop SDK](https://developer.intuit.com/app/developer/qbdesktop/docs/get-started/download-and-install-the-sdk).

Finally, it needs to be noted that this process is only expected to work in a 32-bit PowerShell environment.  If necessary, you should obtain a x86 version of PowerShell.

## Getting Started

I strongly recommend prefacing your scripts with a guard to ensure that you're running 32-bit PowerShell:

```powershell
if ([System.Environment]::Is64BitProcess) {
    throw "Please restart me in 32-bit PowerShell."
}
```

If you make it past this prompt, then you should next try to instantiate yourself a request processor to handle communications with QuickBooks:

```powershell
$requestProcessor = New-Object -ComObject QBXMLRP2.RequestProcessor
```

If you encounter any errors, possible explanations include:

* The QB Desktop SDK is not installed
* You're not running in a 32-bit process
* The class name has a typo

You'll also want to pull in some supporting types from the interop assemblies:

```powershell
[System.Reflection.Assembly]::LoadWithPartialName("Interop.QBXMLRP2")
```

If you made it this far, terrific!  This is the foundation on which the rest of the application will be built.

## Fault Tolerance

I haven't explored it in-depth, but I've been led to understand that you want to clean up after yourself when working with QuickBooks sessions.  From this point on in the application, you'll want to take some extra steps to tear down any resources you might have handles on, so let's create ourselves a try/finally block:

```powershell
try {
    # We'll be creating some session-related resources here
} finally {
    # Cleanup goes here
}
```

We'll be revisiting this section of code periodically as the application evolves.

## Creating Our Session

Before you go any further, there are some things you should come in knowing about your application:

1.  What are you calling your script/API?
2.  Which SDK interaction(s) do you need?
3.  Where is your company file?

Point 1 is important, because as you establish a connection to QuickBooks you will be effectively namespacing your attempt with a custom name.  When running in unattended mode in particular, you will need to have designated that the connection made under this name has the [necessary rights to interact programmatically](https://developer.intuit.com/app/developer/qbdesktop/docs/develop/connections-sessions-and-authorizations#authorizations) (this should become apparent shortly).

As for point 2, you came here for a reason, right?  Refer back to the [API Reference](https://developer.intuit.com/app/developer/qbdesktop/docs/api-reference) documentation to find the specific behavior you wish to perform.

For illustration purposes, I'll be working with the following assumptions:

* We're calling our script "API Example 123" (sorry, I'm bad at naming stuff)
* We are planning to run a [Payroll Summary report](https://developer.intuit.com/app/developer/qbdesktop/docs/api-reference/payrollsummaryreportquery) with a custom date range, and are otherwise leaving everything as default.
* I've picked up one of the sample company files (service-based business) from QB and placed it in my Documents folder.

The API Reference documentation will explain (in the XMLOps tab for a given method) the exact structure of both the request and response XML structures.  We'll be stripping this document down to just the minimum necessary information to perform the request.

Below, I will prepare these three points of information, placing them just above the try block:

```powershell
$applicationName = "API Example 123"
$pathToCompanyFile = "~/Documents/sample_service-based business.qbw"
$requestDoc = [xml]@"
<?xml version="1.0" encoding="utf-8"?>
<?qbxml version="13.0"?>
<QBXML>
    <QBXMLMsgsRq onError="stopOnError">
        <PayrollSummaryReportQueryRq>
            <PayrollSummaryReportType>PayrollSummary</PayrollSummaryReportType>
            <ReportPeriod>
                <FromReportDate />
                <ToReportDate />
            </ReportPeriod>
        </PayrollSummaryReportQueryRq>
    </QBXMLMsgsRq>
</QBXML>
"@

# Set some date range parameters (also could have just buried this in the XML):
$requestDoc.SelectSingleNode("//FromReportDate").InnerText = "2015-01-01"
$requestDoc.SelectSingleNode("//ToReportDate").InnerText = "2019-12-31"
```

With these requirements in place, we can now proceed to create and execute our request.  Let's revisit that try/finally block and flesh it out as follows:

```powershell
try {
    # Open the connection
    $requestProcessor.OpenConnection2(
        "",
        $applicationName,
        [Interop.QBXMLRP2.QBXMLRPConnectionType]::localQBD
    )

    # Create a session
    $session = $requestProcessor.BeginSession(
        (Get-Item $pathToCompanyFile).FullName,
        [Interop.QBXMLRP2.QBFileMode]::qbFileOpenDoNotCare
    )

    # Make our request
    $response = [xml]$requestProcessor.ProcessRequest($session, $requestDoc.OuterXml)
} finally {
    # Close session first
    if ($null -ne $session) {
        $requestProcessor.EndSession($session)
    }
    # ...and then close connection
    if ($null -ne $requestProcessor) {
        $requestProcessor.CloseConnection()
    }
}
```

## Firing It Up

You'll want to have your company file open in QuickBooks before proceeding, as you will be required to authorize the request.  Expect error messages in the response XML to indicate what privileges need to be granted in order to successfully execute.

Supposing you get past all that and all went well, we should have successfully retrieved the XML payload for the Summary Report response.  What you end up doing with this response is up to you, but note that the expected structure (as with the accompanying request) is described in the XMLOps tab for that API method.

The full example is available below.
<script src='https://gitlab.com/snippets/1956344.js'></script>
