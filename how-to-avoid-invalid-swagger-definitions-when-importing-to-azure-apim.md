## How to avoid invalid Swagger definitions when importing to Azure APIM

## Use case
Any modern API should use auto-genrated code-based API documentation in the Open API format. In c# Swashbuckle is a great and commonly used package to achieve this. Except for genrating a visual and testabe representation of the API operations, it also makes imports to API gateways. In Azure API Management the OpenAPI formatted json file can be imported to create or update an API.

## Problem
However, the import does not always work. After having had conversation with Microsoft support, the cause for why this can sometimes happen is because Open API specification sometimes supports features that API Management doesn't support and sometimes API Management supports features that Open API specicikation does not support. There can be a mismatch. But often this is not the reason why an import fails.

## The error
Let's look at the following error:

![image](https://stgtoffenr1.blob.core.windows.net/$web/github/apim_json_error.png)

Quite cryptic at first glance, but when looking closer at the file you will eventaully see that there duplicate entries of the defintion. This will not be a valid json for import in Azure API Management, rightfully so.

## The solution
Swashbuckle sometimes creates duplicate defintions of types, which is a known issue. The good thing is that it can easily be mitiated by registrering an operation filter to remove duplicates. I solved it like this:

![image](https://stgtoffenr1.blob.core.windows.net/$web/github/apim_json_error_duplicate_solution.png)

A simple extension filter where I replace the 

After the filter has been applied you will now see that the json does not have duplicate consume definitions anymore. The API can now be imported to API Managenemtn without any issues! No more headaches!
