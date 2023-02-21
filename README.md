# Client Hints

Client Hints are a newly introduced way of exchanging browser information between a the browser and a server. It is meant to replace the user agent over time and intorduce more privacy-friendly handling of client-related information.

## The User Agent

The user agent collects browser information that can potentially used for fingerprinting informaiton to identify users, most notably:

- OS
- OS version
- device model
- browser brand
- browser version

This information is used by tracking tools like Adobe Analytics to derive some of their standard variables, such as device type or browser version.

Chromium browser will only support a reduced version of the the user agent [with upcoming version v101 lasting unti v113 for full migration](https://blog.chromium.org/2021/09/user-agent-reduction-origin-trial-and-dates.html). This will potentially break parts of data collection.

The reduced format looks as follows:

Before changes:

`Mozilla/5.0 (<platform>; <oscpu>) AppleWebKit/537.36 (KHTML, like
Gecko) Chrome/<majorVersion>.<minorVersion>; Safari/537.36`

After changes:

`Mozilla/5.0 (<unifiedPlatform>) AppleWebKit/537.36 (KHTML, like Gecko)
Chrome/<majorVersion>.0.0.0 Safari/537.36`

More about user agent reduction can be found [here](https://www.chromium.org/updates/ua-reduction/).

## Client Hints

## User Agent Client Hints API

The User-Agent Client Hints API extends Client Hints to provide a way of exposing browser and platform information via User-Agent response and request headers, and a JavaScript API.

Client Hints operate as a substitute for the user agent providing browser information in a more privacy-centered way potentially preventing fingerprinting.

The workflow is as folllows:

1. The client dispatches a server request sending low-entropy client hints (see below) with it.
2. In the response headers the server can set `accept-ch` to define high-entropy client hints (see below) it wants to receive with subsequent requests.
3. The browser decideds whether or not it wants to send the requested additional headers with the next request

Client Hints support the following request header information:

- List of browsers and their significant or full version
- is mobile device
- OS name (e.g. Android)
- OS version
- device model (e.g. Pixel 3)
- Underlying bitness architecture (64/86)

Client Hints can be accessed in two ways:

1. **Request Headers** send along when the browser requests a ressource (SEC-CH-UA)
2. **JS API** [navigator.userAgentData](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/userAgentData)

Request headers distinguish between low- and high-entropy client hints. The low entropy hints are those that don't give away much information that might be used to "fingerprint" (identify) a particular user. They are sent by default on every client request, irrespective of the server Accept-CH response header, depending on the permission policy. These hints include: Save-Data, Sec-CH-UA, Sec-CH-UA-Mobile, Sec-CH-UA-Platform.

> The general information about a browser brand and version, mobile device and OS are provided by default with every request

> OS version, bitness, architecture and device model are not available without a request by the server. This also means that they might not be be available with the first tracking event recorded.

The full list of high-entropy client hints:

| Client Hint                 | Description                                                                                                            | Example                                                                               |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Sec-CH-UA-Arch              | Provides the user-agent's underlying CPU architecture, such as ARM or x86.                                             | x86                                                                                   |
| Sec-CH-UA-Bitness           | This is the size in bits of an integer or memory address—typically 64 or 32 bits.                                      | 64                                                                                    |
| Sec-CH-UA-Full-Version-List | Provides the brand and full version information for each brand associated with the browser, in a comma-separated list. | Not A;Brand";v="99.0.0.0", "Chromium";v="98.0.4750.0", "Google Chrome";v="98.0.4750.0 |
| Sec-CH-UA-Model             | Indicates the device model on which the browser is running.                                                            | Pixel 3 L                                                                             |
| Sec-CH-UA-Platform-Version  | Provides the version of the operating system on which the user agent is running.                                       | Windows 10.0.0                                                                        |

The Client Hints can be accessed via JavaScript using `navigator.userAgentData`.

The [`getHighEntropyValues`](https://developer.mozilla.org/en-US/docs/Web/API/NavigatorUAData/getHighEntropyValues) method of the `NavigatorUAData` interface is a Promise that resolves with a dictionary object containing the high entropy values the user-agent returns.

```javascript
navigator.userAgentData
  .getHighEntropyValues([
    "architecture",
    "model",
    "platformVersion",
    "fullVersionList",
  ])
  .then((values) => console.log(values));

//   {
//     "architecture": "x86",
//     "brands": [
//         {
//             "brand": "Not?A_Brand",
//             "version": "8"
//         },
//         {
//             "brand": "Chromium",
//             "version": "108"
//         },
//         {
//             "brand": "Google Chrome",
//             "version": "108"
//         }
//     ],
//     "fullVersionList": [
//         {
//             "brand": "Not?A_Brand",
//             "version": "8.0.0.0"
//         },
//         {
//             "brand": "Chromium",
//             "version": "108.0.5359.125"
//         },
//         {
//             "brand": "Google Chrome",
//             "version": "108.0.5359.125"
//         }
//     ],
//     "mobile": false,
//     "model": "",
//     "platform": "Windows",
//     "platformVersion": "10.0.0"
// }
```

> The documentation states that... <br> > _The values returned by NavigatorUAData.getHighEntropyValues() could potentially reveal more information. These values are therefore retrieved via a Promise, allowing time for the browser to request user permission, or make other checks._ <br> :exclamation: There is no information about how or what a browser would check at the moment.

## Support

> Only Chromium browsers planing on reducing the user agent information. Firexof and Safari do not plan on reducing the user agent or implementing the client hints API.
> Reduction starts with [rollout of Chromium v101](https://blog.chromium.org/2021/09/user-agent-reduction-origin-trial-and-dates.html)

Chromium browsers

- Chrome 89
- Edge
- Chrome Android
- Opera 76
- Opera Andorid 64
- Samsung Internet 15

### Prerequisits

HTTPS - Client Hints are only send over secure connections. This should be the standard for any tracking implementation anyways.

## Impact on Tracking

### **General**

Tag Management Systems often use the navigator.userAgent API in custom code to perform different tasks such as bot detection.

> Tech Hub members will have to review their code base for references to navigator.userAgent and adapt the logic to work with navigator.userAgentData for Chromium browsers if needed. The presumption should be that the code base has to work with low entropy client hints or the reduced user agent.

Since the reduced user agent still contains most of the relevant information I think it is rather unlikely that existing code need any major updates.

The user agent might also still be used by web devs to block requests by bots directly. If no adaptions are made analysts might see a surge in collected data as malicious bots are more likely to trigger tracking requests.
However, the risk is quite low here. There are already better established methods then browser detection by user agent which can easily be spoofed by bots and scrapers. Devs might use DNS forwarding to detect bot traffic and if they don't it is unlikely that the reduction to hte user agent will change this in any way.

It is also unlikely that a malicious bot has an interest in accepting consent which would be necessary for any tracking to run.

### First Hit

The first request to any server will not include high-entropy client hints. For tracking tools that would mean that the first hit does not include these information if there is no previous request to the same server.

This is usually not a problem since the related data is relevant on session scope only. However, for single-page visitors this might pose a problem as there won't be more than the initial tracking request for the first hit. If the tracking library is requested from the same domain though there might already be a `Accept-CH`header though.
This is also only true for high-entropy client hints.

The MDN documentation says about the lifetime of hints:
_A server specifies the client hint headers that it is interested in getting in the Accept-CH response header. The user agent appends the requested client hint headers, or at least the subset that it wants to share with that server, to all subsequent requests in the current browsing session._

In other words, the request for a specific set of hints does not expire until the browser is shut down. This might help if a user browsing the internet visited a page with tracking for the same tool before and high-entropy client hints were already requested. In that case they should theoretically still be available with ones own page.

### Cross Origin Requests

By default, browsers do not send high-entropy client hints to cross-origin domains. So even if a server example.com responds with the `Accept-CH` header any requests to subdomain.example.com won't set the respective client hint headers.

This should only apply to self-hosted tracking code though.

This should **not** apply to CNAME and A Records though. CNAMEs or A Records are usually implemented in a tracking context to trick the browser into believing that a resource is served from a first-party context. However, resolving DNS entries for CNAMEs and A Records do not happen on the HTTP level and shouldn't alter the headers as such. So, if a TMS like Tealium is loaded via a CNAME it should still contain all the request and response headers it would use when directly requesting it via the original domain.

In case there is any additional requests to other domains based on the requested code the high-entropy client hints can not be used though as they would be considered cross-origin (e.g. Tealium Collect is requested via https://tags.tiqcdn.com/ and in turn makes requests to collect.tealiumiq.com). To resolve this the requested site can implement a permissions-policy header:

> Permissions-Policy: ch-ua-bitness=(self "https://collector.example.com")

...or a meta tag in the html:

> \<meta http-equiv="delegate-ch" content="sec-ch-ua-bitness https://collector.example.com;">

Again, if both https://tags.tiqcdn.com/ and https://collector.example.com are requested via the same CNAME or A Record domain this might not event be necessary as client hint permissions are granted to the CNAMEd domain.

## Adobe Stack

[General](https://experienceleague.adobe.com/docs/experience-platform/edge/fundamentals/user-agent-client-hints.html?mt=false#low-entropy)

[Implementation](https://experienceleague.adobe.com/docs/analytics/implementation/vars/config-vars/collecthighentropyuseragenthints.html?lang=en)

Some dimensions / reports in Adobe Analytics and Audience Manager affected if high-entropy client hints are not implemented.

| Adobe Analytics | Adobe Audience Manager |
| --------------- | ---------------------- |
| Browser         | OS Version             |
| Browser Type    | Device Model           |
| OS              | Device Manufacturer    |
| OS Types        | Device Vendor          |
| Mobile          |                        |

Make sure that none of these dimensinons are crucial to your implementation.
Otherwise they can be reactivated by configuring them with the alloy.js WEB SDK.

### How to update?

For the Web SDK this is done natively with the configure command option ["context"](https://experienceleague.adobe.com/docs/experience-platform/edge/fundamentals/configuring-the-sdk.html?lang=en#context).
Most implementation will use some kind of TMS though. In that case the implementation change depends on the TMS.
Launch offers a simple checkbox option with their Adobe Experience Platform Web SDK extension.

If the WebSDK is not in use instead Adobe Analytics needs updating:
The Launch extension supports a simple check box otherwise AppMeasurement can be updated on the s object directly with setting s.collectHighEntropyUserAgentHints = TRUE

> Client Hint config is only available from App Measurement version 2.23.0

Non-Adobe TMS' must manually check how they can set the "context" configuration options for these tools. or set it manually in their code base

Tealiums Adobe Web SDK tag supports a native mapping option for the context object

## Google Stack

GA4 already collects client hints infromation. The payload uses the following parameters:

| Parameter | Example                                                                        | Client Hint               |
| --------- | ------------------------------------------------------------------------------ | ------------------------- |
| uaa       | x86                                                                            | Architecture              |
| uab       | 64                                                                             | Bitness                   |
| uafvl     | Google%20Chrome;107.0.5304.122 Chromium;107.0.5304.122Not%3DA%3FBrand;24.0.0.0 | Full Version List         |
| uamb      | 0                                                                              | is mobile                 |
| uam       | -                                                                              | Model                     |
| uap       | Windows                                                                        | Platofrm                  |
| uapv      | 10.0.0                                                                         | Platform Version          |
| uaw       | 0                                                                              | Windows-on-Windows 64-bit |

I couldn't find any `accept-ch` header for requests to google-analytics.com. I guess this is still based on the regular `user-agent` header information. There is currently no information from Google if GA will use accept-ch to request high-entropy client hints. This should be revisited when Chromium v110 releases.

## Other Tools

Other tools need manual checking in order to determine if they will collect high-entropy client hints by default or if that information is even needed. Vendors should be able to build HTTP headers to request them themselve usually. There is nothing that analytics departments can do in their configuration to set these response headers if the vendor doesn't support such an option (e.g. like Adobe).

# Other browsers

A word on other browsers

## Firefox

[Resource](https://support.mozilla.org/en-US/kb/firefox-protection-against-fingerprinting)

While Firefox has no plans of implementing client hints and deprecating the user agent it implements fingerprinting protection methods that already spoof information from the user agent.

Users can activtae **Enhanced Tracking Protection**, a feature that spoofs the information available in the user agent.

> privacy.resistFingerprinting

The option is not active by default though. Any users activating this option will skew collected data based on the user agent.
