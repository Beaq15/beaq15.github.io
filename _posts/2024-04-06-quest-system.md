---
title: "Quest system"
categories: [Projects]
image: 
  path: /assets/questsystem/picture.png
description: 2D feature created from scratch while developing a custom engine
---

## 🎓 Development Notes
As part of a group project with seven programmers, I developed a quest system for our RTS game while simultaneously building a custom engine. This project required careful architectural planning to ensure scalability and designer-friendly workflows. The system was designed with serialization as a core feature from the beginning, allowing game designers to create, modify, and save quest chains without programmer intervention.

<video width="320" height="240" controls>
  <source src="/assets/questsystem/questchain.mp4" type="video/mp4" alt="Quest system">
</video>

## 🎮 Project Overview
The quest system manages hierarchical quest chains with multiple stages and objectives. It displays the next stage of a quest once all objectives in the current stage are completed. When every stage in a quest is finished, the quest disappears after a two-second delay and the following quest automatically appears. The system was first prototyped using ImGui and later integrated into the game's custom UI.

<img src = "/assets/questsystem/questevidence.gif" alt="Quest evidence">

**Tech Stack:** C++, custom engine, Cereal (serialization)

**Team Size:** 7 programmers

### Key Features:

- Hierarchical quest structure (Quests → Stages → Objectives)
- Multiple objective types (Kill, Build, Collect, TrainUnits, Location)
- Full serialization support with JSON export/import
- Designer-friendly editor interface
- Polymorphic objective serialization
- Automatic quest progression and UI updates


## 💫 System Architecture
### Refactoring for Scalability
Realizing that my initial quest system was not scalable, I refactored it by creating dedicated functions for generating and initializing quests, stages, and objectives, resulting in a cleaner and more maintainable structure.

```cpp
 auto killquestentity = CreateQuest("Show how strong you are");
    auto killstageentity = CreateStage(killquestentity, "Kill the entire enemy army");
    auto killObjectiveentity = CreateObjective(killstageentity, "Kill the entire enemy army", Kill, 4);
```

### Objective Type System
The quest system file primarily handles the creation and organization of all quest-related data. Each objective type is managed through a switch statement that creates the appropriate specialized objective:

```cpp
switch (objectiveType)
{
    case ObjectiveType::Kill:
    {
        auto newKillObjective = std::make_shared<KillObjective>(objectiveName, objectiveQuantity, lookForHandle);
        stage.objectives.push_back(newKillObjective);
        break;
    }
}
```

### Designer Workflow
For other users who want to create new quests with this system, the process is straightforward: they simply need to set up the system and then use the provided functions to create quests, stages, and objectives, supplying the necessary data. The first quest must be initialized manually after creation, but from that point on, the system automatically handles the rest.

```cpp
auto& questSystem = bee::Engine.ECS().CreateSystem<bee::QuestSystem>();
auto& quest1 = CreateQuest("Establish Presence");
auto& stage1q1 = CreateStage(quest1, "Construct Buildings");
 CreateObjective(stage1q1, "Gather minerals", bee::ObjectiveType::Collect, 30, "Minerals");

   quest1.Initialize(questSystem.GetUIElementID());
  quest1.StartQuest();
 ```

## 🔧 Serialization System
### Runtime Quest Creation
While running the game in the engine, game designers can easily create as many quests as they need with the press of a button. The objectives have multiple properties that can be modified besides the name, such as the type, quantity, handle, and team.

### Enum Serialization
To ensure proper serialization for enums like "type" and "team", I created functions that convert these enums into strings, allowing them to be serialized into a human-readable format.

In the quest system, I serialize only the variables that are visible in the editor:

- Quests: name and stages vector
- Stages: name and objectives vector
- Objectives: name, type, quantity, handle, and team

### Polymorphic Objective Serialization
For the different types of objectives, I used polymorphic serialization since one of the objectives has additional variables compared to the others. This allows for flexible and efficient serialization of varied objective types.

```cpp
CEREAL_REGISTER_TYPE(bee::TrainUnitsObjective)
CEREAL_REGISTER_POLYMORPHIC_RELATION(bee::Objective, bee::TrainUnitsObjective)

const char* QuestSystem::ObjectiveTypeToString(bee::ObjectiveType type)
{
    switch (type)
    {
        case ObjectiveType::Location:
            return "Location";
        case ObjectiveType::Kill:
            return "Kill";
        case ObjectiveType::Build:
            return "Build";
        case ObjectiveType::Collect:
            return "Collect";
        case ObjectiveType::TrainUnits:
            return "TrainUnits";
    }
    return "Unknown";
}
```

### Editor Integration
Before creating a new objective, I added dropdown lists for both the objective type and team. This ensured that the variables were correctly serialized and that the ``CreateObjective()`` function could reference the proper values.

```cpp
static bee::ObjectiveType currentType = bee::ObjectiveType::Build;
if (ImGui::BeginCombo("Type:", ObjectiveTypeToString(currentType)))

static Team currentTeam = Team::Ally;
if (ImGui::BeginCombo("Team:", TeamToString(currentTeam)))

if (ImGui::Button("Add Objective"))
    
    CreateObjective(*m_totalQuests[q]->stages[i], "Do not let enemies in the base", currentType, 10,
                    "Warrior", currentTeam);

```

When a designer uses the "Save" button in the editor, the entire quest system's data is serialized and saved into a JSON file. The file can then be loaded into the game, allowing the quests to be reconstructed with all their details. Finally, in the quest system file, I ensure that the ``m_totalQuests`` vector, which contains all quests, is serialized. This ensures that all quests created or modified are preserved in the saved data.

## 📸 Workflow Demonstration
### Saving the Quest System
![alt text](/assets/questsystem/final.gif)
![alt text](/assets/questsystem/savingfinal.png)

### Loading the Quest System
![alt text](/assets/questsystem/loadfinal.gif)
![alt text](/assets/questsystem/ya_questname.json.png)

### 🎯 Key Achievements

- Designed a scalable, hierarchical quest architecture that supports complex quest chains
- Implemented full serialization support allowing designers to save/load quests without code changes
- Created a designer-friendly interface for runtime quest creation and modification
- Utilized polymorphic serialization to handle varied objective types efficiently
- Successfully integrated the system into both ImGui prototyping and final game UI