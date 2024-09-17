---
title: "Thoughts on styling libraries"
description: "Choosing between Panda CSS, Tailwind, StyleX, Pigment, Vanilla Extract, Linaria, and CSS Modules."
pubDate: "May 25 2024"
---

I conducted a little research on the advantages and disadvantages of different styling libraries. Currently, my project uses styled-components, but due to lack of support and performance issues (especially on the type checking side), I decided to switch to something else.

I must note that all points are heavily subjective, and you might have completely opposite views. I tried to choose between Panda CSS, Tailwind, StyleX, Pigment, Vanilla Extract, Linaria, and CSS Modules.

There will be no overall winner, primarily because I haven’t decided myself. Most importantly, I want readers to form their own opinions.

1. Type Safety: While I don’t consider this the most crucial point, it can be quite useful in some cases. Potential typos aren’t a significant problem, but tracking changes during refactoring is beneficial. Here, Panda CSS, StyleX, and Vanilla Extract are clear winners.

2. Ease of Setup: I have a large monorepo with many different projects and libraries. Therefore, anything that works with just a Vite plugin (or something similar) is a winner.

   Even better would be solutions without any plugins, though I’m not sure how any zero-runtime library could work without one. Additionally, I value tools that are not only easy to set up initially but also straightforward to work with.

   It’s hard to measure, but, for example, StyleX involves a convoluted process of transforming your JavaScript code into classes. This is great for bundle size, but if something breaks, you rely on fast responses from the maintainer.

   In contrast, with Tailwind, even if the maintainer abandons the library, you end up with a bunch of CSS classes that you can copy to your CSS file and modify as needed. So, for me, the winners here are Tailwind, CSS Modules, and possibly Pigment and Vanilla Extract.

3. Bundle Size: This is pretty self-explanatory. With CSS Modules, your CSS bundle can grow exponentially. A downside of Panda CSS is that you still have all your CSS in your JavaScript code, and you generate CSS for the same code.

   Tailwind is similar but at least uses very short class names. StyleX is similar too. Therefore, in my opinion, both of them are winners.

4. Arbitrary CSS: This refers to how easy it is to use properties and values not defined in your design system or styling library. For example, some values in my project come from the backend, and it’s crucial to be able to use them without much hassle.

   Or, if you want to make some fancy transform somewhere. StyleX, for example, doesn’t support descendant selectors, which can be very limiting.

   With Vanilla Extract, you cannot use arbitrary values with the Sprinkles package, although you can without it, but then what’s the point of Sprinkles? With the other libraries on the list, there are no such problems.

5. Ability to Predictably Override Values: This is an interesting topic. I want to make a “Button” component but be able to change some styling of it, albeit a limited number of things. For example, I might want to specify padding but not height. Why is this important?

   First, you can understand how to change its presentation without looking at the component’s code. Most importantly, you can’t break anything.

   If my “Button” uses padding to define its size but one day I decide that the “height” property works better, I can deprecate one property and introduce another. Consumers can adapt to changes safely. With Tailwind, things don’t work like that.

   To override styles, you need to use the tailwind-merge library, but you still can’t restrict passing arbitrary class names to your component. Therefore, StyleX and Panda CSS are winners here.

Every library has some downsides for me, so choose whatever is closest to your needs. And leave comments. I’m open to dialogue.

[Leave a comment](https://x.com/nuclear0/status/1794337832007528563)
