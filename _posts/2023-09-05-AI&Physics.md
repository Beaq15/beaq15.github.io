---
title: "AI&Physics"
categories: [Projects]
image: 
  path: /assets/ai&physics/picture.png
description: Ai agents that follow the player on a set navigation mesh
---

## General information

This was the first project I worked on in my second year of university. It was built using a provided template, without relying on any external engines. The agents navigate a mesh to move toward the player’s position. The code can be run in three different modes: with gravity applied to the agents, with the ability for the player to push them around, or with standard pathfinding behavior.

In the graph class, when implementing the A* search algorithm I used a priority queue for finding the shortest path between two vertices. It efficiently explores nodes based on their estimated total cost, so that the ones with the lowest estimated total cost are explored first. This is crucial for the algorithm so that the most promising path is found. It also helps with a faster search in such a way that the A* prioritizes exploring nodes that are likely to lead to the shortest path. 

```cpp
std::priority_queue<PathNode> open_set;
open_set.emplace(start_id, 0.0f, h_cost);

while (!open_set.empty())
{
    // Get the node with the lowest estimated total cost from the open set
    PathNode current = open_set.top();
    open_set.pop();
}
```
![](../assets/ai&physics/week2.png)

For the file reader I used tokenization by creating a vector of strings in which I added each item from the file, separated by space.  I then checked if the first token was w or o and went through all the lines, starting with the second item and added into a pointD variable pairs of 2 numbers, which represent the x and y of the points. I also created a CleanupGeometry() function which uses the difference function from the clipper library between the walkableArea and the obstacles. For the Triangulation() function i just insert all the vertices and edges using the cdt library. I draw the obstacles by going through all the paths by using the finalPath variable that i created in the CleanupGeometry().

```cpp
NavigationMesh::NavigationMesh(const std::string& filename)
{
    std::ifstream mapFile(filename);
    if (mapFile.is_open())
    {
        std::string item;
        std::string line;

        while (getline(mapFile, line))
        {
            std::stringstream str_strm;
            str_strm << line;
            std::vector<std::string> tokens;
            PathD polygonW;
            PathD polygonO;
            while (getline(str_strm, item, ' '))
            {
                tokens.push_back(item);
            }
            if (tokens[0] == "w")
            {
                for (int i = 1; i < tokens.size(); i++)
                {
                    if (i + 1 < tokens.size())
                    {
                        PointD point(stod(tokens[i]), stod(tokens[i + 1]));
                        polygonW.push_back(point);
                        i++;
                    }
                }
                walkablearea.push_back(polygonW);
            }
            if (tokens[0] == "o")
            {
                for (int i = 1; i < tokens.size(); i++)
                {
                    if (i + 1 < tokens.size())
                    {
                        PointD point(stod(tokens[i]), stod(tokens[i + 1]));
                        polygonO.push_back(point);
                        i++;
                    }
                }
                obstacles.push_back(polygonO);
            }
             mapFile.close()
        }
    }
```

![](../assets/ai&physics/week4.png)

Creating the agents: 
```cpp
for (auto agents : navigationSystem.GetNavigationMesh().GetAIPosition())
{
    {  // Create the agents
        auto entity = Engine.ECS().CreateEntity();

        auto& transform = Engine.ECS().CreateComponent<Transform>(entity);
        transform.Translation = vec3(agents.x, agents.y, 0);
        transform.Scale = vec3(0.4, 0.5, 0.2);
        Engine.ECS().CreateComponent<NavMeshAgent>(entity);
        auto& mesh_renderer = Engine.ECS().CreateComponent<SimpleMeshRender>(entity);
        mesh_renderer.mesh = Engine.Resources().Load<SimpleMesh>("meshes/cylinder.obj");
        mesh_renderer.texture = Engine.Resources().Load<SimpleTexture>("textures/red.png");
        auto rigidbody = Engine.ECS().CreateComponent<RigidBody>(entity, 1.0f, glm::vec2(transform.Translation.x, transform.Translation.y));
        auto diskCollider = Engine.ECS().CreateComponent<DiskCollider>(entity, 0.5f);
    }
}
```

I created a physics system and a class for the rigid body that I added as a component to the agents and player. In the physics loop, I go through all the agents and add a total force, then update their position and velocity using the force and the inverse mass. I clear the forces and in the end update their translation. In order to make this work, I also changed the navigation system so that it sets the linear velocity of the rigid body. Apart from this, i added a disk collider class which only had a getter for the radius and created it as a component for both the player and the agents. I then create a function that checks whether two disks collide or not and just draw a circle around them.

```cpp
void PhysicsSystem::ImpulseResolveCollision(RigidBody& a, RigidBody& b, const CollisionData& p)
{
    float total_mass = a.GetInverseMass() + b.GetInverseMass();

    a.SetPosition(a.GetPosition() + p.normal * p.depth * (a.GetInverseMass() / total_mass));
    b.SetPosition(b.GetPosition() - p.normal * p.depth * (b.GetInverseMass() / total_mass));

    glm::vec2 relativeVeocity = a.GetVelocity() - b.GetVelocity();

    float cRestitution = 0.66f;
    float dotproduct = glm::dot(relativeVeocity, p.normal);
    if (dotproduct <= 0)
    {
        float j = -(1 + cRestitution) * (dotproduct / total_mass);

        a.SetVelocity(a.GetVelocity() + a.GetInverseMass() * j * p.normal);
        b.SetVelocity(b.GetVelocity() - b.GetInverseMass() * j * p.normal);
    }
}
```

The CollisionData struct only stores the normal and depth and I changed the disk to disk collision detection to return those 2 variables. In the function I also made sure to calculate them. In the physics loop, I checked if the two objects collide by checking the value of data:

```cpp
auto data = DiskDiskCollision(body1.position, body2.position, diskCollider1.GetRadius(), diskCollider2.GetRadius());
if (data.depth > 0)
{
    ImpulseResolveCollision(body1, body2, data);
}
```

For the polygons it is mostly the same process, but for that I had to also create a getter for the obstacles in the navigation mesh. Because the obstacles are of type pathsd, I created a function that makes pathsd into polygons.

```cpp
const geometry2d::PolygonList NavigationMesh::MakePathsDIntoPolygons(PathsD pathsd) const
{
   geometry2d::PolygonList polygonList;
   for (int i = 0; i < pathsd.size(); i++)
   {
       geometry2d::Polygon polygon;
       for (int j = 0; j < pathsd[i].size(); j++)
       {
           polygon.push_back(glm::vec2(pathsd[i][j].x, pathsd[i][j].y));
       }
           polygonList.push_back(polygon);
   }
   return polygonList;
}
```

I am also making all the obstacles in the game class by creating a polygonCollider component. In the physics system, I just go through all the bodies, and then the polygons and just resolve the collision.

My code can do three things in total:

-apply gravity to the agents
<img src = "../assets/ai&physics/gravity.gif"></img>
-
-push the agents around
<img src = "../assets/ai&physics/pushing.gif"></img>
-
-classic path following
<img src = "../assets/ai&physics/normalbehaviour.gif"></img>
-