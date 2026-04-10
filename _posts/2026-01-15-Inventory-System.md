---
title: "Inventory System"
categories: [Projects]
image: 
  path: /assets/inventorysystem/main.png
description: Custom inventory system built from scratch in C++ with drag-and-drop, stacking, and serialization
---

![alt text](/assets/inventorysystem/main2.png)

## 🎓 Development Notes
This inventory system was built entirely from scratch in C++ using a custom engine template from university, with SDL2 for rendering and ImGui for the item creation tool. The project demonstrates low-level game systems programming, including UI coordinate transformations, entity lifecycle management in an ECS architecture, and performance optimization through caching. Rather than relying on engine features like Unity or Unreal provide, I implemented every aspect myself—from pixel-perfect slot positioning to multi-digit quantity displays using individual sprite entities.

## 🎮 Project Overview
An inventory system combining features from games like Minecraft, 7 Days to Die, and Dying Light. The system supports stackable items, drag-and-drop manipulation, context menus, and complete save/load functionality—all built without relying on external game engine UI systems.

A key design goal was making the system fully designer-friendly. Any aspect of the inventory — the number of slot groups, their dimensions and spacing, icon sprite sheets, popup layout, and button positioning — can be configured at runtime through ImGui without touching a single line of C++. Swapping in a completely different visual style is as simple as changing a texture path and hitting reload, making it straightforward to create anything from a minimal hotbar to a complex multi-grid inventory.

**Tech Stack:** C++, SDL2, OpenGL, ImGui, nlohmann/json, Custom ECS

**Key Features:**
- Fully configurable layout — slot groups, grid dimensions, spacing, and icon paths set through ImGui at runtime
- Swap inventory visuals entirely by changing texture paths, no code required
- Custom item creation tool for designers (no code required)
- Drag-and-drop with left-click (move all) and right-click (split stack) support
- Multi-digit quantity displays using individual sprite entities
- Item popup system (Use/Equip/Drop functionality)
- Full JSON serialization for save/load
- Grid-based layout (10x4 = 40 slots)
- Stackable items with configurable max stack sizes
- Pixel-perfect slot positioning with coordinate transformation

## 💫 System Architecture

### ECS Approach
The system uses an Entity Component System architecture where items aren't just data structures—they're actual entities in the world. This provides several benefits:

**Items as Entities:**

- Each item is an entity with Transform, Sprite, and ItemData components
- When dropped, items already exist in the world (no conversion needed)
- Picking up items simply moves entities from world to inventory
- Consistent behavior whether items are held or dropped

### State Management:
Items exist in different states:

- **In the world** - Normal entity with physics and collision
- **In inventory** - Visual only, no physics
- **Being dragged** - Following mouse, semi-transparent

This is managed through a simple state machine with component enable/disable logic.

### Core Data Structures

**Inventory Class:**

```cpp
class Inventory {
private:
    std::vector<Slot> m_slots;
    bool m_isVisible = false;
    int m_capacity;
    
public:
    bool AddItem(Entity* item);
    bool RemoveItem(int slotIndex);
    void ToggleUI();
    void Update(float deltaTime);
    void MoveItem(int sourceSlot, int destSlot);
};
```

**Slot Class:**
```cpp
class Slot {
private:
    Item* m_item;
    int m_quantity;
public:
    bool IsEmpty() const { return m_item == nullptr; }
    bool AddQuantity(int amount);
    bool RemoveQuantity(int amount);
    void Clear();
};
```

**Item Class:**
```cpp
class Item {
private:
    std::string m_name;
    std::string m_description; 
    std::string m_texturePath;
    ItemType m_type;
    bool m_stackable;
    int m_maxStackSize;
    int m_spriteX;
    int m_spriteY;
}
```
## 🔧 UI Implementation
### InventoryLayoutConfig
Before any slots are positioned or rendered, all layout parameters flow through a single `InventoryLayoutConfig` struct in the `graphics2d` namespace. Every rendering and hit-detection calculation references this struct directly, so changing a value — slot count, spacing, popup size, button offset — propagates instantly through the entire system without touching any other code.

