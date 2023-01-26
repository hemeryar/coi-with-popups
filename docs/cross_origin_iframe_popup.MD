# Popups from cross-origin iframes in COOP: restrict-properties pages

## Proposal
We want the initial document of popups opened from iframes cross-origin to their top-level document to inherit COOP. If we don't, we cannot support COOP: restrict-properties, for which we cannot simply set noopener.

We do not currently inherit COOP, because it would mix the origin, inherited from the iframe, with the COOP value, inherited from the top-level document. 


![image](../resources/coop_inheritance_issue.jpg)  
_By simply inheriting, we break COOP. In this example, everything lives in the same process, which should never be allowed._

</br>

Our proposal is to supplement the COOP structure with the origin that initially set it. This way we prevent the mix-up of origins and COOP value. The COOP origin is used exclusively by the COOP algorithm.

![image](../resources/coop_inheritance_solution.jpg)  
_By supplementing COOP with the setting origin, we solve the confusion. In this example, we have two distinct BrowsingContext groups._

## Potential alternatives
We could query the opener's top-level document origin directly when doing the first navigation. However this could end up being complicated, if popups are nested for example. This would require a more complex algorithm.

We could use noopener for cross-origin iframes, but that would significantly reduce the usefulness of the feature, because it is expected that a lot, if not the majority of popup flows are actually started from such iframes.

## Additional considerations

### COEP and crossOriginIsolated
### Permissions Policy