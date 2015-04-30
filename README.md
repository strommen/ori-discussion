Hi Ori,

I read your article about how Wikipedia has transitioned to HHVM.  Great writeup!  One of the details that stood out to me was that logged-in users **never** get server the cached version.  It's fantastic that you were able to speed it up so much by switching to HHVM, but I decided it would be interesting to see whether Wikipedia could make pages server-cacheable for everybody.

Making HTML cacheable on the server is a problem that I've thought a lot about (see the OSS project I recently started, [TwoStage](http://www.twostage.io)).  Serving the HTML from a caching proxy like Varnish or nginx is obviously super fast.  Also, by decoupling app server load from traffic, it becomes feasible to preload pages with something like [InstantClick](http://instantclick.io) (which begins loading a page when you mouse over the link).

Unfortunately, after digging into the Wikimedia source code, I found that server-caching authenticated pages is impossible for the general case due to skinning (you probably knew this already).  But I decided to pretend for a moment that all skins render the HTML exactly like Vector.  After all, in theory it should be possible to modify other skins to do exactly this, and implement all their customization in JavaScript and CSS. 

So, with that (admittedly giant) assumption, here are the differences I found when viewing an article:

1. The CSS links in the `<head>` (from OutputPage:2677).
1. The inline JavaScript code in the `<head>` for mw.config, mw.loader.implement, and mw.loader.load (from OutputPage:2680).  
1. The anonymous version has a `<noscript>` element after the article (I was unable to determine where this came from).
1. The *Personal Tools* area is different (see SkinTemplate:545).
1. "Add to Watchlist" shows up when logged in (see SkinTemplate:1003).
1. The inline scripts (for mw.loader.load) at the end of the `<body>` (see OutputPage:3090).

I think we can resolve all of these differences so that wikimedia serves the same content to all users, whether they're logged in or not.  I was hoping to show you a code diff that would accomplish this, but frankly I ran out of time.  So instead I'll just explain to you how I think this can be done:

For #1, rather than rendering user-specific CSS links into the head, we could render a static link to a dynamically-generated user-specific CSS file (e.g. `<link rel="stylesheet" href="/UserCss.php" />`).  This PHP handler would return all the CSS for that user, concatenated.  For anonymous users, this response would be cached by Squid/Varnish.

For #2 and #6, the same approach could be applied to the user-specific JavaScript.  Having three separate requests to the app server would probably slow down the first page load for authenticated users, but subsequent navigation could be lightning quick.

For #4 and #5, SkinTemplate could be modified to always include the content for both anonymous and logged-in users.  Then, the invalid links can be hidden via CSS (this logic would have to be included in the user-specific CSS generator). 
  
I'm sure that I'm missing some other use cases of how wikimedia renders differently for authenticated vs. anonymous, but I think this might be a good start. 

Thanks
Joe
