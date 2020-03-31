## How to implement a seamless multiple social account linkage in a SPA webapp using B2CAAD and Custom Policies

### Use case
Users should be able to easily and seamlessly be able to connect their social account sign-ins to their existing business user. Giving business users the ability to sign in with their social accounts is considered an advantage over the competitors and there crucial in achieving the overall seamless login experience for the end user, one of the clear goals in the project.

### Problem
The use case represented several problems that needed to be solved. For instance the default custom policies includes a step for manually the IDP. How does the frontend relay the authenticated sesson to the custom policy? How is the the connection maintained and store? How do you check that the access token has been issued with a valid connected business user?

### Solution architecture and chain of events
User authenticates in the web app<br>
User clicks connect button in frontend, selected IDP is set as id_token_hint<br>
Backend API generates a self-issued token for client_assertion<br>
Backend invokes the B2C custom policy with the token and passes the id_token_hint<br>
B2C policy validates the token using the same signing key as the backend<br>
B2C policy signs in the user automatically using the corresponding technical profile<br>
B2C policy goes directly the claims exchange orchestration step based on id_token_hint<br>
B2C policy makes the necessary rest API call to add or append to the list of connected social identites<br>
Backend saves the mapping of which identity that has been connected<br>
Backend sends user back to the url of the social account connection page in the web app<br>
When the user signs in next time, a custom middleware checks which local account the identity was saved for and sets the business user accordingly.

## Web application
The page where the user can connect their social identites will be a list of the social providers available to the user, in this case Microsoft (live.com), Google (google.com) and Facebook (facebook.com). The names in parantheseis are the provider names that will later be stored in identity collection in the user object in B2C, more on that later.

When the user authenticates in the web application an access token is first issued for the web application itself and then other resources, this is how MSAL handles the sign in procedure. Therefore, when the user navigates to the "Connected" page, where the user can manage the social accounts, we already know that the user is authenticated. If using i.e. the B2C custom policy starter pack, the policy will prompt the user to sign in again. After all, technically we are leaving the web application and navigating to an other domain if the B2C policy is just opened "as is". The user context and authentication that MSAL has saved for us is then lost. How can we get around this?

To get around the need of having to sign in in the policy (which for the user would make it seem like they have to sign in twice, since they have already signed in once to the web app) I thought of the idea to still simply to an "ordinary" redirect to the custom policy, similar to how MSAL is doing it, but with a twist. The url is constructed based on parameters like the clicked identity provider name, redirect_url, but it will also consist of a client_assertion token that we can later use with in the custom policy to extract the user's email and perform an auto login of the local account user using a technical profile. But first things first. The web application steps at this point are basically there:

1. The user clicks one of the identity provider buttons.
2. In the Angular event handlar, we make a retrieve a client_assertion token from the user API.
3. We construct the linkage policy url using the the set policy name based on environment (it's a multi environment setup), clientId,   redirect uri (simply the current replyUri of the app + the path to the connections page), client_assertion token and the clicked identity provider as the domain_hint (more on this later).

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_ui.png)

## B2C client assertion token with .Net Core
One of the most critical components in achieving a seamless experience is to auto sign-in the user, but how can we do this? The solution is not as straight forward as it might seem, but I will explain step-by-step below how I solved it.

Instead of opening or redirecting directly to the url of the custom policy, we will instead invoke it from the backend in the user API, the API responbile for user managenent of the portal business users and their settings. What we will do is to issue something called a client assertion token and handle a OIDC hybrid flow in the backend. To achieve this we need to install a NuGet package called Microsoft.AspNetCore.Authentication.AzureADB2C.UI which we be a great help in integration against B2CAAD in .Net Core.

In the Startup class we will configure it like this:

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2cui_startup1.png)

The first part if relatively basic, we start with configiring what schemas, scopes and an event handler for recieved messages, very straight forward.

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2cui_startup2.png)

And further down we have the settings on how the backend handles the redirect of the user navigating to and from the policy. There are a few very important things to note here. The most important setting here is probabably the ClientAssertionType where we basically tell the policy to expect a JWT Bearer token. We take the client assertion token as a parameter when the redirect occurs. (Since the policies are also invoked for inviting and registering new users to the portal, we also have some settings here for the B2C UI customization and registered url).

Now, finally we arrive at our come to the actual API controller method. In the API controller we create a list of claims, the claims we want to include the token and that we will later be able to use within the B2C policy, we give it an expiration of three hours and set the default language.

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_link_invoke.png)

But how do we actually create the token? We do that using a namespace called System.IdentityModel.Tokens.Jwt made for creating, serializing and validating JSON Web Tokens. There is an important security aspect here, which is to sign the token using a secret. The token is self issued and we don't want anyone with programmatic know-how to be able to create token in the same we are and invoke the B2C policy, right? That's where the signing comes in. We use a secret that will later use within B2CAAD too in order to validate the token. Other than that, we are basically simply setting each property of the token with the values. Normally the "aud" claim would be the applicationId and the issuer would be the identity provider, but in this case it's not important as we are self-issuing the token. We return a OkObjectResult with the token back to the web application and the web application will now redirect back to the custom policy.

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_link_invoke2.png)

## Auto-sign in users in B2C custom policies using extracted claims from a JWT Bearer token
It's in the B2C policy the real magic happens. Let's start with looking at the orchestration steps.

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_linkage_custom_policy.png)

The user journey consists of eight steps. Compared to several of the B2C packs out there, some of the most crucal differences here are that:
1. We do not want to prompt the user to sign in
2. We do not want to force the user the select the IDP again, as the user has already made the selection in the web application before navigating to the policy page.
3. Successfully make the connection of connected social identity to the local account B2C user.