The struct holds a vector of `SlotGroup` entries, meaning the inventory isn't limited to a fixed hotbar + grid layout. Any number of groups can be added, each with its own row/column count, size, and spacing. The default values are calibrated to match the initial design, but everything can be overridden at runtime through the ImGui panel.

![](/assets/inventorysystem/graphics2d.png)
*The graphics2d namespace — shared state and utilities accessible by any file without coupling to a specific game class*
```cpp
struct SlotGroup
{
    std::string name = "New Group";
    int rows = 1;
    int cols = 1;

    float startOffsetX = 0.f;
    float startOffsetY = 0.f;
    float colSpacing = 0.f;
    float rowSpacing = 0.f;
    float slotSize = 1.f;
    float iconSize = 0.78f;
};

struct InventoryLayoutConfig
{
    std::vector<SlotGroup> slotGroups = {
        { "Hot Bar",   1, 10, -0.02f,  0.9f,  0.f,   0.f,   5.43f, 0.78f },
        { "Main Grid", 5,  6,  1.64f,  0.76f, 0.f,   0.09f, 3.26f, 0.78f }
    };

    // Icon position
    float iconOffsetX = 0.0f;
    float iconOffsetY = -1.06f;

    // Inventory dimensions
    float invAspectRatio = 150.0f / 200.0f;
    float invTextureScale = 5.84f;

    int TotalSlots() const {
        int total = 0;
        for (auto& g : slotGroups) total += g.rows * g.cols;
        return total;
    }

    // Popup
    float popupOffsetX = -8.5f;
    float popupOffsetY = 0.0f;
    float popupWidth = 4.0f;
    float popupHeight = 5.0f;

    // Popup buttons
    float buttonWidth = 1.1f;
    float buttonHeight = 0.4f;
    float buttonSpacing = 0.15f;
    float buttonOffsetY = -1.35f;
};
```
![alt text](/assets/inventorysystem/inventoryvariables-1.png)

*All inventory layout variables exposed in the ImGui panel — slot groups, icon offsets, popup dimensions, and button placement, all editable at runtime*

To avoid unnecessary GPU resource allocation, texture reloading is gated behind an explicit reload button rather than triggering on every keystroke, since each path change requires mesh and material recreation. Layout parameters update in real time as they only affect transform calculations.

![alt text](/assets/inventorysystem/paths.png)

*Texture path fields for the inventory background, popup, and sprite sheet — Reload recreates the materials without restarting the project*

### Positioning the Inventory Slots
With all parameters centralised in `InventoryLayoutConfig`, slot creation is fully data-driven. Instead of separate hardcoded loops for the hotbar and grid, a single loop iterates over every `SlotGroup` and derives each slot's position from that group's own row count, column count, size, and spacing values. Adding a new slot section to the inventory requires no changes to the positioning code at all — just a new entry in the `slotGroups` vector.

Each group calculates its own `baseX` and `baseY` as the top-left origin, then steps through rows and columns applying the group's spacing and offset values:
```cpp
for (auto& group : GetLayoutConfig().slotGroups)
{
    float invWidth = group.slotSize * m_invScale;
    float invHeight = invWidth * GetLayoutConfig().invAspectRatio;

    float slotSize = invWidth / (float)group.cols;

    float baseX = -invWidth / 2.f + slotSize / 2.f;
    float baseY = invHeight / 2.f - slotSize / 2.f;

    float iconScale = slotSize * group.iconSize;

    for (int row = 0; row < group.rows; row++)
    {
        for (int col = 0; col < group.cols; col++)
        {
            float x = baseX + (slotSize + group.colSpacing) * col
                + group.startOffsetX + GetLayoutConfig().iconOffsetX;
            float y = baseY - (slotSize + group.rowSpacing) * row
                + group.startOffsetY + GetLayoutConfig().iconOffsetY;

            iconTransform.SetTranslation(vec3(x, y, 0.2f));
            iconTransform.SetScale(vec3(iconScale));

            globalIdx++;
        }
    }
}
```
![alt text](/assets/inventorysystem/inventory2.png)
*Slot boundaries visualized with debug lines — yellow outlines mark each slot's hit region*

