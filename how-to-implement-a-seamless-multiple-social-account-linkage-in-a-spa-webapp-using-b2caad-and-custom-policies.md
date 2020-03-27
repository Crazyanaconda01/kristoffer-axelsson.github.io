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

## Web application part 1
The page where the user can connect their social identites will be a list of the social providers available to the user, in this case Microsoft (live.com), Google (google.com) and Facebook (facebook.com). The names in parantheseis are the provider names that will later be stored in identity collection in the user object in B2C, more on that later.

When the user authenticates in the web application an access token is first issued for the web application itself and then other resources, this is how MSAL handles the sign in procedure. Therefore, when the user navigates to the "Connected" page, where the user can manage the social accounts, we already know that the user is authenticated. If using i.e. the B2C custom policy starter pack, the policy will prompt the user to sign in again. After all, technically we are leaving the web application and navigating to an other domain if the B2C policy is just opened "as is". The user context and authentication that MSAL has saved for us is then lost. How can we get around this?

To get around the need of having to sign in in the policy (which for the user would make it seem like they have to sign in twice, since they have already signed in once to the web app) I thought of the idea to still simply to an "ordinary" redirect to the custom policy, similar to how MSAL is doing it, but with a twist. The url is constructed based on parameters like the clicked identity provider name, redirect_url, but it will also consist of a client_assertion token that we can later use with in the custom policy to extract the user's email and perform an auto login of the local account user using a technical profile. But first things first. The web application steps at this point are basically there:

1. The user clicks one of the identity provider buttons.
2. In the Angular event handlar, we make a retrieve a client_assertion token from the user API.
3. We construct the linkage policy url using the the set policy name based on environment (it's a multi environment setup), clientId,   redirect uri (simply the current replyUri of the app + the path to the connections page), client_assertion token and the clicked identity provider as the domain_hint (more on this later).

## B2C client assertion token with .Net Core
One of the most critical components in achieving a seamless experience is to auto sign-in the user, but how can we do this? The solution is not as straight forward as it might seem, but I will explain step-by-step below how I solved it.

Instead of opening or redirecting directly to the url of the custom policy, we will instead invoke it from the backend in the user API, the API responbile for user managenent of the portal business users and their settings. What we will do is to issue something called a client assertion token and handle a OIDC hybrid flow in the backend. To achieve this we need to install a NuGet package called Microsoft.AspNetCore.Authentication.AzureADB2C.UI which we be a great help in integration against B2CAAD in .Net Core.

In the Startup class we will configure it like this:

The first part if relatively basic, we start with configiring what schemas, scopes and an event handler for recieved messages, very straight forward.

And further down we have the settings on how the backend handles the redirect of the user navigating to and from the policy. There are a few very important things to note here. The most important setting here is probabably the ClientAssertionType where we basically tell the policy to expect a JWT Bearer token. We take the client assertion token as a parameter when the redirect occurs. (Since the policies are also invoked for inviting and registering new users to the portal, we also have some settings here for the B2C UI customization and registered url).

Now, finally we arrive at our come to the actual API controller method. In the API controller we create a list of claims, the claims we want to include the token and that we will later be able to use within the B2C policy, we give it an expiration of three hours and set the default language. 


But how do we actually create the token? We do that using a namespace called System.IdentityModel.Tokens.Jwt made for creating, serializing and validating JSON Web Tokens. There is an important security aspect here, which is to sign the token using a secret. The token is self issued and we don't want anyone with programmatic know-how to be able to create token in the same we are and invoke the B2C policy, right? That's where the signing comes in. We use a secret that will later use within B2CAAD too in order to validate the token. Other than that, we are basically simply setting each property of the token with the values. Normally the "aud" claim would be the applicationId and the issuer would be the identity provider, but in this case it's not important as we are self-issuing the token. We return a OkObjectResult with the token back to the web application and the web application will now redirect back to the custom policy.

## Auto-sign in users in B2C custom policies using extracted claims from a JWT Bearer token
It's in the B2C policy the real magic happens. Let's start with looking at the orchestration steps.

The user journey consists of eight steps. Compared to several of the B2C packs out there, some of the most crucal differences here are that:
1. We do not want to prompt the user to sign in
2. We do not want to force the user the select the IDP again, as the user has already made the selection in the web application before navigating to the policy page.
3. Successfully make the connection of connected social identity to the local account B2C user.

So how can we achieve this in a B2C custom policy?

A note of warning: B2C policies are for identity experience framework experts. There are no really easy way to edit the xml-formatted poilcy. There is an Azure AD B2C extenstion for VS Code https://marketplace.visualstudio.com/items?itemName=AzureADB2CTools.aadb2c that is good, but one should be aware of that the difficulty level is rather high. There are no WYSIWYG editors like there are for xslt editing for instance. And the documentation can be rather heavy but good, like this deep dive into B2C schemas that helped me in solving this http://download.microsoft.com/download/3/6/1/36187D50-A693-4547-848A-176F17AE1213/Deep%20Dive%20on%20Azure%20AD%20B2C%20Custom%20Policies/Azure%20AD%20B2C%20Custom%20Policies%20-%20Deep%20Dive%20on%20Custom%20Policy%20Schema.pdf (can also be found on docs.microsoft.com). There are of course even more higher mountains to climb out there than to work with B2C policies, but be aware that this kind of customization will require some persistance and probably an evening or two, but the results make it worth it. 

The first step I looked at in achieving the optimal outcome for the desired use case was to auto sign in the user. To accomplish this I first needed to extract the email claim from the JWT token. But the token is signed with a key. So how can we decode a signed JWT token in a custom policy? 

It's actually rather easy. We first need to upload the 



To support the use case we will need an API endpoints supporting the page for where the user can connect their social identites. 

