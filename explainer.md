# Highlight API Explained

## Overview

The Highlight API provides a way for web developers to style the text of one or more Range objects. This is useful in a variety of scenarios, including editing frameworks that wish to implement their own selection, find-on-page over virtualized documents, multiple selection to represent online collaboration, and spellchecking frameworks.

Current browsers do not provide this functionality which forces web developers and framework authors to modify the underlying structure of the DOM tree in order to achieve the rendering they desire. This can quickly become complicated in cases where the desired highlight/selection spans elements across multiple subtrees, and requires DOM updates to the view in order to adjust highlights as they change. The Highlight API aims to provide a programmatic way of adding and removing highlights that do not affect the underlying DOM structure, but instead apply styles based on Range objects.

## Example usage
**HighlightsMap** lives off of the window (view) and maps from a group name to a sequence of **HighlightRange** objects.
The ```::highlight(ident)``` CSS pseudo-element function will take a CSS identifier that will be matched against the group name from the **HighlightsMap** of the associated document. The style properties contained within the rule describes a cascaded highlight style for Ranges that were inserted into the highlights map under a group name equal to the identifier.
**HighlightRange** objects allow for inline-like styles to apply in addition to the cascaded styles, providing both declarative and programmatic interfaces for performing highlight styling. The combination of the styles will be applied to the various runs of text contents of the Range.

```css
:root::highlight(example-highlight) {
    background-color: yellow;
    color:blue;
}
```

```javascript
function highlightText(startNode, startOffset, endNode, endOffset) {
    let highlightRange = new HighlightRange();
    highlightRange.setStart(startNode, startOffset);
    highlightRange.setEnd(endNode, endOffset);
    highlightRange.style.color = "black";

    window.highlights.append("example-highlight", highlightRange);
}
```

## Application of CSS properties

The HighlightsMap is structured as a map so that there is a logical grouping of highlights. This allows web developers and frameworks to have highlights grouped in such a way that they are more easily composed (e.g. one framework can do spellcheck highlighting, while another can manage find-on-page, with yet another performing highlighting for selection).

The grouping of highlights participates in the CSS cascade. The ::highlight pseudo will collect/cascade style properties into a map on matching elements, indexed by the group name. Any Range that exists in the highlight map under that group name will be styled based on the computed map of the element corresponding to the portions of the range. Additionally, the HighlightRange object's 'inline' style (i.e. properties set directly on the .style member) will be applied on top of the cascaded values for the group. The values of the HighlightRange objects' style are not stored as part of the cascade, but instead are used when determining which properties to apply to a given inline box.

Following the code example above, if we have the following snippet of HTML:

```html
<p>Some |text|</p>
```

Where 'text' is covered by the ```highlightRange``` (as denoted by the ```|``` characters), there will be two inline boxes created, one for 'Some ' and one for 'text'. The 'text' inline box will first lookup the 'example-highlight' grouping, and apply ```background-color:yellow``` and ```color:blue```, based on the map that was cascaded onto the ```<p>``` element. Subsequently, ```color:black``` will be applied based on the style of the HighlightRange itself.

HighlightRanges added to the map can overlap --- in these cases, the properties are computed by applying the styles of the applicable Ranges in ascending priority order, where the last write of a given property wins. In the event that Ranges overlap and have the same priority, the timestamp of when the HighlightRange was added to the map is used.

It is also possible to add entries in the HighlightsMap, without there being a corresponding ::highlight() pseudo element for the associated document. In this case an there are no cascaded properties to apply when creating inline boxes --- only the inline properties directly set on the HighlightRange objects will apply (and if there are none, there will be no impact on inline boxes).

In terms of painting, the ::highlight pseudo is treated as a highlight pseudo-element, as described in [CSS Pseudo Elements Level 4](https://drafts.csswg.org/css-pseudo-4/#highlight-pseudos). Only a specific subset of style properties will apply and are limited to those that affect text.

## Invalidation
HighlightRanges are live ranges - DOM changes within one of the HighlightRange objects will result in the new contents being highlighted. Changes to the boundary points of HighlightRanges in the  HightlightsMap will result in the user-agent invalidating the view and repainting the changed highlights appropriately. If there are DOM/CSS changes that result in a different cascaded highlight map for a given element, and there exists one or more Range objects in the highlights map for the cascaded group names, the layout representation of that element should be notified that the painting of the element might have changed. HighlightRanges that are positioned inside of documents that are not in the view are ignored.

## Removal of highlights

Because HighlightRange objects are live ranges, they must be modified when web developers wish their contents to no longer be highlighted. This can be achieved by removing the HighlightRange from the map, via ```remove()``` (note that ```remove()``` is overloaded so that passing a string will remove an entire group).

## Open questions

Ranges can be set such that they live entirely in a separate document. Should these cases be allowed, and the highlight simply not apply? Or do we want to detect this up front and throw?

Should we allow empty Ranges to be rendered like a caret, or are they not rendered at all.

How is generated content handled, since they can't be directly referenced by Range? (e.g. if I have a highlight with start at the beginning of an element, but the element has ::before content)

How should 'inherit' values be treated?

Can a HighlightRange participate in multiple groups? If so how do we order the cascaded properties for each group?

## Webidl

```webidl
[Constructor, Exposed=(Window)]
interface HighlightRange : Range {
    [SameObject, PutForwards=cssText] readonly attribute CSSStyleDeclaration style;
    attribute unsigned long priority;
};

interface HighlightsMap {
    HighlightRange? get(USVString name);
    sequence<HighlightRange> getAll(USVString name);
    boolean has(USVString name);

    void set(USVString name, HighlightRange... highlights);
    void append(USVString name, HighlightRange... highlights);
    void insert(USVString name, unsigned long index, HighlightRange... highlights);
    void delete(USVString name);
    void delete(HighlightRange range);
    void clear();

    iterable<USVString, sequence<HighlightRange>>;
};

partial interface Window {
    readonly attribute HighlightsMap highlights;
};
```
