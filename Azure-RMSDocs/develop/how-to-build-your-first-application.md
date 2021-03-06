﻿---
# required metadata

title: IPCHelloWorld - an example application | Azure RMS
description: This topic contains instructions to create an example rights-enabled application.
keywords:
author: bruceperlerms
manager: mbaldwin
ms.date: 08/24/2016
ms.topic: article
ms.prod:
ms.service: rights-management
ms.technology: techgroup-identity
ms.assetid: 581451A2-9558-4D0D-9D01-BEAB282C5A83
# optional metadata

#ROBOTS:
audience: developer
#ms.devlang:
ms.reviewer: shubhamp
ms.suite: ems
#ms.tgt_pltfrm:
#ms.custom:

---
** This SDK content is not current. For a short time, please find the [current version](https://msdn.microsoft.com/library/windows/desktop/hh535290(v=vs.85).aspx) of the documentation on MSDN. **
# IPCHelloWorld - an example application

This topic contains instructions to create an example rights-enabled application.

This simple application, IPCHelloWorld, will help orient you to the basic concepts and code of a rights-enabled application.

Download the sample application, [Webinar\_Collateral.zip](https://connect.microsoft.com/site1170/Downloads/DownloadDetails.aspx?DownloadID=42440), from Microsoft Connect. The remaining downloadable items on the site are integrated here for your convenience.

**Note**  The IPCHelloWorld project is already configured for the Rights Management Services SDK 2.1. For information about how to configure a new project to use the RMS SDK 2.1, see [Configure Visual Studio](how-to-configure-a-visual-studio-project-to-use-the-ad-rms-sdk-2-0.md).

 
The following sections cover the key application steps and understandings needed.

## Loading MSIPC.dll

Before you can call any RMS SDK 2.1 functions, you need to first call [**IpcInitialize**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcinitialize) function to load the MSIPC.dll.



    hr = IpcInitialize();

    if (FAILED(hr))
    {
      wprintf(L"Failed to initialize MSIPC. Are you sure the runtime is installed?\n");
      goto exit;
    }



## Enumerating templates

An RMS template defines the policy used to protect the data, i.e. defines the users that are allowed to access the data and their rights. RMS templates are installed on the RMS server.

The following code snip enumerates the available RMS templates from the default RMS server.



    hr = IpcGetTemplateList(NULL, 0, 0, NULL, NULL, &pcTil);

    if (FAILED(hr))
    {
      DisplayError(L"IpcGetTemplateList failed", hr);
      goto exit;
    }



This call will retrieve RMS templates installed on the default server and load the results in the [**IPC\_TIL**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcinitialize) structure pointed by the *pcTil* variable, then display the templates.



    if (0 == pcTil->cTi)
    {
      wprintf(L"* No templates configured for your RMS server * \n\n");
      wprintf(L"\\------------------------------------------------------\n\n");
      goto exit;
    }

    for (DWORD dw = 0; dw < pcTil->cTi; dw++)
    {
      wprintf(L"Template #%d:\n", dw);
      wprintf(L"    Name:         %s\n", pcTil->aTi[dw].wszName);
      wprintf(L"    Issued by:    %s\n", pcTil->aTi[dw].wszIssuerDisplayName);
      wprintf(L"    Description:  %s\n", pcTil->aTi[dw].wszDescription);
      wprintf(L"\n");
    }



## Serializing a License

Before you can protect any data, you need to serialize a license and get a content key. The content key is used to encrypt the sensitive data. The serialized license is usually attached to the encrypted data and is used by the consumer of the protected data. The consumer will need to call the [**IpcGetKey**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcgetkey) function using the serialized license to get the content key for decrypting the content and for getting the policy associated with the content.

For the sake of simplicity use the first RMS template returned by [**IpcGetTemplateList**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcgettemplatelist) to serialize a license.

Normally, you would use a user interface dialog to allow the user to select the desired template.



    hr = IpcSerializeLicense((LPCVOID)pcTil->aTi[0].wszID, IPC_SL_TEMPLATE_ID,
    0, NULL, &hContentKey, &pSerializedLicense);

    if (FAILED(hr))
    {
      DisplayError(L"IpcSerializeLicense failed", hr);
      goto exit;
    }



After doing this you have the content key, *hContentKey*, and the serialized license, *pSerializedLicense*, that you need to attach to the protected data.

## Protecting Data

Now you are ready to encrypt the sensitive data using the [**IpcEncrypt**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcencrypt) function. First, you need to ask the **IpcEncrypt** function how big the encrypted data is going to be.



    cbText = (DWORD)(sizeof(WCHAR)*(wcslen(wszText)+1));
    hr = IpcEncrypt(hContentKey, 0, TRUE, (PBYTE)wszText, cbText,
    NULL, 0, &cbEncrypted);

    if (FAILED(hr)) {
      DisplayError(L"IpcEncrypt failed", hr);
      goto exit;
    }



Here *wszText* contains the plain text that you are going to protect. The [**IpcEncrypt**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcencrypt) function returns the size of the encrypted data in the *cbEncrypted* parameter.

Now allocate memory for the encrypted data.



    pbEncrypted = (PBYTE)LocalAlloc(LPTR, cbEncrypted);

    if (NULL == pbEncrypted) {
      wprintf(L"Out of memory\n");
      goto exit;
    }


Finally, you can do the actual encryption.



    hr = IpcEncrypt(hContentKey, 0, TRUE, (PBYTE)wszText, cbText,
    pbEncrypted, cbEncrypted, &cbEncrypted);

    if (FAILED(hr)) {
      DisplayError(L"IpcEncrypt failed", hr);
      goto exit;
    }


After this step you have the encrypted data, *pbEncrypted*, and the serialized license, *pSerializedLicense*, that will be used by consumers to decrypt the data.

## Error Handling

Throughout this example application the **DisplayError** function is being used to handle errors.



    void DisplayError(LPCWSTR wszErrorInfo, HRESULT hrError)
    {
        LPCWSTR wszErrorMessageText = NULL;

        if (SUCCEEDED(IpcGetErrorMessageText(hrError, 0, &wszErrorMessageText))) {
          wprintf(L"%s: 0x%08X (%s)\n", wszErrorInfo, hrError, wszErrorMessageText);
        }
        else {
          wprintf(L"%s: 0x%08X\n", wszErrorInfo, hrError);
        }
    }   


The **DisplayError** function uses the [**IpcGetErrorMessageText**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcgeterrormessagetext) function to get the error message from the corresponding error code and prints it to the standard output.

## Cleaning up

Before you are done, you also need to release all the allocated resources.



    if (NULL != pbEncrypted) {
      LocalFree((HLOCAL)pbEncrypted);
    }

    if (NULL != pSerializedLicense) {
      IpcFreeMemory((LPVOID)pSerializedLicense);
    }

    if (NULL != hContentKey) {
      IpcCloseHandle((IPC_HANDLE)hContentKey);
    }

    if (NULL != pcTil) {
      IpcFreeMemory((LPVOID)pcTil);
    }


## Related topics

* [Developer notes](developer-notes.md)
* [Configure Visual Studio](how-to-configure-a-visual-studio-project-to-use-the-ad-rms-sdk-2-0.md)
* [**IpcEncrypt**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcencrypt)
* [**IpcGetErrorMessageText**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcgeterrormessagetext)
* [**IpcGetKey**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcgetkey)
* [**IpcGetTemplateList**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcgettemplatelist)
* [**IpcInitialize**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcinitialize)
* [**IPC\_TIL**](/rights-management/sdk/2.1/api/win/functions#msipc_ipcinitialize)
* [Webinar\_Collateral.zip](https://connect.microsoft.com/site1170/Downloads/DownloadDetails.aspx?DownloadID=42440)
 

 