### Multi-Digit Quantity Display
Instead of rendering text dynamically (expensive), the system uses individual digit sprites (0-9) combined to show any number.
### The Approach:
Convert quantity to string, process each character, load corresponding digit sprite, and position side-by-side:
```cpp
std::string qtyStr = std::to_string(qty);  // e.g., 42 becomes "42"
int numDigits = static_cast<int>(qtyStr.length());

float digitSpacing = digitScale * 0.3f;
float totalWidth = (numDigits - 1) * digitSpacing;

float startX = slotPos.x + (iconScale * 0.4f) - totalWidth;
float startY = slotPos.y - (iconScale * 0.5f) + (digitScale * 0.5f) - 0.1f;

for (int d = 0; d < numDigits; d++)
{
    char digitChar = qtyStr[d];
    int digitValue = digitChar - '0';  // '4' -> 4, '2' -> 2
    
    auto digitEntity = ecs.CreateEntity();
    float digitX = startX + (d * digitSpacing);
    
    std::string digitTexture = "textures/digit-" + std::to_string(digitValue) + ".png";
    // Create and position individual digit sprite
}
```
![alt text](/assets/inventorysystem/numbers.png)

*Any quantity rendered by combining individual digit sprites positioned side-by-side*

## 💫 Item Management
### Item Creation Tool
The ImGui item creation panel lets designers create items entirely at runtime without touching any C++ code. Item types — such as Potion, Armour, Weapon, and Valuable — are stored in a `m_itemTypes` vector rather than a hardcoded enum, meaning new categories can be added, renamed, or removed live in the panel. Each type defines which region of the sprite sheet it maps to via `minX`, `maxX`, `minY`, `maxY` coordinates:
```cpp
m_itemTypes = {
{ "Potion",   0, 6,  0, 0  },
{ "Armour",   4, 7,  3, 5  },
{ "Valuable", 7, 15, 0, 10 },
{ "Weapon",   0, 3,  4, 8  }
};
```
Selecting a type in the panel filters the sprite picker to only show icons from that region. The picker renders each sprite as a clickable ImGui image button by computing its UV coordinates from the sheet dimensions, with the selected sprite highlighted in green:

![alt text](/assets/inventorysystem/itemtypes.png)

*Item type definitions panel — each type maps to a sprite sheet region, and types can be added or removed without touching code*

When the designer clicks "Create Item", the system generates a unique key from the item's name and sprite coordinates, registers both a mesh renderer and an Item entry in the shared databases, and spawns the item as a world entity directly in front of the player:

```cpp
std::string uniqueKey = names[i] + "_" +
    std::to_string(spriteXCoords[i]) + "_" +
    std::to_string(spriteYCoords[i]) + "_" +
    std::to_string(i);

graphics2d::m_itemDatabase[uniqueKey] = new Item(
    names[i], descriptions[i],
    graphics2d::m_iconSpriteSheetPath,
    types[i],
    spriteXCoords[i], spriteYCoords[i],
    isStackable[i], stackSizes[i]
);

glm::vec2 spawnPos2D = playerPos + glm::vec2(2.0f, 0.0f);
auto entity = ecs.CreateEntity();
ecs.CreateComponent<ItemPickup>(entity, uniqueKey);
```

### Stacking Logic
When adding items, the system first scans existing slots for a matching stackable item with space remaining, filling those before opening a new slot:
```cpp
for (int i = 0; i < m_slots.size(); i++)
{
    if (m_slots[i].IsEmpty())
        continue;

    Item* existingItem = m_slots[i].GetItem();

    if (existingItem == item || existingItem->IsSameItem(*item))
    {
        int currentQty = m_slots[i].GetQuantity();
        int maxStack = existingItem->GetMaxStackSize();
        int spaceAvailable = maxStack - currentQty;

        if (spaceAvailable > 0)
        {
            int amountToAdd = std::min(spaceAvailable, quantity);
            m_slots[i].AddQuantity(amountToAdd);
            quantity -= amountToAdd;

            if (quantity == 0)
                return true;
        }
    }
}

if (quantity > 0)
{
    int emptySlot = FindEmptySlot();
    if (emptySlot == -1)
        return false;

    m_slots[emptySlot].SetItem(item);
    m_slots[emptySlot].SetQuantity(quantity);
    return true;
}
```
<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/stacking.mp4" type="video/mp4">
</video>

