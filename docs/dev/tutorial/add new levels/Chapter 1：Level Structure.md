---
sidebar_position: 0
---
# Chapter One: Level Structure

:::info
### Adding New Levels: Three Chapters Planned
- Chapter One: Constructing Level Structures
- Chapter Two: Constructing Map Environments
- Chapter Three: Random Structures
:::
:::tip
This chapter assumes you have the following prerequisites:
1. Experience modifying `data.cdb`.

2. Foundational knowledge of the `script` module.


- If you are familiar with the `script` module, this chapter will be relatively straightforward for you.
- `core modding` offers greater modification privileges than `script`.

:::
:::warning
The latest `core modding` framework is required.
As the framework was released only recently, only a handful of individuals are currently researching mods for this framework. We hope you can conduct further testing and refine this tutorial.
:::

## 一.Preparatory Work
### 1. Create the `dccm` module
[Click to view the tutorial](i18n/zh/docusaurus-plugin-content-docs/current/dev/tutorial/first-mod.md)
```csharp
using ModCore.Events.Interfaces.Game;
using ModCore.Mods;
namespace Outside_Clock
{
    public class Mian : ModBase,
        IOnGameExit
    {
        public Mian(ModInfo info) : base(info)
        {
        }
        
        public override void Initialize()
        {
            Logger.Information("你好，世界");
        }

        void IOnGameExit.OnGameExit()

        {
            Logger.Information("游戏正在退出");
        }

    }

}
```


### 2.Create a new class

```csharp
using System;
namespace Outside_Clock;

public class Out_Clock
{

}
```
:::warning 
 The class name here: `Out_Clock` will be your level's `id`.
:::  

### 3.`data.cdb` Configuration
Extract a brand-new `data.cdb` file into your project's root directory.

![](./level/strct-1.png)

#### 3.1 Create a new level in the `level` column of the `cdb`.
![](./level/strct-cdb.png)
:::warning

 The level created has an `id` corresponding to the class name of your newly created class.
 
 After adding a level, you must add a hook.
 ``` csharp
 Hook_LevelLogos.getLevelLogo += Hook_LevelLogos_getLevelLogo;
 
 private Tile Hook_LevelLogos_getLevelLogo(Hook_LevelLogos.orig_getLevelLogo orig, LevelLogos self, dc.String levelLogoCoordinate)
        {
            if (!self.textureCoordinateByLevelKind.exists.Invoke(levelLogoCoordinate))
            {
                return orig(self, "ClockTower".AsHaxeString());
            }
            else return orig(self, levelLogoCoordinate);
        }
 ```
 Adding your level to the world map: (~~I'm too lazy~~ Haven't figured it out yet, so using the clock tower as a placeholder for now)
 ![](./level/stuct-logo.png)
 Otherwise, an error will occur!
:::

:::info
Since I wish to create an outdoor environment without external walls, I have selected `Outside`.

These options may also be configured within the code (as covered later), provided you have opened it in `cdb`.

Additionally, for instructional purposes, we shall refrain from adding monsters or items for now.

Other options may be explored at your discretion.
:::

### 4.Configuring the `csproj`
is your project file.

![](./level/strct-2.png)


```bash
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>

    <TargetFramework>net9.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <ModType>mod</ModType>
    <ModName>Outside_Clock</ModName>
    <ModMain>Outside_Clock.Mian</ModMain>
    <AutoInstallMod Condition="'$(Configuration)' == 'Debug'">true</AutoInstallMod>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>

  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="AWSSDK.DynamoDBv2" Version="4.0.6.2" />
    <PackageReference Include="DeadCellsCoreModding.MDK" Version="1.0.1" />
  </ItemGroup>

  <PropertyGroup>
    <GenerateDiffCDB>true</GenerateDiffCDB>
    <GameVersion>35</GameVersion>
  </PropertyGroup>
</Project>

```

:::info
```
<PropertyGroup>
    <GenerateDiffCDB>true</GenerateDiffCDB>
    <GameVersion>35</GameVersion>
</PropertyGroup> 
```
Function: Automatically packages the `cdb` file within the project directory into a `pak` file.
:::

### 5.Reference the `pak` file

