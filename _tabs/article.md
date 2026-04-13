---
title: "Article"
layout: page
icon: fa-solid fa-newspaper
order: 2
---

# Building an Inventory System from Scratch

![alt text](/assets/article/main.png)
![alt text](/assets/inventorysystem/main2.png)

## Introduction

Ever wondered how inventory systems actually work under the hood? Not the Unity tutorial version where you drag components around, but the *real* nitty-gritty implementation? That's what I set out to build.

**Prerequisites:** Basic C++ knowledge will help you follow along. Fair warning though - I built this in Visual Studio using a custom template from my university (hence some of the quirky code structure you might spot), SDL2 for text rendering, ImGui for item creation and a whole lot of trial and error.

### Why Build This?

An inventory system is often the backbone of survival games, RPGs, and adventure games. Think Minecraft's grid system, 7 Days to Die's crafting integration, or Dying Light's equipment management. I couldn't pick just one to emulate, so I basically mashed together my favorite features from all of them. 

Here's the thing - most inventory tutorials online assume you're using Unity or Unreal, where half the work is already done for you with built-in UI systems and drag-and-drop components. But what if you want to understand what's *actually* happening? What if you're working with a custom engine? That's where this project came in.

### What I Built

Here's what this system can do:
- **Custom item creation tool** - Level designers can create items without touching code
- **Drag and drop** - Because clicking to move items is so 2005
- **Right-click stack splitting** - For when you need exactly 32 wood planks, not 64
- **Multi-digit quantity displays** - Your 9,999 gold coins deserve proper representation
- **Item popup system** - Use, equip, or drop items with a clean context menu
- **Full serialization** - Save and load item data and positions in the scene
- **Fully configurable layout** - Swap inventory visuals, slot groups, and dimensions entirely through ImGui at runtime, no code required

### Starting Point: Design First, Code Later

Before I wrote a single line of inventory code, I needed to figure out what this thing should actually *look like* and *behave like*. When I started, I had a blank canvas and way too many ideas. So I broke it down into the core questions every inventory system needs to answer:

**The Big Questions:**
1. How many slots should the inventory have?
2. Should items stack? If yes, what's the max stack size?
3. What information does each item need to store?
4. How should the UI look and feel?
5. What can players actually *do* with items?

For my system, I landed on:
- **Grid-based layout** - A 10x4 grid (40 slots total) felt right for a survival-style game
- **Stackable items** - Because who wants to carry 50 individual arrows in separate slots?
- **Item properties** - Name, description, type (potions/armour/weapons/valuable), sprite, and stack limit
- **Visual feedback** - Highlight slots on hover, show quantities clearly, smooth drag animations

## The ECS Approach

My engine uses an Entity Component System (ECS) architecture, which means items aren't just structs sitting in an array - they're actual entities in the world. This might seem overkill for an inventory, but it came with some nice benefits:

**Items as Entities:**
- Each item in the inventory is an entity with components (Transform, Sprite, ItemData)
- When you drop an item, it already exists in the world - no conversion needed
- Picking up items? Just move the entity from the world to the inventory
- Consistent behavior whether items are held or dropped

**The tricky part?** Managing entity lifecycles. Items needed to exist in different "states":
- In the world (normal entity with physics)
- In the inventory (visual only, no physics)
- Being dragged (following mouse, semi-transparent)

I handled this with a simple state machine and component enable/disable logic.

### Data Structure

Here's the basic structure for the inventory itself:

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

Each inventory slot is super simple:

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
Last but not least, the item class:

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
Clean and straightforward. No over-engineering here.

## Building the Inventory UI

Now for the visual part - actually displaying the inventory on screen.

### Creating the Inventory Entity

I followed the template's approach of using quads with mesh renderers and materials. Here's how I set up the main inventory UI:
```cpp
  m_meshRenderers["Inventory"] =
     MeshRenderer(CreateQuad(vec2(0.f), vec2(invTextureWidth, invTextureHeight)), CreateMaterial(FileIO::Directory::Assets, m_inventoryPath));
```

The key decision here: **parenting to the UI canvas**. This ensures the inventory moves with the camera and scales properly across different screen sizes.

### Show/Hide with Alpha Blending

I initially tried scaling the inventory to 0 for hiding it, but this broke the child icon transforms. The solution? **Alpha blending**:

