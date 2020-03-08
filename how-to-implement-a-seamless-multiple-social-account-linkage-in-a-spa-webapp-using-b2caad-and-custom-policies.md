## How to implement a seamless multiple social account linkage in a SPA webapp using B2CAAD and Custom Policies

### Use case
Users should be able to easily and seamlessly be able to connect their social account sign-ins to their existing business user.

### Problem
The use case represented several problems that needed to be solved. For instance the default custom policies includes a step for manually the IDP. How does the frontend relay the authenticated sesson to the custom policy? How is the the connection maintained and store? How do you check that the access token has been issued with a valid connected business user?

### Solution architecture and chain of events
User authenticates in the web app
User clicks connect button in frontend, selected IDP is set as id_token_hint
Backend API generates a self-issued token for client_assertion
Backend invokes the B2C custom policy with the token and passes the id_token_hint
B2C policy validates the token using the same signing key as the backend
B2C policy signs in the user automatically using the corresponding technical profile
B2C policy goes directly the claims exchange orchestration step based on id_token_hint
B2C policy makes the necessary rest API call to add or append to the list of connected social identites
Backend saves the mapping of which identity that has been connected 
Backend sends user back to the url of the social account connection page in the web app
When the user signs in next time, a custom middleware checks which local account the identity was saved for and sets the business user accordingly.
