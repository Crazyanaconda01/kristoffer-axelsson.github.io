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

## B2C policy invocation in .Net Core
One of the most critical components in achieving a seamless experience is to auto sign-in the user, but how can we do this? The solution is not as straight forward as it might seem, but I will explain step-by-step below how I solved it.

Instead of opening or redirecting directly to the url of the custom policy, we will instead invoke it from the backend in the user API, the API responbile for user managenent of the portal business users and their settings. What we will do is to issue something called a client assertion token and handle a OIDC hybrid flow in the backend. To achieve this we need to install a NuGet package called Microsoft.AspNetCore.Authentication.AzureADB2C.UI which we be a great help in integration against B2CAAD in .Net Core.

In the Startup class we will configure it like this:

The first part if relatively basic, we start with configiring what schemas, scopes and an event handler for recieved messages, very straight forward.

And further down we have the settings on how the backend handles the redirect of the user navigating to and from the policy. There are a few very important things to note here. The most important setting here is probabably the ClientAssertionType where we basically tell the policy to expect a JWT Bearer token. We take the client assertion token as a parameter when the redirect occurs. (Since the policies are also invoked for inviting and registering new users to the portal, we also have some settings here for the B2C UI customization and registered url).

Now,Finally we arrive at our come to the actual API controller method. In the API controller we create a list of claims, the claims we want to include the token and that we will later be able to use within the B2C policy, we give it an expiration of three hours and set the default language. 





To support the use case we will need an API endpoints supporting the page for where the user can connect their social identites. 