**Important detail**: I also stop player movement when the inventory is open by setting linear velocity to 0 in the player control code.

```cpp
// Smooth fade in/out
    float targetAlpha = m_playerInventory->IsVisible() ? 1.f : 0.f;
    float currentAlpha = renderer.Material->BaseColorFactor.a;
    float newAlpha = Damp(currentAlpha, targetAlpha, 15.f, dt);

    renderer.Material->BaseColorFactor.a = newAlpha;
```

## Positioning the Inventory Slots

This was trickier than expected - and honestly went through a big redesign. My original approach hardcoded separate loops for the hotbar and the storage grid, with magic offset numbers I'd tweaked by hand until things lined up. It worked, but adding a third slot section would have meant writing another loop from scratch. That's not great.
So I ripped it out and replaced it with a data-driven system.

### InventoryLayoutConfig

Before any slots are positioned or rendered, all layout parameters flow through a single `InventoryLayoutConfig` struct. Every rendering and hit-detection calculation references this struct directly - so changing a value (slot count, spacing, popup size, button offset) propagates instantly through the entire system without touching any other code.

The key idea is that the inventory isn't limited to a fixed hotbar + grid layout. The config holds a vector of `SlotGroup` entries, meaning any number of groups can be added, each with its own row/column count, size, and spacing. Want a hotbar, a main grid, and an equipment panel? Just add another entry to the vector - no code changes required.

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

All of these values are exposed in an ImGui panel at runtime. Layout parameters like spacing and offsets update in real time since they only affect transform calculations. Texture paths are gated behind an explicit Reload button - loading a new texture requires mesh and material recreation, so you don't want that firing on every keystroke.

![alt text](/assets/inventorysystem/inventoryvariables.png)
*All inventory layout variables exposed in the ImGui panel — slot groups, icon offsets, popup dimensions, and button placement, all editable at runtime*

### The Data-Driven Slot Loop

With everything centralised in `InventoryLayoutConfig`, slot creation becomes a single loop that works for any layout - no separate hotbar and grid code, no hardcoded magic numbers. It iterates over every `SlotGroup` and derives each slot's position from that group's own settings:

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

Adding a new slot section to the inventory now requires no changes to this code at all - just a new entry in the `slotGroups` vector.

**Pro tip:** Use debug rendering to visualise the slot boundaries while tweaking the offset values. I had debug lines drawing rectangles around each slot, which made alignment much easier.

![alt text](/assets/article/debuglines.png)
*Any quantity rendered by combining individual digit sprites positioned side-by-side*


## Creating Items in the World

Before items can go into the inventory, they need to exist in the game world as entities that the player can pick up. 

### The Item Database

I maintain a global item database that stores the definitions of all items in the game. This database acts as a template - when I spawn an item in the world, I reference this database to get its properties.
```cpp
m_itemDatabase["Potion1"] = new Item(
    "Health Potion",
    "Restores 1 heart of health",
    "textures/itemset.png",
    ItemType::Potion,
    0, 0,  // sprite coordinates in sheet
    true,  // stackable
    99     // max stack size
);
```
### Creating Mesh Renderers for Items
For each item type, I create a reusable mesh renderer using the sprite sheet:
```cpp
auto spriteSheetMaterial = CreateMaterial(FileIO::Directory::Assets, "textures/itemset.png");
spriteSheetMaterial->IsUnlit = true;

m_meshRenderers["Potion1"] = MeshRenderer(
    CreateSpriteSheetQuad(0, 0, 16, 11, vec2(0.8f, 0.8f)), 
    spriteSheetMaterial
);
```
The `CreateSpriteSheetQuad` function calculates UV coordinates for a specific sprite in the sheet.

### Spawning Items in the World

I use a helper function which does all the heavy lifting, it creates the meshRenderer and the physics collider so that the items can be picked up.

```cpp
auto& meshRenderer = m_meshRenderers[meshRendererName];
ecs.CreateComponent(entity, meshRenderer.Mesh, meshRenderer.Material);
auto& collider = ecs.CreateComponent(entity, radius);
ecs.CreateComponent(entity, physics::Body::Type::Kinematic, collider, 1.0f, 1.0f).SetPosition(vec2(position));
```

### The ItemPickup - custom component that marks this as a pickable item
This simple component just stores which item this entity represents:
```cpp
struct ItemPickup {
    std::string m_itemName;  // Key into m_itemDatabase
    
    ItemPickup(const std::string& itemName) : m_itemName(itemName) {}
};
```
When the player collides with an entity that has `ItemPickup`, I know to add it to the inventory.

