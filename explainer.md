# Highlight API Explained

## Overview

The Highlight API provides a way for web developers to style the text of one or more Range objects. This is useful in a variety of scenarios, including editing frameworks that wish to implement their own selection, find-on-page over virtualized documents, multiple selection to represent online collaboration, and spellchecking frameworks.

Current browsers do not provide this functionality which forces web developers and framework authors to modify the underlying structure of the DOM tree in order to achieve the rendering they desire. This can quickly become complicated in cases where the desired highlight/selection spans elements across multiple subtrees, and requires managing the view in order to remove adjust highlights as they change. The Highlight API aims to provide a programmatic way of adding and removing highlights that do not affect the underlying DOM structure, but instead apply styles based on Range objects.

## Example usage
**HighlightsMap** lives off of the window (view) and maps from a group name string to a sequence of **HighlightRange** objects.
The ```::highlight(ident)``` CSS pseudo-element function will take a CSS identifier that will be matched against the group name strings from the **HighlightsMap** of the associated document. The style properties contained within the rule describes a cascaded highlight style for Ranges that were inserted into the highlights map under a group name equal to the identifier.
**HighlightRange** objects allow for inline-like styles to apply in addition to the cascaded styles, providing both declarative and programmatic interfaces for performing highlight styling. The combination of the styles will be applied to the various runs of text contents of the Range.

```css
::highlight(example-highlight) {
    background-color: yellow;
}
```

```javascript
function highlightElement(element) {
    assert(element instanceof Element);
    if (element.highlighted) {
        return;
    }

    let highlightRange = new HighlightRange();
    highlightRange.setStart(element, 0);
    highlightRange.setEnd(element, element.childNodes.length);
    highlightRange.style.color = "black";

    document.defaultView.highlights.append("example-highlight", highlightRange);
    element.highlighted = true;
}
```

## Application of CSS properties

The **HighlightsMap** is structured as a map so that there is a logical grouping of highlights. This allows web developers and frameworks to have highlights grouped in such a way that they are more easily composed (e.g. one framework can do spellcheck highlighting, while another can manage find-on-page, with yet another performing highlighting for selection).

The grouping of highlights participates in the CSS cascade. The ::highlight pseudo effectively will collect/cascade style properties into a map on any matching element, indexed by group name. Any Range under that group name will subsequently be styled based on the cascade, by applying those properties the individual text runs as described by the Range. In terms of painting, the ::highlight pseudo is treated as a highlight pseudo-element, as described in [CSS Pseudo Elements Level 4](https://drafts.csswg.org/css-pseudo-4/#highlight-pseudos). Only a specific subset of style properties will apply and are limited to those that affect text.

Additionally, HighlightRange objects with 'inline' style can also be added to the map. The values of the HighlightRange objects is not stored as part of the cascade, but instead are read from when determining which properties to apply to a given text run. It is also possible to add entries in the HighlightsMap, without there being a corresponding ::highlight() pseudo element for the associated document. In this case an implicit ::highlight() pseudo will be created and applied to the elements' text runs that are covered by the Range objects in the map.

Ranges added to the map can overlap - in these cases, the cascaded properties are ordered by the cascade and the inline style properties are applied based on the priority of the HighlightRange objects. In the event that Ranges overlap and have the same priority, the implicit ordering of the map is used. The map is ordered first by group name, then by the sequence Ranges in each group. The group names are ordered based on group creation time, so when executing the following, the 'highlight1' will always be applied with higher priority than 'highlight2'.

```javascript
document.defaultView.highlights.append('highlight1', range1);
document.defaultView.highlights.append('highlight2', range2);
document.defaultView.highlights.append('highlight1', range3);
```

## Invalidation
When there is a DOM change within one of the HighlightRange objects in the HighlightsMap, or there is a change to the HighlightRange itself, the user-agent must invalidate the view so and repaint the changed highlights appropriately. If there are DOM/CSS changes that result in a different cascaded highlight map for a given element, and there exists one or more Range objects in the highlights map for the cascaded group names, the layout representation of that element should be notified that the painting of the element might have changed. Similarly, any previously covered or newly covered elements in response to Range updates must be invalidated (if the changed Range had 'inline' style, or the element has the related group name applied based on the cascade).

## Removal of highlights

**HighlightRange** objects should be removed from the map when web developers wish their contents to no longer be highlighted. Ranges are live ranges and so the **HighlightRange** objects in the map must be manually removed if the web developer no longer wants the highlight to be rendered. Ranges that are not in the view are ignored.

Example:
```javascript
highlights.set(groupname, ...highlights.getAll(groupname).
    filter(highlight => highlight !== toBeRemoved))
```

## Open questions

Ranges can be set such that its start and end containers live in a separate document. How to handle this? I don't think we'd want one view to influence the highlighting of, e.g. a same-origin parent frame. Can we scope to the document? should we disallow such start/end setting of the HighlightRange object?

It is a bit unfortunate that we will have a Range-based object that has a CSSStyleDeclaration property hanging off of it. It feels a bit like a layering violation. I had hoped that we could instead have HighlightsMap take a dictionary with vanilla Range and something like StylePropertyMap from (CSS Typed OM Level 1 spec)[https://www.w3.org/TR/css-typed-om-1/#the-stylepropertymap], but just like CSSStyleDeclaration, those are not constructable. Another option is to have a method that takes a dictionary of property (string) value (also string) pairs and hide the implementation details of having a CSSStyleDeclaration under the hood. But that will require an object model that is solely based on strings.

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
    void delete(USVString name);
    void clear();

    iterable<USVString, sequence<HighlightRange>>;
};

partial interface Window {
    readonly attribute HighlightsMap highlights;
};
```
