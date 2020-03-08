## How to avoid invalid Swagger definitions when importing to Azure APIM

## Use case
Any modern API should use auto-genrated code-based API documentation in the Open API format. In c# Swashbuckle is a great and commonly used package to achieve this. Except for genrating a visual and testabe representation of the API operations, it also makes imports to API gateways. In Azure API Management the OpenAPI formatted json file can be imported to create or update an API.

## Problem
However, the import does not always work. After having had conversation with Microsoft support, the cause for why this can sometimes happen is because Open API specification sometimes supports features that API Management doesn't support and sometimes API Management supports features that Open API specicikation does not support. There can be a mismatch. But often this is not the reason why an import fails.

## The error
Let's look at the following error:

Quite cryptic at first glance, but when looking closer at the file you will eventaully see that there duplicate entries of the defintion. This will not be a valid json for import in Azure API Management, rightfully so.

## The solution
Swashbuckle sometimes creates duplicate defintions of types, which is a known issue. THe good thing is that it can easily be mitiated by registrering an operation filter to remove duplicates. I solved it like this:

If you take a look at the json file after the filter has been applied you will now see that there are no duplicate definitions anymore. The API can now be imported to API Managenemtn without any issues!
