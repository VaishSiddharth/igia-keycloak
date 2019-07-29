The igia-keycloak component includes three Keycloak extensions to support the SMART-on-FHIR standalone launch sequence. You will need familiarity with Keycloak authentication flows, scopes and clients to configure this component.

## smart-launch-context-authenticator

Custom Keycloak authenticator that will redirect to an external URL for context selection if the app requested scope matches the authenticator supported scope. If an external URL config property is setup in Keycloak in the authenticator execution config, the authenticator will redirect to an external SMART launch context URL. The launch URL must include a query param with placeholder value "{TOKEN}" for Keycloak to insert a generated token into the external request. Example http://localhost:9000/#/patient?token={TOKEN}. The external application is called at that URL with a generated callback URL with auth session details, a FHIR server endpoint, and an access token for the FHIR server endpoint.

After context selection, the external app redirects to the callback URL with a signed token that includes claims for the selected launch context parameters. That endpoint returns control to the authenticator to complete the authentication action. The authenticator will then:
1. Extract the launch app signed JWT token from the app-token query parameter.
2. Verify signature of the client app token provided to the callback URL endpoint using the HMAC secret key entered as the 'External SMART Launch Secret Key' config property in the authenticator execution. If signature is not valid, Keycloak login will fail with an INVALID_CLIENT_SESSION error.
3. Check that each requested launch context from the scope is provided as a claim in the returned JWT. If not, Keycloak login will fail with an INVALID_CLIENT_SESSION error. For example, if launch/patient is a requested scope and the external application does not include a "patient" claim in the returned JWT, then Keycloak login will fail.
4. Write all of the claims in the JWT to the client session notes for later access by the token endpoint. Claims will be added to the client session notes map with key "launch/" + claim name.

If no external URL is configured, the authenticator will use a basic Keycloak template that just allows a patient id to be entered. This is the stable FHIR Patient.id, not the patient's MRN.

## smart-openid-connect protocol

Custom Keycloak login protocol that wraps the core oidc-connect protocol. This protocol exposes a new smart-launch-context endpoint for the external smart launch application to callback. This endpoint verifies that the callback session code is valid, then returns control of flow to the authenticator to complete the login action. The protocol also implements an alternative Token endpoint that includes the additional SMART launch context parameters in the REST access token response. This protocol wraps the existing token endpoint code and inserts the additional response fields before returning to the client.

### /auth/realms/igia/protocol/smart-openid-connect/smart-launch-context

Callback URL for external SMART launch authenticators to redirect browser back to Keycloak after user launch context selection. The request GET parameters include identifiers generated by Keycloak during the initial external launch app redirect in order to identify and resume the authorization session in progress. The app-token parameter is generated by the external launch app and is a JWT token which includes launch context parameters as additional claims. The endpoint identifies the in-progress session and resumes the authentication flow, which returns control to the smart-launch-context-authenticator to verify and handle the additional launch context parameters.

### /auth/realms/igia/protocol/smart-openid-connect/token

This is an OAuth2 token endpoint which accepts a standard token request and returns a token response with additional properties for launch context parameters. For each Keycloak client session note with a key that begins with "launch/", it will add a property to the token response. For example, if the client session notes includes a key "launch/patient" with value "12345", the resulting token endpoint response will include a property "patient" with value "12345". See the example below.
```
{"access_token":"access_token","expires_in":60,"refresh_expires_in":1800,"refresh_token":"refresh_token","token_type":"bearer","id_token":"id_token","scope":"patient/*.read profile openid email launch/patient","patient":"12345"}
```

## smart-oidc-client-session-note-mapper

Custom protocol mapper that adds selected client session note to token claims. The Client session model notes map is used to store launch context information, such as patient id, for the client session. This functions similarly to the existing core oidc-usersessionmodel-note-mapper but allows multiple launch contexts for multiple client applications (ie, if 2 browser windows are open with 2 different SMART apps) instead of one context per user.