+++
title = "How does Brown University know where you are?"
date = 2020-12-05
tags = ["Brown University", "surveillance"]
+++

In September 2020, Brown University [accused students](https://www.browndailyherald.com/2020/09/28/remote-students-receive-emails-brown-accusing-violating-code-student-conduct) of lying about their location; e.g.:

<a href="https://www.facebook.com/permalink.php?story_fbid=2408913859415693&id=100008913073123"><img src="/blog/brown-location-surveillance/notice.png" style="outline: 1px black solid" height="auto" width="100%" alt="The University has learned that between September 14, 2020 and September 21, 2020 you were allegedly in the Providence area during which time your location of study was listed as remote. This alleged behavior is a violation of the Student Code of Conduct and the COVID-19 Campus Safety Policy. A copy of the Student Commitment to COVID-19 Community Health and Safety Requirements is attached for your review, along with this link to the COVID-19 Campus Safety Policy. failure to abide by these requirements is a violation of the Code of Student Conduct. 
Based on the details of the incident and your student conduct history, the Office of Student Conduct & Community Standards has decided to allow you the opportunity to accept responsibility for the following prohibited conduct without having a COVID-19 Dean's Review Meeting:
• D.8 Failure to Comply
• D.13 Misrepresentation
"></img></a>

**What was Brown's basis for these accusations?**

<!-- more -->

In an [interview with The Brown Daily Herald](https://www.browndailyherald.com/2020/09/28/remote-students-receive-emails-brown-accusing-violating-code-student-conduct), University Spokesperson Brian Clark said the University evaluated a variety indicators, including:
1. indications of building access,
2. indications of accessing private electronic services,
3. indications of accessing secure networks, and
4. reports from community members.

The mechanics of that last indicator are pretty self-explanatory, but what about the others? [Canvas](https://it.brown.edu/services/type/canvas-learning-management-system) *doesn't* [Want To Know Your Location](https://knowyourmeme.com/memes/google-wants-to-know-your-location). **In this post, I'm going to break down the technical mechanisms behind each of these indicators.**

For the most part, I do not have insider knowledge on how Brown reached its decisions. Rather, I'm going to consider each of the indicators Brian Clark named, and describe the technical mechanisms to which Brown *could* have availed itself to generating location data.

## Indications of Building Access
This is an easy one. Brown's buildings are located on Brown's campus. Brown's campus is in Providence. If you are in Brown's buildings, you are on Brown's campus, in Providence. QED.

At Brown, building access is primarily regulated with electronic control systems (namely Software House's [**C•CURE 9000** system](https://www.swhouse.com/products/software_CCURE9000.aspx) system), not mechanical keys.

Encoded on [track 2](https://en.wikipedia.org/wiki/Magnetic_stripe_card#Track_2) of the magnetic stripe on every University ID card is a sixteen digit number that uniquely identifies the card:

<img src="/blog/brown-location-surveillance/brown-id-card-back.jpg" height="auto" width="100%" alt="Image of back of Brown ID card. A tall magnetic stripe runs across the entire width of the card."></img>

Well, that's underwhelming — of *course* you can't *see* it! However, up until 2017 or 2018, this number was also *printed* on the front of ID cards, just above the card-holder's name:

<img src="/blog/brown-location-surveillance/brown-id-card-front.jpg" height="auto" width="100%" alt="Image fo the front of a Brown ID card, displaying the building access code: 6009553660926201"></img>

This pseudo-random identifier (well, its last ten digits) are what uniquely identify you whenever you swipe your Brown ID card *anywhere*. And, if you lose your Brown ID card, this is the *only* thing that's changed when you're issued a replacement. Convenient! In contrast, when you lose your dorm room's mechanical key, Brown must replace (or rather, [rekey](https://en.wikipedia.org/wiki/Rekeying)) the locks.

But, *also* unlike a mechanical key, *every* swipe of a Brown ID card is logged in a central database. **The C•CURE 9000 lets administrators view the complete historical building access history of a person.** Last Spring, Brown used this mechanism to identify and prod students who were slow to evacuate Providence.

## Indicators from Electronic Services
University web services like Canvas *don't* directly ask you for your location. Nonetheless, accessing these services leaves a location finger print: your IP address.

You [IP address](https://en.wikipedia.org/wiki/IP_address) is a number that identifies your device (computer, phone, etc.) for the purposes of networking routing. In principle, nobody but you and your internet provide know the *exact* mapping of your IP address to your physical address.

In practice, IP addresses can be used to *roughly* geolocate a device. Batches of IP addresses are associated *loosely* with geographic areas. Since every web service access leaves an IP address as a trace, there is tremendous incentives for advertisers to be able accurately identify what city or town an IP address is probably associated with.

Brown probably *isn't* analyzing the access logs of its *individual* web services (like Canvas). Rather, they need only to audit the access logs of its three identity access management (IAM) systems:
* The [Google Workplace](https://workspace.google.com/) IAM system is used to control access to your @brown.edu email, and to the various Google Drive services. [**Google Workplace** provides administrators with login audit logs that include users' IP addresses.](https://support.google.com/a/answer/4580120?hl=en)
* The [Shibboleth](https://www.shibboleth.net) IAM system controls access to all *other* Brown web services, such as Canvas. It's what you think of as your "Brown account". Shibboleth is *very* flexible, and can be [configured to log IP addresses](https://wiki.shibboleth.net/confluence/display/IDP30/AuditLoggingConfiguration#AuditLoggingConfiguration-GenericFields).
* [DUO](https://duo.com/) is used to provide two-factor authentication for Shibboleth logins. [It too provides administrators with detailed access logs.](https://help.duo.com/s/article/1023?language=en_US#docs-internal-guid-0aa3b4ce-c686-7559-8814-1377592fce4a:~:text=Access%20Device).

You can partly view your Google Workplace login history for yourself by opening your Brown email and clicking "Details" in the bottom right-hand corner of the page; e.g.:

<img src="/blog/brown-location-surveillance/gmail-account-activity.png" height="auto" width="100%"></img>

With *either* of these access logs in hand, Brown could then turn to any number of geolocation services ([like this one](https://tools.keycdn.com/geo)) to guess your physical location.

## Indicators from Secure Networks
This is another easy one. Brown's WiFI network [blankets Brown's campus](blog/brown-location-surveillance/Campus_Wireless_Coverage_Map_24x36_1.pdf). Brown's campus is in Providence. If you are on Brown's WiFi network, you are on Brown's campus. QED.

What might surprise you is the sheer depth of surveillance that's capable with WiFi alone. This section will *barely* scratch the surface.

### Identification
Brown's WiFi routers each broadcast three [service sets](https://en.wikipedia.org/wiki/Service_set_(802.11_network)):
1. [Brown](https://it.brown.edu/services/type/wireless-network-brown)
2. [eduroam](https://it.brown.edu/services/type/wireless-network-eduroam)
3. [Brown Guest](https://it.brown.edu/services/type/wireless-access-brown-guest)

The Brown and eduroam networks require that you authenticate with your Brown account credentials. Brown University is thus able to identify the owner of any device connected to these networks.

While Brown Guest does *not* require authentication, it still provides mechanisms of identification. Your network devices broadcast a unique identifier called a [**MAC address**](https://en.wikipedia.org/wiki/MAC_address).

<a href="https://drawings.jvns.ca/mac-address/"><img type="image/svg+xml" src="https://drawings.jvns.ca/drawings/mac-address.svg" height="auto" width="100%" alt="Comic by Julia Evans. Text: Every computer on the internet has a network card. When you make HTTP requests with Ethernet/WiFi, every packet gets sent to a MAC address. (&#34;Wait, how do I know someone else on the same network isn't reading all my packets?&#34; &#34;You don't! That's one reason we use HTTPS & secure WiFi networks.&#34;) Your router has a table that maps IP addresses to MAC addresses.  (Read about ARP for more.)
"></img></a>

Brown [maintains databases of the MAC addresses of all connected devices](https://it.brown.edu/computing-policies/network-connection-policy#32:~:text=CIS%20maintains%20a%20database%20of%20unique,a%20computer%20when%20it%20is%20necessary.).

If you have ever connected to an authenticated network, Brown will be able to de-anonymize your connections to Brown Guest — *unless* your device implements [MAC address randomization](https://en.wikipedia.org/wiki/MAC_address#Randomization), which (as the name suggests) randomizes your device's MAC address on a per-network basis.

### Localization
Brown's access points log the MAC addresses of the devices that have connected to them. As of 2015, Brown retrained these logs for at least several years — possibly indefinitely. Since there are so many access points on campus, which access point you are connected to can narrow your location down to a particular room. **Combined, these logs paint a *very* accurate picture of your location on campus at any time.**

You do not need to be *actively* browsing the internet for Brown to know where you are via this mechanism. As you walk through campus, your phone likely *automatically* reconnects to the nearest available access point. If you are within a literal stone's throw of campus, you should assume that CIS can (roughly) identify your location.

## Bonus: Surveillance Cameras

As of February 2020, [*eight hundred* surveillance cameras monitor campus](https://www.browndailyherald.com/2020/02/21/cameras-installed-hegeman-hall/#post-2838338:~:text=The%20University%20uses%20approximately%20800%20cameras,with%20high%20crime%20activity%2C%20Porter%20said.) 24/7. This map documents a *mere seven percent* of Brown University's total camera surveillance capacity:

<iframe width="100%" style="height:40em" src="https://overpass-turbo.eu/map.html?Q=%2F*%0AThis%20query%20looks%20for%20nodes%2C%20ways%20and%20relations%20%0Awith%20the%20given%20key%2Fvalue%20combination.%0AChoose%20your%20region%20and%20hit%20the%20Run%20button%20above!%0A*%2F%0A%5Bout%3Ajson%5D%5Btimeout%3A25%5D%3B%0A%2F%2F%20gather%20results%0A(%0A%20%20%2F%2F%20query%20part%20for%3A%20%E2%80%9Cman_made%3Dsurveillance%E2%80%9D%0A%20%20node%5B%22man_made%22%3D%22surveillance%22%5D(41.82261385454296%2C-71.40611171722412%2C41.83260715798001%2C-71.39570474624634)%3B%0A)%3B%0A%2F%2F%20print%20results%0Aout%20body%3B%0A%3E%3B%0Aout%20skel%20qt%3B%0A%0A%0A%0A%0A%0A%0A%0A%0A%0A%0A%7B%7Bstyle%3A%20%0Anode%0A%7B%0A%20%20icon-image%3A%20url(%22http%3A%2F%2Fjack.wrenn.fyi%2Fblog%2Fbrown-location-surveillance%2Fnoun_Surveillance_63907.svg%22)%3B%0A%20%20icon-width%3A%2020%3B%0A%20%20icon-height%3A%2020%3B%0A%20%20symbol-size%3A%205%3B%0A%20%20color%3Ablack%3B%0A%20%20fill-color%3Ablack%3B%20%7D%0A%20%7D%7D"></iframe>

Brown University has a longstanding policy governing the appropriate uses of its surveillance cameras. **Unfortunately, [this policy is secret](https://www.browndailyherald.com/2008/01/10/surveillance-cameras-on-campus-triple/#post-1679525:~:text=DPS%20would%20not%20release%20the%20University%E2%80%99s%20full%20policy%20on%20the%20surveillance%20camera%20system).**

While Brown probably does not *currently* have the capacity to both broadly and deeply inspect the firehose of data produced by these cameras, expect this to change in the near future. Axis Communications, Brown's primary supplier of surveillance cameras, [now touts cameras that can perform *on-board* facial recognition](https://www.axis.com/customer-story/3767). And Software House, the provider of the C•CURE 9000 access control system, has begun marketing the integration of facial recognition with its access control systems:

<div style='position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;'><iframe src='https://www.youtube.com/embed/bk395D0tPRA' style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" frameborder='0' allowfullscreen></iframe></div>

## How do I find out more?
The [*Family Educational Right and Education Act*](https://en.wikipedia.org/wiki/Family_Educational_Rights_and_Privacy_Act) empowers students to request their education records from their University.

<div style='position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;'><iframe src='https://www.youtube.com/embed/jWzBrC8dVnw' style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" frameborder='0' allowfullscreen></iframe></div>

If you would like to go beyond my blog post and learn *exactly* how Brown University knows your location, [file a FERPA request](https://www.brown.edu/about/administration/registrar/student-information-rightsferpa). Brown University is obligated to respond within 45 days. If you'd like to attempt this, please <a href="mailto:jack@wrenn.fyi">drop me an email</a> — I'd be happy to help you formulate your request.

Additionally, the [*General Data Protection Regulation*](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation) gives EU citizens and residents expansive control over how their personally identifiable information (PII) is used, and the right to request a copy of or the destruction of collected data. I believe these requests should be directed to [Brown's compliance office](https://compliance.brown.edu/).

**If you attempt either of these steps, <a href="mailto:jack@wrenn.fyi">please get in touch</a>!** I'm very curious as to what Brown is *actually* doing.