## 🎭 Interaction Systems
### Hover Detection & Coordinate Transformation
To detect which slot the mouse is hovering over, I convert the mouse position from screen space into the same coordinate space the inventory slots live in — UI canvas space. I normalize the mouse position against the screen size and remap it to the canvas dimensions, then do a simple AABB check against each slot's position:
```cpp
glm::vec2 mouseUIPos = {
    (mouseScreenPos.x / screenWidth) * canvasSize.x - canvasSize.x / 2.f,
    -(((mouseScreenPos.y / screenHeight) * canvasSize.y - canvasSize.y / 2.f)) + 1.05f
};

float slotCenterX = baseX + (slotSize + group.colSpacing) * col
    + group.startOffsetX;
float slotCenterY = baseY - (slotSize + group.rowSpacing) * row
    + group.startOffsetY;

if (mouseUIPos.x >= minX && mouseUIPos.x <= maxX &&
    mouseUIPos.y >= minY && mouseUIPos.y <= maxY)
{
    currentHoveredSlot = globalIdx;
}
```
If a slot is hit, its icon scales up by 10% as visual feedback.

### Item Popup System
Popup displays item name/description using SDL_ttf for text rendering. Three action buttons (Use, Equip, Drop) are part of the popup sprite with invisible collision boxes for click detection. The popup can be locked in place by clicking on an item, keeping it visible when moving the mouse away, and unlocks when clicking outside the inventory.
```cpp
void UpdatePopupText(const std::string& itemName, const std::string& itemDescription)
{
    TTF_Font* font = TTF_OpenFont("assets/textures/font.ttf", 24);
    SDL_Color textColor = { 255, 255, 255, 255 };

    int nameWidth, nameHeight;
    auto nameImage = CreateTextImage(itemName, font, textColor, nameWidth, nameHeight);

    if (nameImage)
    {
        auto nameTexture = std::make_shared<Texture>(nameImage, nameSampler);
        auto nameMaterial = std::make_shared<Material>();
        nameMaterial->BaseColorTexture = nameTexture;
        
        auto& renderer = ecs.Registry.get<MeshRenderer>(m_popupItemName);
        renderer.Material = nameMaterial;
    }
}
```
<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/hovering.mp4" type="video/mp4">
</video>

## ⚙️ Advanced Features
### Drag and Drop System
State machine tracking mouse button, drag distance threshold, and quantity to move:
**Left-click** - Drags entire stack

**Right-click** - Drags half (rounded up)

```cpp
// Detect potential drag start
if (!m_isDragging && (leftPressed || rightPressed))
{
    m_dragStartMousePos = glm::vec2(mouseWorldX, mouseWorldY);
    m_potentialDragSlot = currentHoveredSlot;
    m_potentialDragIsRightClick = rightPressed;
}

// Check distance threshold
float dragDistance = glm::length(currentMousePos - m_dragStartMousePos);
const float dragThreshold = 0.1f;

if (dragDistance > dragThreshold && m_potentialDragSlot >= 0)
{
    m_isDragging = true;
    
    if (m_isSplitDragging)
    {
        int totalQuantity = slot.GetQuantity();
        m_splitDragQuantity = (totalQuantity + 1) / 2; // Half, rounded up
    }
    
    CreateDragGhost(m_potentialDragSlot); // Semi-transparent entity follows mouse
}
```

### Drop Handling:
```cpp
if ((!leftButton && !rightButton) && m_isDragging)
{
    if (currentHoveredSlot != -1 && currentHoveredSlot != m_dragSourceSlot)
    {
        if (m_isSplitDragging)
            MoveSplitItems(m_dragSourceSlot, currentHoveredSlot, m_splitDragQuantity);
        else
            MoveItems(m_dragSourceSlot, currentHoveredSlot);
    }
    
    DestroyDragGhost();
    m_isDragging = false;
}
```

