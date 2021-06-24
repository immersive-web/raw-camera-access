# Security and Privacy Questionnaire

This document answers the [W3C Security and Privacy
Questionnaire](https://w3ctag.github.io/security-questionnaire/) for the
WebXR Raw Camera Access Module specification.

01.  What information might this feature expose to Web sites or other parties,
     and for what purposes is that exposure necessary?

This feature exposes camera image obtained from the camera(s) present on the users'
device. The cameras that the data will be coming from will depend on the WebXR
session type. Spec author(s) expect that the cameras will initially only be used
for Augmented Reality sessions on smartphones or tablets.

02.  Do features in your specification expose the minimum amount of information
     necessary to enable their intended uses?

Yes. There are scenarios which require exposing camera images. The WebXR
specification attempts to cater all other scenarios by providing more
privacy-preserving APIs such as Hit Test API.

03.  How do the features in your specification deal with personal information,
     personally-identifiable information (PII), or information derived from
     them?

The features do not attempt to collect PII from the user, although they will
result in exposing the camera image to applications. The camera images may be used
to extract user's PII. The specification mandates that user agents collect
user consent prior to creating WebXR session with the features enabled.

04.  How do the features in your specification deal with sensitive information?

The features do not attempt to collect sensitive information about the user, 
although they will result in exposing the camera image to applications. The camera
images may be used to infer some sensitive information about the user. The
specification mandates that user agents collect user consent prior to creating WebXR 
session with the features enabled.

05.  Do the features in your specification introduce new state for an origin
     that persists across browsing sessions?

Not explicitly. The specification mandates that user agents collect user consent 
prior to creating WebXR session with the features enabled. Depending on the
implementation, this consent may be persisted across browsing sessions.

06.  Do the features in your specification expose information about the
     underlying platform to origins?

Not explicitly. The specification exposes information derived from camera image size.

07.  Does this specification allow an origin to send data to the underlying
     platform?

No.

08.  Do features in this specification enable access to device sensors?

Yes - the features explicitly allow access to device's camera.

09.  What data do the features in this specification expose to an origin? Please
     also document what data is identical to data exposed by other features, in the
     same or different contexts.

The features expose access to camera image and its resolution. The same information
could be exposed by getUserMedia() APIs.

10.  Do features in this specification enable new script execution/loading
     mechanisms?

No.

11.  Do features in this specification allow an origin to access other devices?

No. The WebXR Device API already allows the origin to access XR hardware - this
specification allows the origin to potentially access camera sensors of that
hardware.

12.  Do features in this specification allow an origin some measure of control over
     a user agent's native UI?

No.

13.  What temporary identifiers do the features in this specification create or
     expose to the web?

None.

14.  How does this specification distinguish between behavior in first-party and
     third-party contexts?

No.

15.  How do the features in this specification work in the context of a browser’s
     Private Browsing or Incognito mode?

No difference in behavior.

16.  Does this specification have both "Security Considerations" and "Privacy
     Considerations" sections?

Yes.
    
17.  Do features in your specification enable origins to downgrade default
     security protections?

No.
    
18.  What should this questionnaire have asked?

N/A.