### Adding Items - The Stacking Logic

This is where it gets interesting. When adding an item, the inventory needs to:
1. Check if the item already exists in a slot (for stacking)
2. Find the first available empty slot if it doesn't exist
3. Handle partial stacks when a slot is nearly full
```cpp
if (item->IsStackable())
{
	// First pass: try to stack with existing items
	for (int i = 0; i < m_slots.size(); i++)
	{
		if (m_slots[i].IsEmpty())
			continue; // Skip empty slots in this pass

		Item* existingItem = m_slots[i].GetItem();

		// If it's literally the same Item pointer, skip the IsSameItem check
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

				// If we've added everything, we're done!
				if (quantity == 0)
					return true;

				// Otherwise, continue to next slot with remaining quantity
			}
		}
	}
	// If we still have items left, we need a new slot
	if (quantity > 0)
	{
		int emptySlot = FindEmptySlot();
		if (emptySlot == -1)
			return false; // Inventory full

		m_slots[emptySlot].SetItem(item);
		m_slots[emptySlot].SetQuantity(quantity);
		return true;
	}

	return true;
}
else
{
	// Non-stackable item
	int emptySlot = FindEmptySlot();
	if (emptySlot == -1)
		return false;

	m_slots[emptySlot].SetItem(item);
	m_slots[emptySlot].SetQuantity(quantity);
	return true;
}
```
**The algorithm breaks down to:**
1. **Stacking pass** - Fill existing partial stacks first (most space-efficient)
2. **Empty slot pass** - Create new stacks in empty slots for remaining items
3. **Overflow handling** - Return false if inventory is full

<video width="320" height="240" controls>
  <source src="/assets/article/additems.mp4" type="video/mp4">
</video>

## Item Creation Tool

To enhance new items being added to the scene, I created an interface to help level designers - or you - visualise items before spawning them. For this I used ImGui and added all the variables that can be changed for items. For choosing the item sprite, I also made the picking visible using boxes where icons are displayed based on certain parts of the spritesheet for each type.

To make adding new items easier for level designers (and myself), I built a visual editor tool using ImGui. This lets you create items without touching code and runs directly in the game window alongside the scene.

### Configurable Item Types

One thing I wanted to avoid was a hardcoded enum for item categories. If a designer wanted a new type called "Quest Item" or "Consumable", they'd have to recompile. Instead, item types are stored in a `m_itemTypes` vector that can be edited live in the panel:
```cpp
m_itemTypes = {
    { "Potion",   0, 6,  0, 0  },
    { "Armour",   4, 7,  3, 5  },
    { "Valuable", 7, 15, 0, 10 },
    { "Weapon",   0, 3,  4, 8  }
};
```

Each entry defines which region of the sprite sheet maps to that type (`minX`, `maxX`, `minY`, `maxY`). Selecting a type in the panel filters the sprite picker to only show icons from that region.

### Visual Sprite Selection

Instead of typing sprite coordinates like `(5, 3)`, I display the actual sprite sheet with clickable boxes. The picker renders each sprite as a clickable ImGui image button by computing UV coordinates from the sheet dimensions, with the selected sprite highlighted in green.

When the designer clicks **Create Item**, the system generates a unique key from the item's name and sprite coordinates, registers both a mesh renderer and an Item entry in the shared databases, and spawns the item as a world entity directly in front of the player:
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

![alt text](/assets/inventorysystem/itemtypes.png)
*Item type definitions panel — each type maps to a sprite sheet region, and types can be added or removed without touching code*

## Displaying Item Quantities with Individual Digit Sprites

One of the visual details I'm proud of is how item quantities are displayed. Instead of rendering text dynamically (which can be expensive), I use individual digit sprites (0-9) that I combine to show any number.

### The Approach

The system displays any quantity by converting the number to a string, then processing each character individually. For example, the number 42 becomes the string "42", which contains two characters: '4' and '2'. The code loops through each character, converts it back to its numeric value (using the technique digitChar - '0', which turns '4' into 4 and '2' into 2), and loads the corresponding digit image file (digit-4.png and digit-2.png). Each digit is created as a separate visual entity and positioned side-by-side with small spacing between them in the bottom-right corner of the inventory slot. This way, any number—whether it's 9, 42, or 999—can be displayed by combining individual digit sprites (0-9) positioned next to each other.

