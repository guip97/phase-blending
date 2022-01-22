# Phase blending technique
## Intro
Several months ago, when I was checking up assets on the Unreal Marketplace, I noticed Kubold’s animation pack on 50% (!) sale, a good deal indeed. Once I got the asset, I started setting up a fairly simple animation blueprint, and it all was quite simple until I faced stop animations. The problem is, that it's not sufficient just to play stop animation after walk cycle, as a "double step issue" will occur:

![double-step-issue](https://raw.githubusercontent.com/guip97/files/main/phase-blending/before.gif?token=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)

That’s weird and completely unacceptable, so I needed to figure out how to synchronize stop animations with the walking cycle. I thought about some kind of special points or events, which would transfer the data of the walking animation to the stop one during the transition. Unreal Engine has a "feature" called sync groups, which, in theory, would solve my problem in just about 1 minute. This is how it works: you put sync markers at frames where poses are almost the same (e.g. when the character touches the ground), the engine will do the rest – seamless transition between sequences. 

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/9.png?token=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)

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/10.png?token=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)

However, the problem hasn't gone anywhere.

After several hours of trying to change syncing groups and roles, I found this post on Unreal forums: https://forums.unrealengine.com/t/unexpected-sync-groups-behavior/245783, which not only highlights the problem I have faced, but also proves that the feature is broken and has unexpected behavior.
At that point, I had to find an alternative solution. And I found one.

## Phase blending

Here’s what I came up with: walking animation has a curve, which represents the time when we want to play our stop animation at; when it’s time to stop, we get a value from that curve, and then simply play stop animation. Sounds pretty simple, but the implementation is, however, not that straightforward. 

![image](https://github.com/guip97/files/blob/main/phase-blending/1.png?token=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)

Alright, but how to create such curve? Where do we get those magic values? Here’s what we need to do: 

- Choose a pose in the walking animation

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/2.png?token=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)

- Open up stop animation and search for a similar pose

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/3.png?token=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)

- Copy the playback position

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/4.png?token=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)

- Go back to the walking animation and paste that value

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/5.png?token=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)

- Adjust stopping animation respectively

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/6.png?token=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)

It’s better to find as much similar poses as possible to achieve high accuracy during transition. 

## Dead zones

In some cases, difference between similar poses can be quite big, here’s an example:

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/7.png?token=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)

Or there’s a huge number of frames in walking animation, for which we just can’t find similar poses in the stop anim. How would we handle such cases? Here’s where **“dead”** zones come in. The idea is that walking animation has a notify state (just a preference, you can also use animation notifies), which has 2 zones: normal (where we can start stop animation) and dead (when we are in such state, walking animation will continue playing until it reaches the nice zone). Obviously, root motion should be **turned on**, otherwise foot sliding is expected.

![image](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/8.png?token=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)

But why are there RightStop and LeftStop? That’s because Kubold’s animation pack provides 2 types of stopping animations: when right foot is up and when the left one is up. 
And here’s a final result:

![gif](https://github.com/guip97/files/blob/0a1a9759fe8b03157554fdfbc95a57a8e60aeda5/phase-blending/after.gif?token=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)

## Limitations

Even though everything is working nicely, this approach is too far from perfect, and here’s why:

1) Sync curves have to be made manually, no automation possible at this point (unless a neural network?), as the definition of “similar” poses is vague
2) Result depends too much on the source animations. In my case walking and stopping animations were done pretty nicely, so it wasn't really a problem to search for similar poses
3) A lot of animation clips. For a standard 8-way locomotion, you’d need 8 (!) stopping animations and moreover, you’d need to set up them correctly. What you’d end up with, most likely is an untestable, janky system, which can hardly be expanded upon.

## Alternatives

A perfect solution would be a fully procedural system, which would create stopping animation in a runtime, with the help of dynamic animation retargeting, IK and Unreal’s feature called “virtual bones”.

## Final thoughts

Phase blending is a powerful tool, which should be used only for simple animation problems, like transition from walk to sprint state. More complex problems may require an advanced solution, most likely including procedural techniques.
