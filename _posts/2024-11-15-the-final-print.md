---
title: "The final print"
categories: [Projects]
image: 
  path: /assets/thefinalprint/thefinalprint.png
description: 3D detective game made in UE5
---

## Gameplay
{% include embed/youtube.html id='1zgv7JVbAiA' %}

## General information
The Final Print is a group project I worked on with +-18 other students over the course of 24 weeks.
You're a journalist investigating a suspicious case during the Harlem riots. Take pictures of evidence, gather information by interrogating people, and put it all together in an article to show people the truth.

> More information on the [Steam page](https://store.steampowered.com/app/3365840/The_Final_Print/?curator_clanid=45433508).

## My contribution

### Puzzle development
I took the lead in developing several key puzzles to enrich the gameplay.

#### Lock pick

lock-picking puzzle that features a pin which jiggles if not correctly placed

 <video width="320" height="240" controls>
  <source src="../assets/thefinalprint/lockpickjiggle.mp4" type="video/mp4">
</video>

#### Numerical lock pick

The numerical lock pick consists of 4 numbers that are rotated and that represent a code. The part of the lock pick currently being rotated is highlighted, giving the player a visual representation of what he's doing. 

 <video width="320" height="240" controls>
  <source src="../assets/thefinalprint/numberlock.mp4" type="video/mp4">
</video>

### Controller support

To ensure a versatile gaming experience, I implemented controller support for the user interface, allowing seamless navigation through the game using either a keyboard or a controller. This support extends from basic menu interactions to more complex interfaces such as the publish board, pin board, journal, and fast travel menus.

Controller support for main menu:

![menu controller support](/assets/thefinalprint/controllersupport.gif)

Controller support for the publish board:

<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/ControllerSupportPublishBoard.mp4" type="video/mp4">
</video>

Controller support for journal and fast travel menus:

<video width="320" height="240" controls>
  <source src="../assets/thefinalprint/ControllerSupportJournal.mp4" type="video/mp4">
</video>

### Graphics settings

Within the settings menu, players can:

- change window mode (Fullscreen, Windowed, Windowed Fullscreen)
- adjust resolution and resolution scale
- set overall graphics quality (Low to Max)

All settings are applied and saved after clicking the Apply button.

 <video width="320" height="240" controls>
  <source src="../assets/thefinalprint/GraphicsSettings.mp4" type="video/mp4">
</video>

### Bar level

In the Bar level:

- NPCs are highlighted when hovered over with the mouse.
- clicking a highlighted NPC starts a conversation.
- after closing the conversation, the NPC returns to their original position.
- during interaction, the player cannot move, rotate the camera, or take pictures.

 <video width="320" height="240" controls>
  <source src="../assets/thefinalprint/BarLevel.mp4" type="video/mp4">
</video>

### Evidence interaction

Beneath the evidence photo in the journal, the item name confirms that the object has been recognized by the camera. Players can:
- pick up evidence by pressing E
- rotate it using the mouse
- place it back down with E

 <video width="320" height="240" controls>
  <source src="../assets/thefinalprint/Evidence.mp4" type="video/mp4">
</video>

### In-game animations

I implemented idle animations in both the bar and alley levels, ensuring they use the correct meshes and materials provided by the va's. For characters like Lindy and Curtis, I set up a system to switch between idle and annoyed animations based on specific dialogue lines.

Due to the animations being exported as Alembic files, I couldn’t use animation blueprints for smooth blending, which results in a slight snap when switching between states.

<video width="320" height="240" controls>
  <source src="../assets/thefinalprint/Animations.mp4" type="video/mp4">
</video>

Typically, animations are implemented in the engine using animation blueprints, blending between states for smooth transitions. That was my initial approach as well. However, I quickly ran into an issue—Unreal was generating a new skeleton for each animation. Some NPCs have two distinct animations for conversations (e.g., an idle and an annoyed or shrug animation), and using them within the same blueprint requires a shared skeleton. Unfortunately, replacing the skeleton on an animation caused visual glitches, and retargeting wasn’t possible because the animations were exported using Alembic files with curves instead of bones. This was due to the animators working with MetaHumans and lacking the presence of a rigger in the team.

To fully understand the situation, I had multiple discussions with the animators and the tech artist assisting them. We eventually had a joint meeting with our animation instructor and another teacher to confirm whether there was any alternative approach that would still meet our needs. One option was to wait for an animation to finish and then switch to the next one, since both started and ended in the same pose—but this risked breaking the timing with the voice acting.

In the end, we settled on a workaround: switching both the animation asset and the skeletal mesh asset mid-dialogue. While this causes a small visual snap, it’s subtle enough that most players won’t notice it unless they’re actively looking for it.

### Newspaper Throw Mini Game

Before I took over this task, the mini-game had a basic, partially working version developed by a designer—but it was causing major performance issues. My first step was to make the projectile visible before the newspaper is thrown and recalculating the trajectory in a more efficient way that wouldn't impact performance when adding a mesh.

Next, I implemented a timeline-based camera movement using a spline component attached to the player, creating a smooth, cinematic transition. I also designed a UI widget featuring a fade animation to enhance the overall presentation and flow of the mini-game.

<video width="320" height="240" controls>
  <source src="../assets/thefinalprint/Newspaper.mp4" type="video/mp4">
</video>
<video width="320" height="240" controls>
  <source src="../assets/thefinalprint/NewspaperPerformance.mp4" type="video/mp4">
</video>

### Wesley's call

During the dialogue in the bar, Wesley, the bar owner, receives a phone call and is made unavailable to the player for a short period of time. In Wesley's dialogue data table, there are two dialogue events being triggered—one starts the phone ringing sound, and the other stops it. After the dialogue ends, Curtis is removed from the level entirely. He reappears when the player re-enters the level.

<video width="320" height="240" controls>
  <source src="../assets/thefinalprint/WesleyCall.mp4" type="video/mp4">
</video>

### Dialogue lines

<video width="320" height="240" controls>
  <source src="../assets/thefinalprint/Dialogue1.mp4" type="video/mp4">
</video>

Dialogue options such as "Present Pictures," "Ask Questions," "Leave," and "Back" stay the same color and don't turn gray like the other buttons when clicked upon. This is a small change that improves the overall UX during conversations.

<video width="320" height="240" controls>
  <source src="../assets/thefinalprint/Dialogue2.mp4" type="video/mp4">
</video>

The dialogue line "Has Sharon been here?" turns white again after interacting with the publishboard, signaling that it may be worth revisiting. While in the publishboard, a small note appears hinting that the player doesn't have enough evidence—implying they’ve missed a crucial testimony. This system gently nudges the player to re-explore past conversations and highlights the dialogue choices that are essential for completing the game.

In total, there are four key testimonies that serve as evidence:
- Alonzo – asking about the note
- Curtis – asking if Sharon has been there before
- Lindy – asking about the receipt
- Wesley – asking if he smoked with Alonzo

Each of these lines is highlighted again if the player misses them, helping guide them without ruining the flow of the game.