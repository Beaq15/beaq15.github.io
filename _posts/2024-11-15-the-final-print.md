---
title: "The final print"
categories: [Projects]
image: 
  path: /assets/thefinalprint/thefinalprint.jpg
description: 🏆 Winner of Best Student Game at Dutch Game Awards 2025 - 3D narrative investigation game developed in UE5
---

## 🏆 Award-Winning Project
## 🎉 Winner of Best Student Game 2025 at the Dutch Game Awards!
The Final Print won the prestigious Best Student Game award at the Dutch Game Awards 2025, competing against top student projects from across the Netherlands.

**Jury feedback:** "A beautifully crafted and deep game experience! The team made bold choices for their storyline to create an intriguing and unique narrative experience. The game could benefit from a hint-mechanism and some polish to squash the last bugs, but the judges are fully confident this team can deliver a quality experience."

## 🎓 Development Notes
The Final Print is a narrative investigation game developed over 24 weeks with approximately 18 students as a third-year university project. This project provided extensive experience with complex game systems, UI/UX design, puzzle implementation, and cross-functional collaboration. My contributions spanned gameplay programming, puzzle design, controller support, performance optimization, and technical problem-solving around animation pipeline constraints. The game was successfully published on Steam, demonstrating our team's ability to deliver a polished, commercially viable product.

{% include embed/youtube.html id='1zgv7JVbAiA' %}

## 🎮 Project Overview
You're a journalist investigating a suspicious case during the Harlem riots. Take pictures of evidence, gather information by interrogating people, and put it all together in an article to show people the truth.