After implementing drag and drop, I added a m_wasClick flag to distinguish between actual clicks (for locking the popup) and drag operations, and modified hover detection to not interfere when the popup is locked or actively dragging:

```cpp
// Modified hover detection to handle interaction states
if (!m_popupLocked && !m_isDragging)
{
    if (currentHoveredSlot != m_hoveredSlotIndex)
    {
        if (m_hoveredSlotIndex != -1)
            OnSlotHoverExit(m_hoveredSlotIndex);
        if (currentHoveredSlot != -1)
            OnSlotHoverEnter(currentHoveredSlot);
        m_hoveredSlotIndex = currentHoveredSlot;
    }
}

// Distinguish clicks from drags
if (m_wasClick && !m_isDragging)
{
    if (currentHoveredSlot != -1)
    {
        m_popupLocked = true;
        m_selectedSlotIndex = currentHoveredSlot;
    }
    m_wasClick = false;
}
```

<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/draganddrop.mp4" type="video/mp4" alt="DragAndDrop">
</video>

### Item Actions
**Use (Potions):**
```cpp
void InventoryUI::UseItem(int slotIndex, Item* item)
{
    if (!item)
        return;

    m_playerInventory->RemoveItem(slotIndex, 1);
    printf("Used item: %s\n", item->GetName().c_str());
    m_popupLocked = false;
    m_selectedSlotIndex = -1;
    m_hoveredSlotIndex = -1;
    HidePopup();
}
```

**Drop:**
```cpp
void DropItem(int slotIndex, Item* item)
{
    glm::vec2 playerPos = playerBody->GetPosition();
    glm::vec2 dropPos = playerPos + glm::vec2(2.0f, 0.0f);

    auto entity = CreateObject(itemID, vec3(dropPos.x, dropPos.y, 1.0f), true, 0.4f);
    ecs.CreateComponent<ItemPickup>(entity, itemID);

    m_playerInventory->RemoveItem(slotIndex, 1);
}
```

**Equip:**
```cpp
void EquipItem(int slotIndex, Item* item)
{
    // Delete previously equipped item
    if (ecs.Registry.valid(m_equippedItem))
        ecs.DeleteEntity(m_equippedItem);

    glm::vec2 playerPos = playerBody->GetPosition();
    m_equippedItem = CreateObject(itemID, vec3(playerPos.x + 0.8f, playerPos.y, 1.0f), false, 0.0f);
}

// In Update - equipped item follows player
if (ecs.Registry.valid(m_equippedItem))
{
    glm::vec2 playerPos = playerBody->GetPosition();
    equippedTransform.SetTranslation(vec3(playerPos.x + 0.8f, playerPos.y, 1.0f));
}
```

<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/buttons.mp4" type="video/mp4" alt="Buttons">
</video>

## 💾 Serialization System
### JSON Save/Load
Due to compilation conflicts between Windows headers (from OpenGL) and C++17's std::byte, serialization code was separated into dedicated files with forward declarations to avoid the symbol ambiguity while keeping full functionality.

The save file bundles two things together into a single SaveData struct: the item data (definitions and world spawn positions) and the full layout configuration. This means an entire session — every item, its position in the world, and every layout parameter tweaked in ImGui — can be saved and restored exactly as left.

Collecting item data gathers every entry in the item database along with all current world-space spawn positions from ECS entities:

