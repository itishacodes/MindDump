---
title: "TIL: The CSS field-sizing Cheatcode"
tags:
  - Randoms
  - CSS
  - Frontend
---

Every developer who has built an auto-expanding chat input or textarea box knows the pain of writing height-recalculation code. 

You write a text input, hook up an `onInput` listener, read the scroll height, set the CSS height style, and handle line-break deletes. It is a complete logic overhead just to make an input fit its text content. A complete layout vibe-kill.

I was scrolling through Chrome release notes yesterday, and I found a single line of CSS that replaces all of it:

```css
textarea {
  field-sizing: content;
}
```

Yes, that is literally it.

---

### how it works

By default, form elements (like `input`, `textarea`, and `select`) have fixed intrinsic sizes set by the browser. 

When you set `field-sizing: content`, you instruct the browser to dynamically adjust the input's bounding box size to fit its typed contents automatically:

```html
<!-- No JS listeners, no scrollHeight math -->
<textarea style="field-sizing: content; min-height: 40px; max-height: 200px; resize: none;">
  Type here and watch the box expand...
</textarea>
```

```
was it a complex JavaScript library? no, one line of CSS.
did it save me 50 lines of height-tracking scroll code? Hell yes.
```

The next time you build a messaging input bar, drop `field-sizing: content` in and throw your height recalculation event listeners in the bin. Let me know what browser hacks you find next!
