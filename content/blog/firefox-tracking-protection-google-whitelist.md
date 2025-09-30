+++
title = "Firefox tracking protection Google whitelist"
date = 2025-09-29
updated = 2025-09-29
+++

Some Firefox configuration strings give the incorrect impression that Firefox excludes Google from tracking protection. These strings and their explanation are difficult to find through internet search. This article is that explanation.

Firefox has built in tracking protection with configuration in `about:config`. Some of the config keys contain "whitelist":

- `urlclassifier.features.fingerprinting.annotate.whitelistTables`
- `urlclassifier.features.fingerprinting.whitelistTables`
- `urlclassifier.features.socialtracking.annotate.whitelistTables`
- `urlclassifier.features.socialtracking.whitelistTables`
- `urlclassifier.trackingAnnotationWhitelistTable`
- `urlclassifier.trackingWhitelistTable`

The keys store a list with these elements:

- `mozstd-trackwhite-digest256`
- `google-trackwhite-digest256`

This gives the incorrect impression that Firefox excludes Google from tracking protection.

Firefox loads the tracking protection database at runtime. One part of the database stores which URLs to block. Another part stores a list of exceptions when not to block. This is the "whitelist". It exists because it doesn't make sense to block a tracking domain in all circumstances. You want to block tracking domains embedded on other domains. For example, on `my-personal-blog.com` you want to block `tracker.evilcorp.com`. But on the main `evilcorp.com` it doesn't make sense to block the tracking subdomain. The same company controls both domains and can already track you through the domain main. The tracking subdomain doesn't add any extra tracking capabilities. Additionally, blocking tracking subdomains often breaks the main website.

Firefox used to have only one whitelist called `mozstd-trackwhite`. At some point they added many Google domains to it. This made the it grow in size significantly which triggered a bug in the code that loads it. As a workaround they separated the new domains into another list called `google-trackwhite`.

The list is not made by Google. The list does not favor Google. The list just contains a subset of the changes intended for the main list. In the future Firefox might merge the lists again. The whitelist configuration is not evidence of Firefox favoring Google.

Sources:
- [example of the misconception](https://linuxreviews.org/Mozilla_Is_Rolling_Out_Redirect_Tracking_Protection_In_Firefox_In_A_Somewhat_Concerning_Fashion)
- [Firefox issue for separating the lists](https://bugzilla.mozilla.org/show_bug.cgi?id=1602348)
- [Github comment explaining the list growing too large](https://github.com/mozilla-services/shavar-prod-lists/pull/79#issuecomment-547186843)
