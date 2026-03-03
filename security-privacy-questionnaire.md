**Security and Privacy questionnaire**
Answering the questions from https://www.w3.org/TR/security-privacy-questionnaire/

What information does this feature expose, and for what purposes?
- With the reporting functionality offered by Connection Allowlists, the reporting destination gets the URL that any resource request on the site was trying to reach but was disallowed due to Connection Allowlists.

Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?
- Yes Connection Allowlists exposes the minimum amount of information necessary. For example for redirects, they are either all allowed or not and so the report does not carry the redirected URL if it was disallowed.

Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?
- No

How do the features in your specification deal with sensitive information?
- N/A

Does data exposed by your specification carry related but distinct information that may not be obvious to users?
- No

Do the features in your specification introduce state that persists across browsing sessions?
- No

Do the features in your specification expose information about the underlying platform to origins?
- No

Does this specification allow an origin to send data to the underlying platform?
- No

Do features in this specification enable access to device sensors?
- No

Do features in this specification enable new script execution/loading mechanisms?
- No

Do features in this specification allow an origin to access other devices?
- No

Do features in this specification allow an origin some measure of control over a user agent’s native UI?
- No

What temporary identifiers do the features in this specification create or expose to the web?
- None

How does this specification distinguish between behavior in first-party and third-party contexts?
- It does not. Each context can opt-in to Connection Allowlists via a response header.

How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?
- No difference b/w regular and incognito modes

Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
- https://wicg.github.io/connection-allowlists/#security covers "Security Considerations"

Do features in your specification enable origins to downgrade default security protections?
- No

What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?
- If a document with Connection Allowlist is recovered from the BFCache, its connection allowlists will continue to be applied to its network access.
  
What happens when a document that uses your feature gets disconnected?
- A document’s Connection Allowlists are only applicable to that document’s network requests. So when it's disconnected there are no network requests it makes and Connection Allowlists is not applicable.

Does your spec define when and how new kinds of errors should be raised?
- Yes

Does your feature allow sites to learn about the user’s use of assistive technology?
- No

What should this questionnaire have asked?
- N/A