```csharp
using dc.tool.mod;
using ModCore.Events.Interfaces.Game;
using ModCore.Mods;
using ModCore.Modules;
using ModCore.Utitities;

namespace Outside_Clock
{
    public class Mian : ModBase,
        IOnGameExit,
        IOnGameEndInit
    {
        public Mian(ModInfo info) : base(info)
        {
        }

        public override void Initialize()
        {
            Logger.Information("你好，世界");
            Hook_LevelLogos.getLevelLogo += Hook_LevelLogos_getLevelLogo;
        }
        void IOnGameExit.OnGameExit()
        {
            Logger.Information("游戏正在退出");
        }
        void IOnGameEndInit.OnGameEndInit()
        {
            var res = Info.ModRoot!.GetFilePath("Kaguya.pak");
            FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
            var json = CDBManager.Class.instance.getAlteredCDB();
            dc.Data.Class.loadJson(
               json,
               default);
        }
    }
}
```

:::info 
```csharp
void IOnGameEndInit.OnGameEndInit()
        {
            var res = Info.ModRoot!.GetFilePath("Kaguya.pak");
            FsPak.Instance.FileSystem.loadPak(res.AsHaxeString());
            var json = CDBManager.Class.instance.getAlteredCDB();
            dc.Data.Class.loadJson(
               json,
               default);
        }
```
Thus your `pak` is read.
:::