> More information on the [Steam page](https://store.steampowered.com/app/3365840/The_Final_Print/?curator_clanid=45433508).

**Tech Stack:** Unreal Engine 5, Blueprints, C++

**Team Size:** ~18 students (24-week project)

**Platform:** PC (Steam)

### Key Features:

- Investigation-based narrative gameplay
- Photography evidence system
- NPC interrogation and dialogue trees
- Multiple puzzle types (lock-picking, code-breaking)
- Full controller and keyboard/mouse support
- Dynamic dialogue system with visual feedback
- Cinematic camera sequences

## 💫 Puzzle Development
I took the lead in developing several key puzzles to enrich the gameplay experience.
### Lock Pick Puzzle
Developed a lock-picking puzzle featuring a pin that jiggles when not correctly placed, providing tactile feedback to guide the player toward the solution.
<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/lockpickjiggle.mp4" type="video/mp4">
</video>

### Numerical Lock Pick
Created a numerical lock puzzle consisting of 4 rotatable numbers that represent a code. The section currently being rotated is highlighted, giving players clear visual feedback about their actions and making the puzzle intuitive to solve.

 <video width="320" height="240" controls>
  <source src="/assets/thefinalprint/numberlock.mp4" type="video/mp4" alt="Camera">
</video>

## 🔧 Core Systems Implementation

### Controller Support
To ensure a versatile gaming experience, I implemented comprehensive controller support for the user interface, allowing seamless navigation using either keyboard/mouse or controller. This support extends from basic menu interactions to complex interfaces including the publish board, pin board, journal, and fast travel menus.

### Main Menu Controller Support:

![menu controller support](/assets/thefinalprint/controllersupport.gif)

### Publish Board Controller Support:
<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/ControllerSupportPublishBoard.mp4" type="video/mp4" alt="Camera">
</video>

### Journal and Fast Travel Controller Support:
<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/ControllerSupportJournal.mp4" type="video/mp4" alt="Camera">
</video>

### Graphics Settings Menu
Implemented a comprehensive graphics settings menu allowing players to:

- Change window mode (Fullscreen, Windowed, Windowed Fullscreen)
- Adjust resolution and resolution scale
- Set overall graphics quality (Low to Max)

All settings are applied and saved after clicking the Apply button, with proper state persistence between sessions.

 <video width="320" height="240" controls>
  <source src="/assets/thefinalprint/GraphicsSettings.mp4" type="video/mp4" alt="Camera">
</video>

## 🎭 Gameplay Features
### Bar Level Interactions
Developed the NPC interaction system for the Bar level with the following features:

- NPCs are highlighted when hovered over with the mouse
- Clicking a highlighted NPC initiates conversation
- After closing conversation, NPCs return to their original positions
- During interaction, player movement, camera rotation, and photography are disabled to maintain immersion

 <video width="320" height="240" controls>
  <source src="/assets/thefinalprint/BarLevel.mp4" type="video/mp4" alt="Camera">
</video>

### Evidence Interaction System
Created an intuitive evidence examination system where:

- Evidence photos in the journal display the item name confirming recognition
- Players can pick up evidence by pressing E
- Evidence can be rotated using the mouse for detailed inspection
- Items can be placed back down with E

 <video width="320" height="240" controls>
  <source src="/assets/thefinalprint/Evidence.mp4" type="video/mp4" alt="Camera">
</video>

### Evidence Highlight System
Implemented a visual highlighting system to guide players toward important evidence without breaking immersion.

<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/EvidenceHighlight.mp4" type="video/mp4" alt="Camera">
</video>

## ⚙️ Technical Challenges & Solutions

### In-Game Animation System
Implemented idle animations in both bar and alley levels, ensuring correct meshes and materials from visual artists were used. For characters like Lindy and Curtis, I created a system to switch between idle and annoyed animations based on specific dialogue lines.

<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/Animations.mp4" type="video/mp4">
</video>

### Technical Challenge:
Typically, animations are implemented using animation blueprints with smooth state blending. However, I encountered a critical issue—Unreal was generating separate skeletons for each animation. Since some NPCs required multiple animations (idle and annoyed states) within the same blueprint, they needed a shared skeleton. Replacing skeletons caused visual glitches, and retargeting wasn't possible because animations were exported as Alembic files with curves instead of bones. This limitation stemmed from animators working with MetaHumans without a rigger on the team.

### Solution Process:
I held multiple discussions with animators and the tech artist, eventually organizing a joint meeting with our animation instructor and another teacher to explore alternatives. We considered waiting for animations to complete before switching (since both shared the same start/end pose), but this risked breaking voice acting synchronization.
The final solution involved switching both the animation asset and skeletal mesh asset mid-dialogue. While this causes a small visual snap, it's subtle enough that most players won't notice unless actively looking for it—an acceptable compromise given the technical constraints.

### Newspaper Throw Mini-Game Optimization

Inherited a basic mini-game implementation from a designer that was causing major performance issues. I completely refactored the system:

### Performance Improvements:

- Made the projectile visible before throwing
- Recalculated trajectory using a more efficient algorithm that didn't impact performance when adding mesh rendering

### Polish & Presentation:

- Implemented timeline-based camera movement using a spline component for smooth, cinematic transitions
- Designed UI widget with fade animations to enhance flow

<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/Newspaper.mp4" type="video/mp4">
</video>

### Performance Demonstration:

<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/NewspaperPerformance.mp4" type="video/mp4">
</video>

## 🎬 Narrative Systems

### Wesley's Phone Call Event

Created a dynamic narrative event where Wesley, the bar owner, receives a phone call during dialogue, making him temporarily unavailable.

### Implementation:

- Wesley's dialogue data table triggers two events: one starts the phone ringing sound, the other stops it
- After dialogue concludes, Curtis is removed from the level
- Curtis reappears when the player re-enters the level

<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/WesleyCall.mp4" type="video/mp4">
</video>

### Dialogue System UX Improvements

Implemented several UX enhancements to the dialogue system:

### Visual Feedback for Navigation Options:

<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/Dialogue1.mp4" type="video/mp4">
</video>

Dialogue options such as "Present Pictures," "Ask Questions," "Leave," and "Back" maintain their color and don't turn gray when clicked, unlike other dialogue choices. This subtle change improves UX by clearly distinguishing navigation options from content choices.

### Dynamic Dialogue Re-Highlighting:

<video width="320" height="240" controls>
  <source src="/assets/thefinalprint/Dialogue2.mp4" type="video/mp4">
</video>

The dialogue line "Has Sharon been here?" turns white again after interacting with the publish board, signaling it may be worth revisiting. In the publish board, a note appears hinting that the player lacks sufficient evidence—implying they've missed a crucial testimony. This system gently guides players to re-explore past conversations without explicit hand-holding.

### Key Testimony System:
Four essential testimonies serve as evidence:

- Alonzo – asking about the note
- Curtis – asking if Sharon has been there before
- Lindy – asking about the receipt
- Wesley – asking if he smoked with Alonzo

Each dialogue line is re-highlighted if missed, helping guide players toward completion without disrupting game flow.

## 📸 Gallery

![](/assets/thefinalprint/thefinalprint7.jpg)
![](/assets/thefinalprint/thefinalprint6.jpg)
![](/assets/thefinalprint/thefinalprint1.jpg)
![](/assets/thefinalprint/thefinalprint2.jpg)
![](/assets/thefinalprint/thefinalprint3.jpg)
![](/assets/thefinalprint/thefinalprint4.jpg)
![](/assets/thefinalprint/thefinalprint5.jpg)