![alt text](/assets/article/digitshowcase.png)

```cpp
// Convert the quantity to a string of digits
std::string qtyStr = std::to_string(qty);  // e.g., qty=42 becomes "42"
int numDigits = static_cast<int>(qtyStr.length());  // = 2 digits

// Calculate spacing between digits
float digitSpacing = digitScale * 0.3f;

// Calculate total width needed for all digits
float totalWidth = (numDigits - 1) * digitSpacing;

// Position for the first digit (right-aligned)
float startX = slotPos.x + (iconScale * 0.4f) - totalWidth;
float startY = slotPos.y - (iconScale * 0.5f) + (digitScale * 0.5f) - 0.1f;

// Loop through each digit character in the string
for (int d = 0; d < numDigits; d++)
{
    char digitChar = qtyStr[d];  // Get character '4', then '2'
    int digitValue = digitChar - '0';  // Convert character to integer: '4' -> 4, '2' -> 2
    
    // Create separate entity for this digit
    auto digitEntity = ecs.CreateEntity();
    auto& digitTransform = ecs.CreateComponent<Transform>(digitEntity);
    
    // Position this digit (each digit is spaced apart)
    float digitX = startX + (d * digitSpacing);
    digitTransform.SetTranslation(vec3(digitX, startY, 27.f));
    
    // Load the correct digit texture (digit-0.png, digit-1.png, etc.)
    std::string digitTexture = "textures/digit-" + std::to_string(digitValue) + ".png";
    
    auto digitMaterial = CreateMaterial(FileIO::Directory::Assets, digitTexture);
    
    // Render this individual digit
    ecs.CreateComponent<MeshRenderer>(digitEntity, m_cachedDigitMesh, digitMaterial);
}
```

## Implementing Hover Detection and Item Popups

Once items are displayed in inventory slots, players need a way to see what each item is. This is where hover detection and item popups come in.

### The Hover Detection

Getting hover detection right took a couple of attempts. My first version converted the mouse position all the way through to world space and compared it against each slot's world position. That worked, but it was fragile - camera movement affected the calculation, and the code was pretty hard to follow.

The cleaner solution was to keep everything in **UI canvas space** the whole time. Instead of projecting out to the world and back, I normalize the mouse position against the screen size and remap it directly to canvas dimensions:

 ```cpp
glm::vec2 mouseUIPos = {
    (mouseScreenPos.x / screenWidth) * canvasSize.x - canvasSize.x / 2.f,
    -(((mouseScreenPos.y / screenHeight) * canvasSize.y - canvasSize.y / 2.f)) + 1.05f
};
 ```
 Then the AABB check uses the same slot positions calculated by the layout loop - no separate coordinate conversion needed:

 ```cpp
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

When a slot is hovered, its icon scales up by 10% as visual feedback:

```cpp
float finalScale = iconScale;
if (i == m_hoveredSlotIndex)
{
    finalScale *= 1.1f; // 10% larger when hovered
}
iconTransform.SetScale(vec3(finalScale));
```

The big win here is that the hover detection now reuses the same `SlotGroup` data as the positioning loop. If you add a new slot group or change a spacing value, hover detection updates automatically - there's no second place to keep in sync.

### Item Popup System

Additionally, a popup quad displaying the item's name and description appears when hovering over an item. The text rendering for the popup is implemented using SDL_ttf, which generates texture images from text that are then displayed on the popup. The three action buttons (Use, Equip, Drop) are part of the popup sprite itself rather than separate UI elements. To detect button clicks, I created collision boxes positioned to match the visual buttons in the sprite image, checking if the mouse position intersects with these invisible AABB regions to determine which button was clicked. This popup can be locked in place by clicking on the item, keeping it visible even when moving the mouse away, and unlocks when clicking outside the inventory, causing it to disappear.
```cpp
m_popup = ecs.CreateEntity();
auto& transform = ecs.CreateComponent<Transform>(m_popup);
transform.Name = "Popup";
transform.SetParent(m_uiCanvas);

auto& meshRenderer = m_meshRenderers["Popup"];
ecs.CreateComponent<MeshRenderer>(m_popup, meshRenderer.Mesh, meshRenderer.Material);

