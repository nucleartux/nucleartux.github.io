---
title: "How I fixed Cities Skylines 2"
description: "How to write a mod for Cities Skylines 2"
pubDate: "January 18 2025"
---

_Alternative title: How I Spent My Christmas Holidays._

But first, a little prehistory to immerse the reader in the context.

Cities: Skylines 2 is a city-building game and the second installment in the series, promised to be the most advanced simulation game in the genre. However, the release was very rough, with numerous performance issues and bugs. Most importantly, the economy seemed very shallow and meaningless. For example, even if you were not a very good player and lost money, the game would give you a passive income called subsidies. So, it didn’t really matter how you played—you would "win" anyway.

After a few months, the developers released a long-awaited patch called _Economy 2.0_, where they removed such strange mechanics and rebalanced the economy.

I tried playing with the new updates, and everything seemed fine until I noticed something strange. Whenever I raised taxes for commercial or industrial companies—even by just 1%—after some time, they would all move out of town.

I also read all the "Developer Diaries" to better understand the economy in the game. From what I understood, the game has a system where companies occasionally move away from the city. This sounds pretty realistic and interesting. The developers didn’t specify when exactly this should happen, but I guessed it could occur when companies were unhappy with the city’s management or simply unsuccessful.

To understand this system better, I ran my city for a long time under different tax rates and conditions. However, as I mentioned earlier, almost all companies eventually moved away if taxes were changed, and this seemed to happen randomly. So, I started suspecting a bug in the game. Eventually, my curiosity got the better of me, and I decided to fix it myself.

_Disclaimer:_ I don’t have any experience with C#, Unity, or game development in general. So, I had to figure out everything as I went along.

The first step was decompiling, which turned out to be the easiest part of the process. A quick Google search led me to [this article](https://pinter.org/archives/15631), where the author explains the steps in detail. Essentially, you just need to download a program and click one button.

As far as I understand, Cities: Skylines 2 uses ECS (Entity Component System), which makes it easier to understand and modify the game’s systems. After scanning the "Game.Simulation" folder, I found some interesting files, such as `CommercialAISystem` and `CompanyMoveAwaySystem`. It took some time to figure out how everything worked, but eventually, I came across these lines of code:

```csharp
if (companyTotalWorth < m_EconomyParameters.m_CompanyBankruptcyLimit || random.NextInt(100) < companyMoveAwayChance)
{
    m_CommandBuffer.AddComponent(unfilteredChunkIndex, entity, default(MovingAway));
}
```

Clearly, this is the part of the game responsible for companies moving away from the city. Let’s break down the code:

- This condition is self-explanatory:
  `companyTotalWorth < m_EconomyParameters.m_CompanyBankruptcyLimit`

We can also search for m_CompanyBankruptcyLimit in the code and find its default value and some documentation:

```csharp
[Tooltip("All company resources added together and compared to the value. If the combined resources value is lower than the limit value, the company goes bankrupt.\n\nResources include:\n- Money resource\n- Input (resources the company buys to manufacture other resources)\n- Output resources (resources made by the company and stored in the company’s building)\n- Resources that are being moved around by vehicles (not yet arrived to other companies or OC)\n- Service resources (Entertainment, Lodging, other immaterial resources)")]
public int m_CompanyBankruptcyLimit = -5000000;
```

- The second condition is more interesting:
  `random.NextInt(100) < companyMoveAwayChance`.

Examining the `companyMoveAwayChance` code reveals that it depends on tax rates and the number of employees.

In plain terms, the condition says: "If a company is below the bankruptcy threshold OR there’s a chance based on tax rates or lack of employees, then the company moves away." While this logic somewhat makes sense, I suspected the bug was caused by the "OR" in the condition. It felt like it should be an "AND" instead. So, I decided to experiment.

Fortunately, the game has a wiki that describes how to create mods. Even better, there’s a convenient system to install all the required tools (Unity, .NET, etc.). All you need to do is go to a special option in the game menu and click "Install."

Instead of using the provided mod template, I copied and pasted a random mod. After that, I was able to click "Build" in VS Code to use the mod in the game.

To modify the code I needed, I found two popular approaches:

1. Use a library called _Harmony_, which allows you to "patch" any part of the game code (or any .NET code).
2. Use the official method to enable a new ECS system in the game.

I preferred the second approach because it seemed easier and more stable. All I had to do was copy the `CompanyMoveAwaySystem.cs` file from the decompiled source, modify it, and enable my new system in the main mod file like this:

```csharp
World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<CompanyMoveAwaySystem>().Enabled = false;
updateSystem.UpdateAt<ModifiedCompanyMoveAwaySystem>(SystemUpdatePhase.GameSimulation);
```

To be honest, I spent more time trying to understand what `[BurstCompile]` is and why I couldn’t log anything inside that directive.

After clicking "Build," everything worked as expected in the game. However, this was just the start of my modding adventure. After fixing this bug, I discovered that companies had more money than they should, while citizens didn’t have enough. I spent a few weeks debugging and ultimately fixed seven different bugs.

Publishing the mod was also convenient—a one-click action. However, I couldn’t do it in VS Code, so I had to install Visual Studio.

At the time of writing, my mod has over 10,000 active players, and the number is growing. I’ve also found other interesting mechanics that I’m not satisfied with and plan to modify.