So how can we achieve this in a B2C custom policy?

A note of warning: B2C policies are for identity experience framework experts and for highly complex scenarios, like the ones described in this article. It's not for the faint-hearted. There are no really easy way to edit the xml-formatted poilcy. There is an Azure AD B2C extenstion for VS Code https://marketplace.visualstudio.com/items?itemName=AzureADB2CTools.aadb2c that is good, but one should be aware of that the difficulty level is rather high. There are no WYSIWYG editors like there are for xslt editing for instance. And the documentation can be rather heavy but good, like this deep dive into B2C schemas that helped me in solving this http://download.microsoft.com/download/3/6/1/36187D50-A693-4547-848A-176F17AE1213/Deep%20Dive%20on%20Azure%20AD%20B2C%20Custom%20Policies/Azure%20AD%20B2C%20Custom%20Policies%20-%20Deep%20Dive%20on%20Custom%20Policy%20Schema.pdf (can also be found on docs.microsoft.com). There are of course even more higher mountains to climb out there than to work with B2C policies, but be aware that this kind of customization will require some persistance and probably an evening or two, but the amount of customization and results one can achieve are truly spectacular and definitely makes it worth it.

Let's start with looking at the actual linkage policy.

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_linkage_custom_policy2.png)

In there, there are a few things that are important. First we include the authentication protocol name, <Protocol Name="OpenIdConnect" />, since this is the protocol that B2C expects, without this set B2C would throw an error and we would not be able to access the claims of the token in the policy. The second important setting here is that we specify the token type, JWT. Same thing here, B2C will throw an error here as well if this isn't specified and a JWT token is passed. The third important setting here is the CryptographicKeys. Do you remermber the signing of the token? It should be uploaded in the Identity Experience Framework / Policy Keys blade of the B2C AD tenant. When adding a signture key, it should look like this:

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_linkage_custom_policy4.png)

The StorageReferenceId in the custom policy will be the name of the secret you create, plus "B2C_1A" which is a prefix for custom polciies. So if you name it linksecret, the name to enter as the StorageReferenceId will be "B2C_A1_linksecret". This is all that's needed to validate the signature of the token, B2C will handle the rest. The client_secret is a static attribute.

Back to the user journey. Since we now have the decoded JWT token, the email claim is accessible (to see which claims are available and sent between differents steps in user journeys I strongly suggest using the UserJourneyRecorder, debugging without it is can be very daunting) and be used to read the B2C user using AAD-UserReadUsingEmailAddress technical profile if you used the B2C start pack that can be found here. By using the client_assertion approach, we have now managed to retrieve the user within the policy without prompting the user for any action.

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_linkage_custom_policy.png)

In step 2 we verify that the user was found using the email address by veryfing that an objectId exists.

In step 3 the user would normally be forced to select which IDP to connect. But there is way to get around that. In the deep dive documentation I posted previously and on docs.microsoft.com, one can see that B2C supports skipping the ClaimProviderSelection orchestration step if a query parameter is passed named "domain_hint" with a value that matches the Domain attribute of the social identity techical profile that the user is about to connect. For instance, if the you add "&domain_hint=facebook.com" and the value matches the Domain element of the Facebook technical profile, the B2C policy will automatically jump to the ClaimsExchange orchestration step which is where the user is promoted to actually sign in with the social identity account to get a key that can be stored to the userIdentites on the user object.

With the client_assertion JWT token with the authenticated user's email address in conjunction with the domain_hint stemming from the web application and matching the Domain element value of technical profile, we have now reached the desired behaviour where the user does not need to sign in twice and does not need to select the IDP again.

## Adding the social identity to a local account B2C user using Graph
There is one important step left though. We need to actually update the B2C user identity collection to add the connected social account. This can be done in different ways. Finding proved tricky back in the day, but I eventually found a Word file describing the procedure in a Github repository. The userIdentities collection can be updated by a PATCH request to the Graph API and the example described this using an Azure function. Since we already had the infrastructure setup for ingress of traffic thorugh API Management and services hosted in an AKS cluster with internal Azure load balancers, I did see the need to create a separata Azure function only to do a PATCH request to Graph and settled with simply integratiing this into the user API. (Something to consider here is that the application making the requst, in this case the user API, needs the proper Graph access i.e. read and write directory data. A more in depth look can be found here: https://docs.microsoft.com/en-us/graph/permissions-reference.)

The method retrieves the POST body from B2C, deserializes it and retrives the user from Graph based on the objectId of the user.

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_add_identity.png)

And when we have the user, we make an additonal check that the user really is a valid business user (since the session can be up to 24 hours and in that that time a user might have been deactivated or deleted and then we don't want to connect an identity to that user), determines if it's an update or not. If it's an update, i.e. the user has changed one Google identity for an other Google identity, we need to first remove the old entry before adding the new one.

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_add_identity2.png)

Finally, the user is redirected back to the web application. The web application retrieves the list of userIdentites on the local account user display the button as Disconnect if there already is a connected identity with the same provider name or Connect is there's not. 

![image](https://kristofferaxelsson.blob.core.windows.net/$web/github/b2c_ui2.png)

Mission accomplished!


Sources and references:
https://docs.microsoft.com/en-us/azure/active-directory-b2c/direct-signin

https://docs.microsoft.com/en-us/azure/active-directory-b2c/openid-connect-technical-profile

https://download.microsoft.com/download/F/1/4/F1475A9B-5AD3-4B54-B16D-8B34CD416159/Claims-based%20Identity%20Second%20Edition%20device.pdf

https://github.com/Azure-Samples/active-directory-b2c-advanced-policies 

