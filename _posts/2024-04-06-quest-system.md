---
title: "Quest system"
categories: [Projects]
image: 
  path: /assets/questsystem/picture.png
description: Quest system for a rts game / Custom engine
---

## ⭐ Gameplay

<video width="320" height="240" controls>
  <source src="/assets/questsystem/questchain.mp4" type="video/mp4" alt="Quest system">
</video>

## 💎 General information

As part of a group project with seven programmers, I developed a quest system for our RTS game. Since we were building a custom engine alongside the game, I designed my system with serialization in mind.

The system works by displaying the next stage of a quest once all objectives in the current stage were completed. When every stage in a quest was finished, the quest would disappear after a two-second delay, and the following quest would appear. The system was first implemented using ImGui, and later I contributed to integrating it into the game’s UI.

-quest system
<img src = "/assets/questsystem/questevidence.gif" alt="Quest evidence"></img>
-

Realizing that my initial quest system was not scalable, I refactored it by creating dedicated functions for generating and initializing quests, stages, and objectives, resulting in a cleaner and more maintainable structure.

```cpp
 auto killquestentity = CreateQuest("Show how strong you are");
    auto killstageentity = CreateStage(killquestentity, "Kill the entire enemy army");
    auto killObjectiveentity = CreateObjective(killstageentity, "Kill the entire enemy army", Kill, 4);
```

The quest system file primarily handles the creation and organization of all quest-related data.
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

For other users who want to create new quests with this system, the process is straightforward: they simply need to set up the system and then use the provided functions to create quests, stages, and objectives, supplying the necessary data. The first quest must be initialized manually after creation, but from that point on, the system automatically handles the rest.

```cpp
auto& questSystem = bee::Engine.ECS().CreateSystem<bee::QuestSystem>();
auto& quest1 = CreateQuest("Establish Presence");
auto& stage1q1 = CreateStage(quest1, "Construct Buildings");
 CreateObjective(stage1q1, "Gather minerals", bee::ObjectiveType::Collect, 30, "Minerals");

   quest1.Initialize(questSystem.GetUIElementID());
  quest1.StartQuest();
 ```

 ## 💫 Serialization

 While running the game in engine, a game designer can easily create as many quests as they need with the press of a button. The objectives have multiple properties that can be modified beside the name, such as the type, quantity, handle and team.

To ensure proper serialization for enums like "type" and "team", I created functions that convert these enums into strings, allowing them to be serialized into a human-readable format.

In the quest system, I serialize only the variables that are visible in the editor: 

-for quests, I serialize the name and the stages vector 

-for stages I serialize the name and the objectives vector 

-for objectives I serialize the name, type, quantity, handle and team.

For the different types of objectives, i used polymorphic serialization since one of the objectives has additional variables compared to the others. This allows for flexible and efficient serialization of varied objective type.

When a designer uses the "Save" button in the editor, the entire quest system's data is serialized and saved into a JSON file. The file can then be loaded into the game, allowing the quests to be reconstructed with all their details. Finally, in the quest system file, I ensure that the "m_totalQuests" vector, which contains all the quests is serialized. This ensures that all quests created or modified are preserved in the saved data.


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

Before creating a new objective, I added lists for both the objective type and the team. This ensured that the variables were correctly serialized and that the CreateObjective() function could reference the proper values.

```cpp
static bee::ObjectiveType currentType = bee::ObjectiveType::Build;
if (ImGui::BeginCombo("Type:", ObjectiveTypeToString(currentType)))

static Team currentTeam = Team::Ally;
if (ImGui::BeginCombo("Team:", TeamToString(currentTeam)))

if (ImGui::Button("Add Objective"))
    
    CreateObjective(*m_totalQuests[q]->stages[i], "Do not let enemies in the base", currentType, 10,
                    "Warrior", currentTeam);

```

### 💫 Saving the quest system:
![alt text](/assets/questsystem/final.gif)
![alt text](/assets/questsystem/savingfinal.png)

### 💫 Loading the quest system:
![alt text](/assets/questsystem/loadfinal.gif)
![alt text](/assets/questsystem/ya_questname.json.png)