### 6.Modifications for testing purposes
Create a new scripts module
```
function buildMainRooms() {

    Struct.createRoomWithType("Entrance")
        .setName("start")
        .chain(Struct.createExit("SampleLevel"))
        .chain(Struct.createExit("Out_Clock"))
        .chain(Struct.createExit("PrisonRoof"));
}
```
[Step-by-step guide]([死亡细胞模组制作教程 第四部分（Script模组） - 哔哩哔哩](https://www.bilibili.com/opus/684984246732849152))
### The preparatory work is now complete.

## 二. Constructing the Level Structure
### 1. Creating the Main Rooms
 
```csharp
using System;
using dc;
using dc.level;
using dc.libs;
using Hashlink.Virtuals;

namespace Outside_Clock;

public class Out_Clock : LevelStruct
{
    public Out_Clock(User user, virtual_baseLootLevel_biome_bonusTripleScrollAfterBC_cellBonus_dlc_doubleUps_eliteRoomChance_eliteWanderChance_flagsProps_group_icon_id_index_loreDescriptions_mapDepth_minGold_mobDensity_mobs_name_nextLevels_parallax_props_quarterUpsBC3_quarterUpsBC4_specificLoots_specificSubBiome_transitionTo_tripleUps_worldDepth_ level, Rand rng) : base(user, level, rng)
    {
    }
    
    public override RoomNode buildMainRooms()
    {
         RoomNode entranceNode = base.createNode(null, "TU_BasicEntrance".AsHaxeString(), null, "start".AsHaxeString());
        entranceNode.addFlag(new RoomFlag.Outside());
        
        var forcedBiome = "ClockTower".AsHaxeString();
        var virtual_add = new virtual_specificBiome_();
        virtual_add.specificBiome = forcedBiome;
        var clockTowerGenData = virtual_add.ToVirtual<virtual_altarItemGroup_brLegendaryMultiTreasure_broken_cells_doorCost_doorCurse_flaskRefill_forcedMerchantType_forcePauseTimer_isCliffPath_itemInWall_itemLevelBonus_killsMultiTreasure_locked_maxPerks_mins_noHealingShop_shouldBeFlipped_specificBiome_subTeleportTo_timedMultiTreasure_zDoorLock_zDoorType_>();
        entranceNode.addGenData(clockTowerGenData);
        
        RoomNode exitNode = base.createExit("ClockTower".AsHaxeString(), "RoofEndExit".AsHaxeString(), null, "end".AsHaxeString())
        .addFlag(new RoomFlag.Outside())
        exitNode.set_parent(entranceNode);
    }
}
```

- First, inherit from `LevelStruct` and override the `buildMainRooms` method

- Create the level entrance room: `RoomNode entranceNode = base.createNode(null, ‘TU_BasicEntrance’.AsHaxeString(), null, ‘start’.AsHaxeString());`

- Create the level exit room: `RoomNode exitNode = base.createExit(‘ClockTower’.AsHaxeString(), “RoofEndExit”.AsHaxeString(), null, ‘end’.AsHaxeString())`

-

:::info

- `createNode`: Places your room file and sets some configurations.

- `‘TU_BasicEntrance’` : Your room file.

- `start` and `end` : Entry tags.

- `addFlag` : Configuration for added rooms.

- `set_parent` : Connects rooms.

- `entranceNode.addGenData(clockTowerGenData);` Sets your level's environment template.

- `createExit` : Creates the level exit room.

- As I've ticked `Outside` in `data.cdb`, we must append this tag to each added room:

`.addFlag(new RoomFlag.Outside())`

- The remainder you may explore independently.

Below are some notes I've compiled for reference:




| 名称                      | 作用                                                                                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **ForceCollExtBottom**​ | The bottom of the room is enforced with an external collision boundary, restricting player movement.                                        |
| **ForceCollExtTop**​    | Forced rooms possess an external collision boundary at the top, often utilised in conjunction with `Bottom` to create a sense of enclosure. |
| **Holes**​              | Permit the generation of pitfall traps within rooms, causing players to sustain injury or death upon falling into them.                     |
| **Indexes**​            | Not required                                                                                                                                |
| **InsideOut**​          | Rooms labelled 'inside-out' may be employed to create distinctive visual effects or a sense of spatial disorientation.                      |
| **LockedNeedCard**​     | Not tested                                                                                                                                  |
| **LockedNeedKey**​      | Not tested                                                                                                                                  |
| **NoCritters**​         | Not tested                                                                                                                                  |
| **NoExitSizeCheck**​    | Disable the dimensional validity check for exits from this room, for use with special exits.                                                |
| **NoLoot**​             | No loot may be generated within the room (such as treasure chests, gold coins, or items).                                                   |
| **Outside**​            | Mark this room as an 'outdoor' area, affecting environmental systems such as sound effects and lighting.                                    |
| **TemplateFlip**​       | Not tested                                                                                                                                  |

:::
#### 1.1 Add main room

```csharp
public override RoomNode buildMainRooms()
    {
        RoomNode entranceNode = base.createNode(null, "TU_BasicEntrance".AsHaxeString(), null, "start".AsHaxeString());
        entranceNode.addFlag(new RoomFlag.Outside());
        entranceNode.addFlag(new RoomFlag.NoExitSizeCheck());
        entranceNode.addFlag(new RoomFlag.Holes());
        entranceNode.setConstraint(new LinkConstraint.NeverUp());

        var forcedBiome = "ClockTower".AsHaxeString();
        var virtual_add = new virtual_specificBiome_();
        virtual_add.specificBiome = forcedBiome;
        var clockTowerGenData = virtual_add.ToVirtual<virtual_altarItemGroup_brLegendaryMultiTreasure_broken_cells_doorCost_doorCurse_flaskRefill_forcedMerchantType_forcePauseTimer_isCliffPath_itemInWall_itemLevelBonus_killsMultiTreasure_locked_maxPerks_mins_noHealingShop_shouldBeFlipped_specificBiome_subTeleportTo_timedMultiTreasure_zDoorLock_zDoorType_>();
        entranceNode.addGenData(clockTowerGenData);


        RoomNode combatNode = base.createNode(null, "OutsideTower4".AsHaxeString(), null, null)
            .addFlag(new RoomFlag.Outside())
            .addFlag(new RoomFlag.Holes());

        combatNode.set_parent(entranceNode);

  
        RoomNode demonNode = base.createNode(null, "OutsideCross1".AsHaxeString(), null, "exit".AsHaxeString())
            .addFlag(new RoomFlag.Outside())
            .addFlag(new RoomFlag.Holes())
            .addNpc(new NpcId.CryptDemon());

        demonNode.set_parent(combatNode);

  
        RoomNode demonNode1 = base.createNode(null, "Teleport_UD".AsHaxeString(), null, null)
            .addFlag(new RoomFlag.Outside())
            .addFlag(new RoomFlag.Holes())
            .addNpc(new NpcId.CryptDemon());

        demonNode1.set_parent(demonNode);

  
        RoomNode demonNode2 = base.createNode(null, "Teleport_UD".AsHaxeString(), null, null)
            .addFlag(new RoomFlag.Outside())
            .addFlag(new RoomFlag.Holes())
            .addNpc(new NpcId.CryptDemon());

        demonNode2.set_parent(demonNode1);

        RoomNode knightNode = base.createNode(null, "RoofSpacer1".AsHaxeString(), null, null)
            .addFlag(new RoomFlag.DarkRoom())
            .addFlag(new RoomFlag.Outside())
            .addFlag(new RoomFlag.Holes())
            .addNpc(new NpcId.Knight());

        knightNode.set_parent(demonNode);
  

        RoomNode exitNode = base.createExit("ClockTower".AsHaxeString(), "RoofEndExit".AsHaxeString(), null, "end".AsHaxeString())
        .addFlag(new RoomFlag.Outside())
        .addFlag(new RoomFlag.Holes());
        exitNode.set_parent(knightNode);

        return base.nodes.get("start".AsHaxeString());

    }
```
- I have added more rooms to the main route of the level, along with a fork in the path: `demonNode1` and `demonNode2`.
- Two new NPC types have been added: `addNpc(new NpcId.Knight());` and `addNpc(new NpcId.CryptDemon());`. 

:::warning
When adding branch junctions, take note: does the referenced room file possess a corresponding entrance?
:::
### 2.Create a secondary room

```csharp
public override void buildSecondaryRooms()

    {
        base.buildSecondaryRooms();
        RoomNode roomNode = base.createNode("Combat".AsHaxeString(), null, 69, null).addFlag(new RoomFlag.Outside()).addBefore(base.getId("exit".AsHaxeString()), null);
        roomNode = base.createNode("Combat".AsHaxeString(), null, 6, null).addFlag(new RoomFlag.Outside()).addBefore(base.getId("exit".AsHaxeString()), null);
        roomNode = base.createNode("Combat".AsHaxeString(), null, 7, null).addFlag(new RoomFlag.Outside()).addBefore(base.getId("exit".AsHaxeString()), null);
    }
```

- Override the `buildSecondaryRooms` method

- Add secondary rooms: Add three `Combat` rooms. Note the number in the third parameter represents an internal reference, where each ID denotes a distinct combat room

- `.addBefore(base.getId(‘exit’.AsHaxeString()),` places the room before the room named `exit` in the current level
  
  
## 三.Example
This is the map code I'm currently developing; feel free to use it as a reference (not the final version).

``` csharp
namespace Outside_Clock;

public class Out_Clock : LevelStruct
{

    public Out_Clock(User user, virtual_baseLootLevel_biome_bonusTripleScrollAfterBC_cellBonus_dlc_doubleUps_eliteRoomChance_eliteWanderChance_flagsProps_group_icon_id_index_loreDescriptions_mapDepth_minGold_mobDensity_mobs_name_nextLevels_parallax_props_quarterUpsBC3_quarterUpsBC4_specificLoots_specificSubBiome_transitionTo_tripleUps_worldDepth_ lInfos, Rand rng) : base(user, lInfos, rng)
    {

    }

    public override RoomNode buildMainRooms()
    {

        Rand rng = base.rng;
        double randnumber = rng.seed * 16807.0 % 2147483647.0;
        rng.seed = randnumber;
        bool randbool = ((int)randnumber & 1073741823) % 100 < 70;

        #region 入口
        RoomNode entranceNode = base.createNode(null, "RoofEntrance".AsHaxeString(), null, "start".AsHaxeString());
        entranceNode.AddFlags(new RoomFlag.NoExitSizeCheck());
        entranceNode.setConstraint(new LinkConstraint.RightOnly());

        var forcedBiome = "ClockTower".AsHaxeString();
        var virtual_add = new virtual_specificBiome_();
        virtual_add.specificBiome = forcedBiome;
        var clockTowerGenData = virtual_add.ToVirtual<virtual_altarItemGroup_brLegendaryMultiTreasure_broken_cells_doorCost_doorCurse_flaskRefill_forcedMerchantType_forcePauseTimer_isCliffPath_itemInWall_itemLevelBonus_killsMultiTreasure_locked_maxPerks_mins_noHealingShop_shouldBeFlipped_specificBiome_subTeleportTo_timedMultiTreasure_zDoorLock_zDoorType_>();


        entranceNode.addGenData(clockTowerGenData);
        #endregion

        #region 入口向右连接
        RandList rand = new RandList(new HlFunc<int, int>(base.rng.random), (ArrayDyn)null!);
        double randomValue = base.rng.random(100) / 100.0;
        if (randomValue < 0.4)
        {
            RoomNode combatNode = base.createNode(null, "TU_teleportSecret".AsHaxeString(), null, "right".AsHaxeString())
                        .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())
                        .setConstraint(new LinkConstraint.RightOnly());

            combatNode.set_parent(entranceNode);
        }
        else
        {
            RoomNode combatNode = base.createNode("Combat".AsHaxeString(), null, null, "right".AsHaxeString())
                        .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())
                        .setConstraint(new LinkConstraint.RightOnly());

            combatNode.set_parent(entranceNode);
        }

        #endregion




        #region 入口后向上连接

        RoomNode up = base.createNode("Teleport".AsHaxeString(), null, null, "upOnly".AsHaxeString())
           .setConstraint(new LinkConstraint.All());


        up.set_parent(entranceNode);

        #endregion






        #region 入口后向左连接

        RoomNode combatNode1 = base.createNode("Teleport".AsHaxeString(), null, null, "left".AsHaxeString())
            .addFlag(new RoomFlag.Outside())
            .addFlag(new RoomFlag.Holes())
            .setConstraint(new LinkConstraint.LeftOnly());

        combatNode1.set_parent(entranceNode);
        #endregion



        return base.nodes.get("start".AsHaxeString());
    }

    public override void addTeleports()
    {

    }

    #region 约束配置
    public override void finalize()
    {
        // int num = 0;
        // ArrayObj all = base.all;
        // for (; ; )
        // {
        //     int length = all.length;
        //     if (num >= length)
        //     {
        //         break;
        //     }
        //     length = all.length;
        //     RoomNode? roomNode;
        //     if (num >= length)
        //     {
        //         roomNode = null;
        //     }
        //     else
        //     {

        //         roomNode = all.array[num] as RoomNode;
        //     }
        //     num++;
        //     if (roomNode != null && roomNode.parent != null)
        //     {
        //         if ((roomNode.flags & 8) != 0)
        //         {
        //             roomNode.childPriority = 1;

        //             RoomNode roomNode2 = roomNode.setConstraint(new LinkConstraint.NeverUp());
        //         }
        //         else if ((roomNode.parent.flags & 8) != 0)
        //         {

        //             RoomNode roomNode2 = roomNode.setConstraint(new LinkConstraint.NeverUp());
        //         }
        //     }
        // }


    }
    #endregion



    public override void buildSecondaryRooms()
    {
        base.buildSecondaryRooms();

        #region 配置
        dc.String upOnlyId = "upOnly".AsHaxeString();
        dc.String a2Id = "left".AsHaxeString();
        dc.String boss = "bossbefore".AsHaxeString();

        RoomNode upOnlyNode = base.getId(upOnlyId);
        RoomNode a2Node = base.getId(a2Id);

        #endregion


        RoomNode roomNode = base.createNode("Combat".AsHaxeString(), null, null, null)
            .setConstraint(new LinkConstraint.All());

        // #region 出口前支线
        // roomNode = base.createNode("Combat".AsHaxeString(), null, null, null)
        //     .addFlag(new RoomFlag.Outside())
        //     .addBefore(exitNode, null);


        // int[] exitCombatData = new int[] { 69, 1, 7, 3, 2, 6, 1 };
        // int i = 0;
        // for (; ; )
        // {
        //     if (i >= exitCombatData.Length) break;
        //     int data = exitCombatData[i];
        //     i++;

        //     roomNode = base.createNode(null, "TU_teleportSecret".AsHaxeString(), null, null)
        //     .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())
        //         .addBefore(exitNode, null);

        //     roomNode = base.createNode("Combat".AsHaxeString(), null, null, null)
        //        .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())
        //         .addBefore(exitNode, null);
        // }


        // roomNode = base.createNode("Shop".AsHaxeString(), null, null, null)
        //    .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())
        //     .addBefore(exitNode, null);
        // #endregion




        #region 入口向上连接支线
        int i = 0;
        for (; ; )
        {
            if (i >= 3) break;
            i++;

            RoomNode roomNodebetween1 = base.createNode("Teleport".AsHaxeString(), null, null, null)
            .AddFlags(new RoomFlag.Holes())
            .setConstraint(new LinkConstraint.UpOnly())
            .addBefore(upOnlyNode, null);


            RoomNode roomNodebetween = base.createNode("Combat".AsHaxeString(), null, null, null)
            .AddFlags(new RoomFlag.Holes())
            .setConstraint(new LinkConstraint.UpOnly())
            .addBefore(upOnlyNode, null);



            double randomValue = base.rng.random(100) / 100.0;
            if (randomValue > 0.5)
            {
                RoomNode roomNodebetween2 = base.createNode(null, "CT_VSpacer".AsHaxeString(), null, null)
                           .AddFlags(new RoomFlag.Holes())
                           .setConstraint(new LinkConstraint.UpOnly())
                           .addBefore(upOnlyNode, null);

            }
            else
            {
                RoomNode roomNodebetween2 = base.createNode("WallJumpGate".AsHaxeString(), null, null, null)
                           .AddFlags(new RoomFlag.Holes())
                           .setConstraint(new LinkConstraint.UpOnly())
                           .addBefore(upOnlyNode, null);
            }

        }


        roomNode = base.createNode(null, "BR_BossTimeKeeper".AsHaxeString(), null, "bossbefore".AsHaxeString())
         .AddFlags(new RoomFlag.Holes())
         .setConstraint(new LinkConstraint.All())
         .addBefore(upOnlyNode, null);




        RoomNode bossbefoer = base.getId(boss);
        roomNode = base.createNode(null, "CT_Key".AsHaxeString(), null, null)
           .addFlag(new RoomFlag.Holes())
           .setConstraint(new LinkConstraint.All())
           .addAfter(bossbefoer, null);

        #endregion




        RandList categoryList = new RandList(new HlFunc<int, int>(base.rng.random), null);
        categoryList.add("Combat", 2);
        categoryList.add("CursedTreasure", 2);



        roomNode = base.createNode(null, "TU_teleport0".AsHaxeString(), null, null)
            .setConstraint(new LinkConstraint.LeftOnly())
            .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())

              .addBefore(a2Node, null);



        roomNode = base.createNode(null, "SwCombat1".AsHaxeString(), null, null)
            .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())
            .setConstraint(new LinkConstraint.LeftOnly())

            .addBefore(a2Node, null);


        #region 左连接支线
        int[] a2NodeLeftOnly = new int[] { 93, 93, 93 };
        i = 0;
        for (; ; )
        {
            if (i >= a2NodeLeftOnly.Length) break;
            int data = a2NodeLeftOnly[i];
            i++;
            dc.String category = categoryList.draw(null);

            roomNode = base.createNode(category, /*"TU_teleport0"*/null, null, null)
            .setConstraint(new LinkConstraint.LeftOnly())
            .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())

            .addBefore(a2Node, null);


            roomNode = base.createNode("Combat".AsHaxeString(), null, data, null)
                .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())
                .setConstraint(new LinkConstraint.LeftOnly())

                .addBefore(a2Node, null);

            roomNode = base.createNode(null, "TU_teleport0".AsHaxeString(), null, null)
                .setConstraint(new LinkConstraint.LeftOnly())
                .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())

            .addBefore(a2Node, null);

        }

        roomNode = base.createNode("CursedTreasure".AsHaxeString(), null, null, null)
            .AddFlags(new RoomFlag.Outside(), new RoomFlag.Holes())
            .setConstraint(new LinkConstraint.LeftOnly())
            .addBefore(a2Node, null);

        #endregion


    }
}



```