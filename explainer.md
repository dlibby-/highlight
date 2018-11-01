# Highlight API Explained

## Overview

The Highlight API provides a way for web developers to style one or more Range objects. This may be useful in a variety of scenarios, including editing frameworks that wish to implement their own selection, website implemented find-on-page, multiple selection to represent online collaboration, and spellchecking frameworks.

Current browsers do not provide this functionality which forces web developers and framework authors to modify the underlying structure of the DOM tree in order to achieve the rendering they desire. This can quickly become complicated in cases where the desired highlight/selection spans elements across subtrees, and requires managing the view in order to remove adjust highlights as they change. The highlight API aims to provide a programmatic way of adding and removing highlights that don't affect the underlying DOM structure, but instead apply styles on Range objects.

## Example usage
**HighlightRange** is a Range object that has a style attached to it.
**HighlightsMap** lives off of the window (view) and maps from string to a sequence of **HighlightRange** objects. Any **HighlightRange** object that exists in the map will have its style applied to the text contents of the Range.

```javascript
function highlightElement(element) {
    assert(element instanceof Element);
    if (element.highlighted) {
        return;
    }

    let highlightRange = new HighlightRange();
    highlightRange.setStart(element, 0);
    highlightRange.setEnd(element, element.childNodes.length);

    window.highlights.append(HIGHLIGHT_GROUP, highlightRange);
    element.highlighted = true;
}
```

## Details

The style of the **HighlightRange** objects will be treated as a highlight pseudo-element, as described in [CSS Pseudo Elements Level 4](https://drafts.csswg.org/css-pseudo-4/#highlight-pseudos). This means that the styles that will actually apply are limited to those that style text.

The **HighlightsMap** is structured as a map so that there is a logical grouping of highlights. This will allow frameworks to have highlights grouped in such a way that they are more easily composed (e.g. one framework can do spellcheck highlighting, while another can manage find-on-page, with yet another performing highlighting for selection).

It is possible for Ranges added to the map to overlap - the style properties are applied where last-write-wins and the map is ordered first by alphabetical name, then by sequence order. **HighlightRange** objects should be removed from the map when web developers wish their contents to no longer be highlighted.
Ranges that are not in the view are ignored.

Ranges are live ranges and so the **HighlightRange** objects in the map must be manually removed if the web developer no longer wants the highlight to be rendered.

## Open questions

Ranges can be set such that its start and end containers live in a separate document. How to handle this? I don't think we'd want one view to influence the highlighting of, e.g. a same-origin parent frame. Can we scope to the document? should we disallow such start/end setting of the HighlightRange object?

Is the overlap something we can implement? Is there a better way to perform the ordering?

It is a bit unfortunate that we will have a Range-based object that has a CSSStyleDeclaration property hanging off of it. It feels a bit like a layering violation. I had hope that we could instead have HighlightsMap take a dictionary with vanilla Range and something like StylePropertyMap from (CSS Typed OM Level 1 spec)[https://www.w3.org/TR/css-typed-om-1/#the-stylepropertymap], but just like CSSStyleDeclaration, those are not constructable. Another option is to have a method that takes a dictionary of property (string) value (also string) pairs and hid the implementation details of having a CSSStyleDeclaration under the hood.

Ergonomics of removing a single highlight don't seem ideal, but may be acceptable - you can highlights.set(groupname, ...highlights.getAll(groupname).filter(highlight => highlight !== toBeRemoved))

## Webidl

```webidl
[Constructor, Exposed=(Window)]
interface HighlightRange : Range {
    [SameObject, PutForwards=cssText] readonly attribute CSSStyleDeclaration style;
}

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