```cpp
LevelItemData ItemCreation::CollectItemData() const
{
    LevelItemData data;

    for (const auto& [itemID, item] : graphics2d::m_itemDatabase)
    {
        ItemDefinitionData def;
        def.itemID      = itemID;
        def.name        = item->GetName();
        def.description = item->GetDescription();
        def.texturePath = item->GetTexturePath();
        def.type        = item->GetType();
        def.stackable   = item->IsStackable();
        def.maxStackSize = item->GetMaxStackSize();
        def.spriteX     = item->GetSpriteX();
        def.spriteY     = item->GetSpriteY();
        data.definitions.push_back(def);
    }

    for (const auto& [entity, pickup, transform] :
         ecs.Registry.view<ItemPickup, bee::Transform>().each())
    {
        ItemSpawnData spawn;
        spawn.itemID   = pickup.m_itemName;
        spawn.position = glm::vec2(transform.GetTranslation());
        data.spawns.push_back(spawn);
    }

    return data;
}
```
Collecting layout data snapshots the entire `InventoryLayoutConfig` — icon offsets, inventory scale, popup dimensions, button placement, all texture paths, sprite sheet dimensions, item type definitions, and every slot group:
```cpp
InventoryLayoutData ItemCreation::CollectLayoutData() const
{
    InventoryLayoutData data;
    const auto& cfg = ...GetLayoutConfig();

    data.iconOffsetX          = cfg.iconOffsetX;
    data.iconOffsetY          = cfg.iconOffsetY;
    data.invTextureScale      = cfg.invTextureScale;
    data.iconSpriteSheetPath  = graphics2d::m_iconSpriteSheetPath;
    data.inventoryPath        = graphics2d::m_inventoryPath;
    data.popupPath            = graphics2d::m_popupPath;
    data.popupOffsetX         = cfg.popupOffsetX;
    data.popupOffsetY         = cfg.popupOffsetY;
    data.popupWidth           = cfg.popupWidth;
    data.popupHeight          = cfg.popupHeight;
    data.buttonWidth          = cfg.buttonWidth;
    data.buttonHeight         = cfg.buttonHeight;
    data.buttonSpacing        = cfg.buttonSpacing;
    data.buttonOffsetY        = cfg.buttonOffsetY;
    // ...

    for (const auto& t : m_itemTypes)
        data.itemTypes.push_back({ t.name, t.minX, t.maxX, t.minY, t.maxY });

    for (const auto& group : cfg.slotGroups)
    {
        SlotGroupData gd;
        gd.name         = group.name;
        gd.rows         = group.rows;
        gd.cols         = group.cols;
        gd.startOffsetX = group.startOffsetX;
        gd.startOffsetY = group.startOffsetY;
        gd.colSpacing   = group.colSpacing;
        gd.rowSpacing   = group.rowSpacing;
        gd.slotSize     = group.slotSize;
        gd.iconSize     = group.iconSize;
        data.slotGroups.push_back(gd);
    }

    return data;
}
```

On load, `ApplyLayoutData` writes all values back into the live `InventoryLayoutConfig`, clears and reconstructs the item type list and slot groups, then calls `RebuildInventory`, `ReloadTextures`, and `UpdateInventoryLayout` in sequence to fully restore the session state.

<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/saveandload.mp4" type="video/mp4">
</video>
<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/serialization.mp4" type="video/mp4">
</video>

## Here are two examples showing how differently the system can be configured:

![alt text](/assets/inventorysystem/inventory2.png) 
![alt text](/assets/inventorysystem/inventory1.png) 

<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/demo2.mp4" type="video/mp4">
</video>
<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/demo1.mp4" type="video/mp4">
</video>

## 🎯 Key Achievements

- Built complete inventory system from scratch without relying on engine UI features
- Solved performance challenges through caching and entity lifecycle management
- Created designer-friendly tools enabling item creation without code
- Designed intuitive drag-and-drop with split-stack functionality
- Developed multi-digit display system using individual sprite entities
- Implemented full serialization supporting save/load of entire item layouts
- Integrated seamlessly with ECS architecture for consistent item behavior
- Ported the system into a second project (RTS), demonstrating engine-agnostic reusability
- Designed clean encapsulation through the graphics2d namespace and validated public APIs


## 📚 References

- SDL2 Development Library - https://www.libsdl.org/
nlohmann JSON for Modern C++ - https://github.com/nlohmann/json
- Game Programming Patterns: Component - https://gameprogrammingpatterns.com/component.html
- OpenGL - https://www.opengl.org/
- SDL2 Drag and Drop Implementation - https://gigi.nullneuron.net/gigilabs/sdl2-drag-and-drop/
