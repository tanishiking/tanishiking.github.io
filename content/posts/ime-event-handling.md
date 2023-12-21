---
title: 'IME event handling in JavaScript cross-browser (2017)'
description: 'In 2017, when CompositionEvent is supported by many major browsers, I thought it would be easy to implement a small input completion function that supports Japanese input in JavaScript, but the details of the implementation differed depending on the browser...'
publishDate: 2017-12-12
author: 'Rikito Taniguchi'
tags:
  - javascript
---


In 2017, when CompositionEvent is supported by many major browsers, I thought it would be easy to implement a small input completion function that supports Japanese input in JavaScript. However, the details of the implementation differed depending on the browser, and I had to do a little workaround.

## Test environment
- Windows 10 / Microsoft Edge 41.16299.150
- Windows10 / InternetExplorer11 version: 11.64.16299.0
- macOS Sierra 10.12.6 / Google Chrome version: 63.0.3239.84
- macOS Sierra 10.12.6 / Safari version: 11.0.2 
- macOS Sierra 10.12.6 / Opera version: 49.0.2725.64
- macOS Sierra 10.12.6 / Firefox Quantum version: 57.0.1

## KeyboardEvent.keyCode
For IME text editing, the `KeyboardEvent.keyCode` fired by the `keyodown` event is `229`.
This can be used to control the process triggered by the `keydown` event during text editing by IME.


```javascript
editorElem.addEventListener('keydown', function(event) {
  if (event.keyCode ! == 229) {
    // do something ...
  }
});
```

https://www.w3.org/TR/uievents/#determine-keydown-keyup-keyCode

> If an Input Method Editor is processing key input and the event is keydown, return 229.

### Gecko

It seems that text editing by IME doesn't fire keydown/keyup events in Gecko.

https://bugzilla.mozilla.org/show_bug.cgi?id=354358

Update: this issue has already fixed in the end of 2018 🎉.

### deprecated
The `KeyboardEvent.keyCode` seems to be deprecated, and the `keyCode` attribute was mentioned in the `Legacy Key Attributes` section of the W3C, so maybe the `keyCode` is a platform/implementation dependent value that we want to get rid of.

https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/keyCode

## KeyboardEvent.isComposing.
https://developer.mozilla.org/en/docs/Web/API/KeyboardEvent/isComposing

- A readonly attribute that is `true` for text composition by [text composition system](https://www.w3.org/TR/uievents/#text-composition-system) such as IME.
  - `true` if the event is fired between `compositionstart` and `compositionend` as described below.
- Browser compatibility
  - Although MDN's Browser compatibility says that IE and Safari (on desktop) are not supported, we were able to get the value of `KeyboardEvent.isComposing` in Safari 11.0.2.
  - On Microsoft Edge 41 and IE11, `KeyboardEvent.isComposing` seemed to be `undefined`.

I suppose it would be better to use this in the future, but since it is not yet supported in IE and Edge, it is not likely to be used yet...

## CompositionEvent
https://developer.mozilla.org/en-US/docs/Web/API/CompositionEvent

- `compositionstart`: Fired when IME starts editing text.
- `compositionupdate`: Fired when the text being edited by IME is changed.
- `compostionend`: Fires when the IME finishes editing text.

---

- According to MDN's Browser compatibility, Opera is not supported and Safari is `? According to MDN Browser compatibility, Opera is not supported, Safari is `?
  - However, the `data` attribute is not yet implemented.
- To check if `InputEvent` is being edited by IME, instead of using `InputEvent.isComposing`, it would be better to monitor CompostionEvent and manage its state.

```javascript
var isComposing = false;

editorElem.addEventListener('input', function(event) {
  if (!isComposing) {
    // do something ...
  }
});
editorElem.addEventListener('compostionstart', function(event) {
  isComposing = true;
});
editorElem.addEventListener('compostionstart', function(event) {
  isComposing = false;
});
```

Easy peasy.

## CompositionEvent/InputEvent firing order
For more information.

- [https://www.w3.org/TR/input-events-2/#event-order-during-ime-composition]
- [https://www.w3.org/TR/uievents/#events-composition-input-events]


## Start/edit text editing with IME.

| event  | note  |
|---|---|
| compositionstart  | start editing  |
| beforeinput  |   |
| compositionupdate  |   |
|   | DOM is updated here  |
| input  |   |

### When text editing is finished
| event  |  note |
|---|---|
| compositionend  |   |
| beforeinput  |   |
|   | COM is updated here  |
| input  |   |


Since the order of firing of events is like this, `InputEvent` during text editing by IME always fires between `compositionstart` and `compositionend`.

#### I checked it.
I wrote an experimental script and checked it with the browsers in the verification environment

- IE11, Edge41: The `input` event at the end of text editing does not seem to fire?
- Google Chrome, Safari, Opera: The `input` event at the end of text editing fires before the `compositionend`?

<script async src="//jsfiddle.net/tanishiking24/5d20bj5o/9/embed/js,html,result/"></script>.

I'm not sure if it's just my scripting style. Please tell me if it's just my scripting.

## CompositionEvent/KeyboardEvent firing order
For details, see [https://www.w3.org/TR/uievents/#events-composition-key-events]


### Start/stop editing text with IME.

| event  | note  |
|---|---|
| keydown | |
| compositionstart  |  start editing |
| compositionupdate  |   |
| keyup | |

### When text editing is finished

| event  |  note |
|---|---|
| keydown | |
| compositionend  |   |
| input  |   |
| keyup | |

### I've checked it out.

<script async src="//jsfiddle.net/tanishiking24/5d20bj5o/11/embed/js,html,result/"></script>.

- Firefox: As I mentioned at the beginning, IME editing does not fire keydown/keyup events [https://bugzilla.mozilla.org/show_bug.cgi?id=354358].
- Safari: `keydown` fires after `compositionend` when text editing ends (`keyCode` in this case is 229)
- Edge: Neither `keydown` nor `keyup` fires at the end of text editing

Currently, it seems that the order of event firing is quite different depending on the browser implementation... I don't think I'll be using this much, but I'm glad to see that https://www.stum.de/2016/06/24/handling-ime-events-in-javascript/ has a detailed workaround for absorbing differences in event firing order between browsers. I don't think I'll use it much, but the workaround for absorbing differences in event firing order between browsers is detailed in [].

## Reference
- https://www.w3.org/TR/uievents/
- https://www.stum.de/2016/06/24/handling-ime-events-in-javascript/
- https://docs.google.com/document/d/1pGo9hmOXCu71lnpXbqpTQNQP70DU9E-1tNN3yEWg5ig/edit/

## Summary
Currently, it's pretty complicated when it comes to browser compatibility, and even in 2017, Japanese input is still difficult.
