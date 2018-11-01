# Highlight API Explained

## Abstract

The Highlight API provides a way for web developers to style one or more Range objects. The styles and semantics of those styles are limited to those that are applicable to the highlight pseudo elements as specified in CSS Pseudo Elements 4. The 'highlights' map exposes a map of names to sequences of HighlightRange objects that specify the desired style to apply to the range. These are grouped as such so that frameworks can have their specific highlights group together in such a way that they are more easily composed (e.g. one framework can do spellcheck highlighting, while another can manage find-on-page). It is possible for Ranges added to the map to overlap - the style properties are applied where last-write-wins and the map is ordered first by alphabetical name, then by sequence order. HighlightRange objects should be removed from the map when web developers wish their contents to no longer be highlighted.
Ranges that are not in the view are ignored.

Questions:

Ranges can be set such that its start and end containers live in a separate document. How to handle this? I don't think we'd want one view to influence the highlighting of, e.g. a same-origin parent frame.

Is the overlap something we can implement? Is there a better way to perform the ordering?

Are there any deal-breakers, implementation-wise given that the Ranges are live ranges

Consider having HighlightsMap take a dictionary with vanilla Range and StylePropertyMap from Houdini CSS Typed OM Level 1 spec instead of extending Range to have a CSSStyleDeclaration. Firefox does not currently support StylePropertyMap. As far as I know, there is no way to create a CSSStyleDeclaration via, e.g., constructor.

Should we have document.createHighlightRange()? I lean towards no.

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
    void Clear();

    iterable<USVString, sequence<HighlightRange>>;
};

partial interface Window {
    readonly attribute HighlightsMap highlights;
};


```