// Lock popup on click (from UpdateInventoryHover)
if (currentHoveredSlot != -1)
{
    m_popupLocked = true;
    m_selectedSlotIndex = currentHoveredSlot;
}
else if (m_popupLocked)
{
    // Unlock when clicking outside
    m_popupLocked = false;
    m_selectedSlotIndex = -1;
}
```
```cpp
void platformer::Platformer::UpdatePopupText(const std::string& itemName, const std::string& itemDescription)
{
    // Load font
    TTF_Font* font = TTF_OpenFont("assets/textures/font.ttf", 24);
    
    SDL_Color textColor = { 255, 255, 255, 255 };

    // Create text image using SDL_ttf
    int nameWidth, nameHeight;
    auto nameImage = CreateTextImage(itemName, font, textColor, nameWidth, nameHeight);

    if (nameImage)
    {
        // Create texture from the rendered text image
        auto nameTexture = std::make_shared<Texture>(nameImage, nameSampler);
        auto nameMaterial = std::make_shared<Material>();
        nameMaterial->BaseColorTexture = nameTexture;
        nameMaterial->UseBaseTexture = true;
        nameMaterial->IsUnlit = true;

        // Apply to mesh renderer to display on screen
        auto& renderer = ecs.Registry.get<MeshRenderer>(m_popupItemName);
        renderer.Mesh = nameMesh;
        renderer.Material = nameMaterial;
    }
    
    TTF_CloseFont(font);
}
```

<video width="320" height="240" controls>
  <source src="/assets/article/hovering.mp4" type="video/mp4">
</video>

## Drag and Drop System

For the drag and drop system, I implemented a state machine that tracks whether the user is dragging, which mouse button was pressed, and how many items to move. When the user presses the left mouse button on a slot, the system prepares to drag all items from that slot. If the right mouse button is pressed instead, it calculates half the quantity (using `(totalQuantity + 1) / 2` to round up for odd numbers) and only drags that amount. The dragging doesn't start immediately—there's a distance threshold to distinguish between clicks and drags. Once the mouse moves beyond this threshold, a semi-transparent "ghost" entity is created that follows the mouse cursor, visually representing the items being dragged. When the mouse button is released, the system checks which slot the mouse is hovering over and either stacks the items if they match and have space, or places them in an empty slot. If the player tries to drag onto a slot containing a different item type, the items automatically return to the source slot and no changes occur. After implementing drag and drop, the hover detection logic needed modification because the system was now tracking multiple interaction states (potential drag, active drag, and locked popup). I added a `m_wasClick` flag to distinguish between actual clicks (for locking the popup) and drag operations, and modified the hover detection to not interfere when the popup is locked or when actively dragging, ensuring that hover effects only apply during appropriate interaction states.

<video width="320" height="240" controls>
  <source src="/assets/article/draganddrop.mp4" type="video/mp4">
</video>

```cpp
// Detect potential drag start
if (!m_isDragging)
{
    bool leftPressed = Engine.Input().GetMouseButtonOnce(Input::MouseButton::Left);
    bool rightPressed = Engine.Input().GetMouseButtonOnce(Input::MouseButton::Right);
    
    if (leftPressed || rightPressed)
    {
        if (currentHoveredSlot != -1)
        {
            m_dragStartMousePos = glm::vec2(mouseWorldX, mouseWorldY);
            m_potentialDragSlot = currentHoveredSlot;
            m_potentialDragIsRightClick = rightPressed;
        }
    }
}

// Check if mouse moved enough to start dragging
float dragDistance = glm::length(currentMousePos - m_dragStartMousePos);
const float dragThreshold = 0.1f;

