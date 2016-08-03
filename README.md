# Severing Window Proxies - a pre-proposal

NOTE: I don't know that much about what terms to use for this document, which means that I will use the wrong or inappropriate terms frequently. 

The idea is that the concept of the WindowProxy and the concept of a "Physical Window" (e.g. a tab within a web browser, or an iframe), and the history which comes with it, should not be tied together. We would like to provide some way for a website to perform navigation within the current physical window, but with a fresh window proxy which is unattached to any other windows. 

This would allow us to perform the navigation in a different process than the original navigation, as the webpage should be guaranteed to be disconnected from any other webpages. 

I've talked with Luke a bit about this, and we have a proposal for how this could be implemented and how it would interact with history. However, it is almost certainly flawed. We are looking for feedback as to what would need to change before this can be turned into an actual proposal for the HTML spec, or input as to why this is not a good idea :).

## History outside of a window proxy

This would lift the location where back-forward history is recorded into a Physical Window. Effectively, history would be devided into chunks. Each chunk of history would have a WindowProxy which represents it. WindowProxies which no longer have references to them do not need to be kept alive by the history, and can be discarded, and re-created lazily when required. When navigating backwards or forwards between chunks, the WindowProxy used by the Physical Window would be changed to the WindowProxy for that chunk, and the load would occur within that WindowProxy.

## `<a rel=severwindowproxy>`

This would be exposed to web content through a new rel attribute, currently strawman-named severwindowproxy. When a navigation with this attribute is initiated, a new chunk would be created, with an associated fresh WindowProxy, and the navigation would occur within this new WindowProxy. Future navigations would occur as normal, except navigations backwards in history from the new chunk to the previous chunk would cause the active WindowProxy to change, and the navigation to occur in the other window proxy

## `<meta name=severwindowproxy>`

Another option would be to expose this to web content through a meta tag. When a page with this meta tag is loaded, a check would be made to see if the current window proxy is "independent", An independent window proxy is a window proxy which has no other references to it, other than the current window. This could happen if the window proxy was the first page loaded in the tab, or if it was navigated to with severwindowproxy. If it was not, the current load would be canceled, and be replaced with a severwindowproxy load. This will cause the page to be loaded with an independent window proxy. Any navigations away from this page will be treated as severwindowproxy navigations, and cause a new window proxy to be created, to keep this page's window proxy isolated from other pages' window proxies.

I don't know yet whether a reload or history navigation load of a `<meta name=severwindowproxy>` page should sever the window proxy again. This is relevant because the page could open new windows with window.open, which would have a reference to the old window proxy, and when the page is loaded, it would technically not be "independent" anymore, as another document has a reference to it.

My current prototype for `<meta name=freshprocess>` (bug 1277066) uses very similar semantics to this.

## Notes about interactions with iframes

If a severwindowproxy navigation was to occur within an iframe, the iframe's contentWindow would continue to be a reference to the original WindowProxy created when the iframe was created, rather than the WindowProxy which is currently within the iframe, and actually being rendered. This would allow for a page inside an iframe to opt itself into being rendered out of process, and sandboxed from its inclosing webpage, which could be nice.

On the other hand, we could also prevent this, and only allow freshProcess within toplevel windows, if we decided that this behavior was undesirable.

## Open Questions

There are lots of open questions. This is by no means an exhaustive list, it's just what I can remember right now as I'm writing it.

* Could we get away with not severing the window proxy, but simply making everything from the old chunk cross-process to everything from the new chunk (including, for example, pages spawning or spawned by that chunk)? Does that limit what pages can do with one another enough that they could be implemented in a cross process manner? Is that too restrictive? (we probably want to allow them to share indexedDB for example).
* How should this be integrated into the spec. I don't understand the specifics of each component (Browsing Session, Window Proxies, etc.), and therefore don't know exactly how this would integrate to avoid breaking everything.
* What state should persist between Window Proxies which would today be tied to a Window Proxy. The first example I can think of is history, which will need to persist, but we also need to consider other session-bound values.
* How should this be exposed to web developers. I have an idea for how it could be done with `<a>` and `<meta>` tags, which I have outlined above, but this by no means covers all options, or other forms of navigation (such as with `location`).
* Is there any way that this could break existing websites without those websites explicitly opting into the attribute?
* If we allow severwindowproxy navigations within iframes, do we allow the enclosing page to navigate the iframe? The obviously wrong thing will happen if they write `iframe.contentWindow.location = ...`, as the window proxy object is not actually the same window proxy object as is being displayed.
* If window A holds a reference to window B, and window B navigates using severwindowproxy, window A will hold a refernce to window B's old window. If window B then navigates back in history into its original chunk, should it return to using the window proxy which window A has a reference to, or should it use a new window proxy, such that window A no longer holds a reference?
