### 01. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

Applications **opt-in** to exposing either a self-selected string, their origin (unspoofable), or both.

### 02. Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes.

### 03. How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

N/A. The application determines what is shared - PII or not. An application interested in leaking PII could do so in multiple other ways much more easily and reliably.

### 04. How do the features in your specification deal with sensitive information?

By making its dissemination opt-in.

### 05. Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No.

### 06. Do the features in your specification expose information about the underlying platform to origins?

No.

### 07. Does this specification allow an origin to send data to the underlying platform?

Yes. That data is not stored long-term. It lives for as long as the application is loaded. It is not processed by the platform in any meaningful way.

### 08. Do features in this specification enable access to device sensors?

No.

### 09. What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

Information which a screen-captured application opts into exposing, is exposed to capturers.

### 10. Do features in this specification enable new script execution/loading mechanisms?

No.

### 11. Do features in this specification allow an origin to access other devices?

No.

### 12. Do features in this specification allow an origin some measure of control over a user agent's native UI?

No.

### 13. What temporary identifiers do the features in this specification create or expose to the web?

Whichever identifiers the application itself assigns and opts into exposing.

### 14. How does this specification distinguish between behavior in first-party and third-party contexts?

It does not.

### 15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

Capture Handle exposure is not disallowed when either the capturing or captured application is in incognito mode, **unless** self-capturing.

### 16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

No. But they can be added.

### 17. Do features in your specification enable origins to downgrade default security protections?

No.

### 18. What should this questionnaire have asked?

I think we're good.