if (dragDistance > dragThreshold && m_potentialDragSlot >= 0)
{
    m_isDragging = true;
    m_isSplitDragging = m_potentialDragIsRightClick;
    
    // Calculate quantity to drag
    if (m_isSplitDragging)
    {
        int totalQuantity = slot.GetQuantity();
        m_splitDragQuantity = (totalQuantity + 1) / 2; // Half, rounded up
    }
    else
    {
        m_splitDragQuantity = slot.GetQuantity(); // All items
    }
    
    CreateDragGhost(m_potentialDragSlot);
}
```
```cpp
// Drop items when button released
if ((!Engine.Input().GetMouseButton(Input::MouseButton::Left) && 
     !Engine.Input().GetMouseButton(Input::MouseButton::Right)) && m_isDragging)
{
    if (currentHoveredSlot != -1 && currentHoveredSlot != m_dragSourceSlot)
    {
        if (m_isSplitDragging)
        {
            MoveSplitItems(m_dragSourceSlot, currentHoveredSlot, m_splitDragQuantity);
        }
        else
        {
            MoveItems(m_dragSourceSlot, currentHoveredSlot);
        }
    }
    
    DestroyDragGhost();
    m_isDragging = false;
}
```
```cpp
// Modified hover detection to handle interaction states
if (!m_popupLocked && !m_isDragging)
{
    if (currentHoveredSlot != m_hoveredSlotIndex)
    {
        if (m_hoveredSlotIndex != -1)
        {
            OnSlotHoverExit(m_hoveredSlotIndex);
        }
        if (currentHoveredSlot != -1)
        {
            OnSlotHoverEnter(currentHoveredSlot);
        }
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

## Item Serialization

For item serialization, I implemented a JSON-based save and load system. Due to a compilation conflict between Windows headers (included through OpenGL) and C++17's `std::byte` type used in the serialization library, I had to split the serialization code into a separate file (`item_serialization.cpp`) with its own helper header (`item_serialization_helpers.hpp`) that uses forward declarations instead of full includes. This separation prevents the `byte` symbol ambiguity while maintaining functionality.

The save file bundles two things together into a single `SaveData` struct: the item data (definitions and world spawn positions) and the full layout configuration. This means an entire session — every item, its position in the world, and every layout parameter tweaked in ImGui — can be saved and restored exactly as you left it.

Collecting item data gathers every entry in the item database along with all current world-space spawn positions from ECS entities:

```cpp
LevelItemData ItemCreation::CollectItemData() const
{
    LevelItemData data;
    
    for (const auto& [itemID, item] : graphics2d::m_itemDatabase)
    {
        ItemDefinitionData def;
        def.itemID = itemID;
        def.name = item->GetName();
        def.description = item->GetDescription();
        def.spriteX = item->GetSpriteX();
        def.spriteY = item->GetSpriteY();
        // ... other properties
        data.definitions.push_back(def);
    }
    
    for (const auto& [entity, pickup, transform] : ecs.Registry.view<ItemPickup, bee::Transform>().each())
    {
        ItemSpawnData spawn;
        spawn.itemID = pickup.m_itemName;
        spawn.position = glm::vec2(transform.GetTranslation());
        data.spawns.push_back(spawn);
    }
    
    return data;
}
```

Collecting layout data snapshots the entire `InventoryLayoutConfig` — icon offsets, inventory scale, popup dimensions, button placement, all texture paths, and every slot group:

```cpp
InventoryLayoutData ItemCreation::CollectLayoutData() const
{
    InventoryLayoutData data;
    const auto& cfg = GetLayoutConfig();

    data.iconOffsetX         = cfg.iconOffsetX;
    data.iconOffsetY         = cfg.iconOffsetY;
    data.invTextureScale     = cfg.invTextureScale;
    data.iconSpriteSheetPath = graphics2d::m_iconSpriteSheetPath;
    data.inventoryPath       = graphics2d::m_inventoryPath;
    data.popupPath           = graphics2d::m_popupPath;
    data.popupOffsetX        = cfg.popupOffsetX;
    data.popupOffsetY        = cfg.popupOffsetY;
    data.popupWidth          = cfg.popupWidth;
    data.popupHeight         = cfg.popupHeight;
    data.buttonWidth         = cfg.buttonWidth;
    data.buttonHeight        = cfg.buttonHeight;
    data.buttonSpacing       = cfg.buttonSpacing;
    data.buttonOffsetY       = cfg.buttonOffsetY;

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
On load, `ApplyLayoutData` writes all values back into the live `InventoryLayoutConfig`, clears and reconstructs the item type list and slot groups, then calls `RebuildInventory`, `ReloadTextures`, and `UpdateInventoryLayout` in sequence to fully restore the session.

<video width="320" height="240" controls>
  <source src="/assets/article/saveandload.mp4" type="video/mp4">
</video>

<video width="320" height="240" controls>
  <source src="/assets/inventorysystem/serialization.mp4" type="video/mp4">
</video>


## Adding functionality to the "Use" button

When using the Use button, the system removes one item from the inventory stack and closes the popup. The slot quantity decreases by one, and if that was the last item in the slot, it clears entirely and becomes ready to accept new items.
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

## Adding functionality to the "Drop" button

When the player clicks the "Drop" button, the selected item is removed from the inventory and spawns as a physical entity in the game world positioned in front of the player. This dropped item behaves like any other item in the world—it has a collider, is animated, and can be picked up again by the player when they walk over it, returning it to the inventory.
```cpp
void InventoryUI::DropItem(int slotIndex, Item* item)
{
    // Get player position
    auto* playerBody = ecs.Registry.try_get<physics::Body>(m_player);
    glm::vec2 playerPos = playerBody->GetPosition();
    glm::vec2 dropPos = playerPos + glm::vec2(2.0f, 0.0f); // Drop in front of player

    // Create sprite mesh for the item
    auto spriteMesh = CreateSpriteSheetQuad(item->GetSpriteX(), item->GetSpriteY(), 16, 11, glm::vec2(0.8f, 0.8f));
    
    // Create the entity in the world with collider
    glm::vec3 spawnPos3D(dropPos.x, dropPos.y, 1.0f);
    auto entity = CreateObject(itemID, spawnPos3D, true, 0.4f);
    ecs.CreateComponent<platformer::ItemPickup>(entity, itemID);

    // Remove from inventory
    m_playerInventory->RemoveItem(slotIndex, 1);
}
```

## Adding functionality to the "Equip" button

When the player clicks the "Equip" button, the selected item appears as a visual entity in front of the player and follows their movement as they walk around. If the player equips a different item while one is already equipped, the previously equipped item is deleted from the scene and replaced with the new one, ensuring only one item can be equipped at a time.

```cpp
void InventoryUI::EquipItem(int slotIndex, Item* item)
{
    // If there's already an equipped item, delete it first
    if (static_cast<int>(m_equippedItem) != -1 && ecs.Registry.valid(m_equippedItem))
    {
        ecs.DeleteEntity(m_equippedItem);
    }

    // Get player position
    auto* playerBody = ecs.Registry.try_get<physics::Body>(m_player);
    glm::vec2 playerPos = playerBody->GetPosition();

    // Create equipped item entity (WITHOUT collider so it can't be picked up)
    m_equippedItem = CreateObject(itemID, vec3(playerPos.x + 0.8f, playerPos.y, 1.0f), false, 0.0f);
    
    m_equippedSlotIndex = slotIndex;
}

// In Update function - equipped item follows player
if (ecs.Registry.valid(m_equippedItem))
{
    auto* playerBody = ecs.Registry.try_get<physics::Body>(m_player);
    if (playerBody)
    {
        glm::vec2 playerPos = playerBody->GetPosition();
        auto& equippedTransform = ecs.Registry.get<Transform>(m_equippedItem);
        equippedTransform.SetTranslation(vec3(playerPos.x + 0.8f, playerPos.y, 1.0f));
    }
}
```

<video width="320" height="240" controls>
  <source src="/assets/article/buttons.mp4" type="video/mp4">
</video>

## Wrapping Up

That's the complete inventory system - from spawning items in the world to managing stacks, dragging between slots, and using/dropping/equipping them. 

Building this taught me a lot about UI systems, coordinate conversions, state management, and the importance of caching for performance. The system went through many iterations - from broken hover detection to flickering digits to drag operations that felt sluggish - but each problem led to a better solution.

If you're building your own inventory system from scratch, I hope this breakdown saves you some of the debugging headaches I ran into. The code examples here should give you a solid foundation, whether you're working with a custom engine like I am or adapting these concepts to Unity, Unreal, or another framework.

Thanks for reading, and happy coding!

<video width="320" height="240" controls>
  <source src="/assets/article/finalproduct.mp4" type="video/mp4">
</video> 

## References

1. SDL2 Development Library. Simple DirectMedia Layer. https://www.libsdl.org/
2. nlohmann. JSON for Modern C++. GitHub. https://github.com/nlohmann/json
3. Nystrom, R. *Game Programming Patterns: Component*. https://gameprogrammingpatterns.com/component.html
5. OpenGL.  The Industry's Foundation for High Performance Graphics. https://www.opengl.org/
6. https://gigi.nullneuron.net/gigilabs/sdl2-drag-and-drop/ - practical implementation guide showing SDL2's event-based approach with mouse button handling and offset tracking
