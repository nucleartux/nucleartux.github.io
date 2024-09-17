---
title: "State management"
description: "Why we probably shouldn't use Redux or any other state management library"
pubDate: "September 17 2024"
---

Often, people who are just starting out with React give in to the temptation to use Redux or some other third-party state management solution.

To be honest, I don’t clearly understand why such libraries are needed at all.

Because of that, I decided to dig into this topic and write an article about why and when we need to use state managers.

_Occasionally in this article, I will mention Redux, as it is the most popular and well-established library. However, in almost every instance, you can substitute it with any other state manager._

I will start with a quote from the renowned [Dan Abramov](https://overreacted.io/): “Don't use Redux until you have problems with vanilla React.”
I would expand this to: “Don't use a state management solution until you have problems with vanilla React.”

I could have ended this article here because it perfectly reflects my approach to choosing libraries to work with.

So, if you are planning to install a library just because a tutorial, influencer, or someone else suggested it—please don’t.
The right approach is to choose a tool that addresses a specific task or pain point.

Now, let’s finally talk about common cases and how to solve them without using any state manager.

1. Network Requests. I believe this is one of the main reasons why people use libraries for state management. However, in my opinion, these are not the most appropriate tools for this task.

   For data that needs to be requested from the network and cached, I suggest using a different term — “network state.” I’m sure most apps don’t need global state, but rather a network cache.

   I think people confuse these two things and misuse them.
   My solution for this problem is libraries like [Tanstack React Query](https://tanstack.com/query/latest). They are perfectly suited for this task because they were specifically created for managing asynchronous state, like network data. They offer a lot of features that no other state management library will ever have. Additionally, you don't need to write your own code to handle network requests, unlike with traditional state management libraries.

2. Global (or shared) state and props drilling are common concerns in React development. But what should we do if we need to manage state across components located in different parts of the component tree, especially when this data isn’t related to a network request?

   For this task, React provides the Context API. React already includes everything needed to work with global state—namely useState and createContext.

   Some argue that using React Context is significantly harder than using state management libraries, but I haven’t found any evidence of this in my own experience. It feels like these ideas are often copied and pasted from one article to another.

   However, the first important decision is whether you even need global state at all. In my experience, it’s a rare case. Some examples that come to mind are themes, user sessions, or maybe some configuration settings.

   Ask yourself: if you have a component on the screen, is there a chance that in the future there might be two such components on the screen, but with different data and state? If yes, then you likely don’t need global state here. And it doesn’t matter whether you're using React Context or another solution like Redux.

   I remember that, at least a few years ago, the Redux community encouraged using Redux for everything—for every component and form—on the grounds that the state would be predictable and could be moved outside React components.

   As a result, there’s a possibility that your app could become slow and difficult to maintain, especially when you need to split one component into two identical components with different data.

3. Excessive rendering. There are some situations when a state management solution could be beneficial because it allows you to change the state at the top of the tree, but re-render only components that are subscribed to those changes.

   However, often we need to take a step back and check if we are able to move the state lower in the tree, to the component that uses this state. This will not only improve the performance of our app but also increase code quality.

   If we can’t move the state lower, React provides a solution for this problem — `memo` and `useMemo`.
   I want to note that this is a preemptive optimization and should be done only if you encounter real performance issues. What's cool is that in React 19, this problem will be [solved automatically](https://react.dev/learn/react-compiler).

4. Centralization of Business Logic. I think this is a valid reason for some applications, but in my observation, such apps are very rare (see below).

   Much more frequently, the situation is the opposite. Usually, in my apps, I create independent components that perform a particular task and do it well. These components don’t need to know anything about the application logic in other components. As a result, you can combine your components like Lego blocks. Can you do this with Redux?

**So, when do we need to use a state management solution?**

I came up with only one valid case: when your app works with a massive amount of locally stored data. And by that, I don't mean just the volume of data itself but the number of variables involved.

Additionally, this data should almost always be global or shared (so you can make a change in one part of the application and see the result in another). The app's logic shouldn’t be trivial, and all the states (variables) might be interconnected in some way.

As a more specific example, Redux allows you to serialize and travel through time between different states, moving back and forth. If you need such features, Redux is an almost perfect solution.

In the end, if you use a library to make your code clearer and more readable (at least in your opinion), that’s a perfect argument in favor of using that library.

Useful links:

- [When should I use Redux?](https://redux.js.org/faq/general#when-should-i-use-redux)
- [You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367)

[Leave a comment](https://x.com/nuclear0/status/1836029153230966890)
