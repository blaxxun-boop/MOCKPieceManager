﻿# Piece Manager

Can be used to easily add new building pieces to Valheim. Will automatically add config options to your mod and sync the
configuration from a server, if the mod is installed on the server as well.

## How to add pieces

Copy the asset bundle into your project and make sure to set it as an EmbeddedResource in the properties of the asset
bundle. Default path for the asset bundle is an `assets` directory, but you can override this. This way, you don't have
to distribute your assets with your mod. They will be embedded into your mods DLL.

### Merging the DLLs into your mod

Download the PieceManager.dll and the ServerSync.dll from the release section to the right. Including the DLLs is best
done via ILRepack (https://github.com/ravibpatel/ILRepack.Lib.MSBuild.Task). You can load this package (
ILRepack.Lib.MSBuild.Task) from NuGet.

If you have installed ILRepack via NuGet, simply create a file named `ILRepack.targets` in your project and copy the
following content into the file

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <Target Name="ILRepacker" AfterTargets="Build">
        <ItemGroup>
            <InputAssemblies Include="$(TargetPath)"/>
            <InputAssemblies Include="$(OutputPath)\PieceManager.dll"/>
            <InputAssemblies Include="$(OutputPath)\ServerSync.dll"/>
        </ItemGroup>
        <ILRepack Parallel="true" DebugInfo="true" Internalize="true" InputAssemblies="@(InputAssemblies)"
                  OutputFile="$(TargetPath)" TargetKind="SameAsPrimaryAssembly" LibraryPath="$(OutputPath)"/>
    </Target>
</Project>
```

Make sure to set the PieceManager.dll and the ServerSync.dll in your project to "Copy to output directory" in the
properties of the DLLs and to add a reference to it. After that, simply add `using PieceManager;` to your mod and use
the `BuildPiece` class, to add your items.

## Example project

This adds three different pieces from two different asset bundles. The `funward` asset bundle is in a directory
called `FunWard`, while the `bamboo` asset bundle is in a directory called `assets`.

```csharp
using System.IO;
using BepInEx;
using HarmonyLib;
using PieceManager;

namespace PieceManagerExampleMod
{
    [BepInPlugin(ModGUID, ModName, ModVersion)]
    public class PieceManagerExampleMod : BaseUnityPlugin
    {
        private const string ModName = "PieceManagerExampleMod";
        private const string ModVersion = "1.0.0";
        internal const string Author = "azumatt";
        private const string ModGUID = Author + "." + ModName;
        private static string ConfigFileName = ModGUID + ".cfg";
        private static string ConfigFileFullPath = Paths.ConfigPath + Path.DirectorySeparatorChar + ConfigFileName;

        private void Awake()
        {
            // Globally turn off configuration options for your pieces, omit if you don't want to do this.
            BuildPiece.ConfigurationEnabled = false;
            
            // Format: new("AssetBundleName", "PrefabName", "FolderName");
            BuildPiece examplePiece1 = new("funward", "funward", "FunWard");

            examplePiece1.Name.English("Fun Ward"); // Localize the name and description for the building piece for a language.
            examplePiece1.Description.English("Ward For testing the Piece Manager");
            examplePiece1.RequiredItems.Add("FineWood", 20, false); // Set the required items to build. Format: ("PrefabName", Amount, Recoverable)
            examplePiece1.RequiredItems.Add("SurtlingCore", 20, false);
            examplePiece1.Category.Set(BuildPieceCategory.Misc);
            //examplePiece1.Category.Set("CUSTOMCATEGORY"); // You can set a custom category for your piece. Instead of the default ones like above.
            examplePiece1.Crafting.Set(CraftingTable.ArtisanTable); // Set a crafting station requirement for the piece.
            examplePiece1.Extension.Set(CraftingTable.Forge, 2); // Makes this piece a station extension, can change the max station distance by changing the second value. Use strings for custom tables.
            //examplePiece1.Crafting.Set("CUSTOMTABLE"); // If you have a custom table you're adding to the game. Just set it like this.
            //examplePiece1.SpecialProperties.NoConfig = true;  // Do not generate a config for this piece, omit this line of code if you want to generate a config.
            examplePiece1.SpecialProperties = new SpecialProperties() { AdminOnly = true, NoConfig = true}; // You can declare multiple properties in one line           
            
            // Add your piece to the hammer, cultivator or other.
            examplePiece1.Tool.Add("Cultivator"); // Format: yourvariable.Tool.Add("Item that has a piecetable")

            BuildPiece examplePiece2 = new("bamboo", "Bamboo_Wall"); // Note: If you wish to use the default "assets" folder for your assets, you can omit it!
            examplePiece2.Name.English("Bamboo Wall");
            examplePiece2.Description.English("A wall made of bamboo!");
            examplePiece2.RequiredItems.Add("BambooLog", 20, false);
            examplePiece2.Category.Set(BuildPieceCategory.Building);
            examplePiece2.Crafting.Set(CraftingTable.ArtisanTable);
            examplePiece2.SpecialProperties.AdminOnly = true;  // You can declare these one at a time as well!.


            // If you want to add your item to the Cultivator or another hammer
            BuildPiece examplePiece3 = new("bamboo", "Bamboo_Sapling");
            examplePiece3.Name.English("Bamboo Sapling");
            examplePiece3.Description.English("A young bamboo tree, called a sapling");
            examplePiece3.Category.Set(BuildPieceCategory.Building);
            examplePiece3.Crafting.Set(CraftingTable.Workbench);
            examplePiece3.RequiredItems.Add("BambooSeed", 20, false);
            examplePiece3.Tool.Add("Cultivator"); // Format: ("Item that has a piecetable")
            examplePiece3.SpecialProperties.NoConfig = true;

            // Need to add something to ZNetScene but not the hammer, cultivator or other? 
            PiecePrefabManager.RegisterPrefab("bamboo", "Bamboo_Beam_Light");
            
            // Does your model need to swap materials with a vanilla material? Format: (GameObject, isJotunnMock)
            MaterialReplacer.RegisterGameObjectForMatSwap(examplePiece3.Prefab, false);
            
            // Does your model use a shader from the game like Custom/Creature or Custom/Piece in unity? Need it to "just work"?
            MaterialReplacer.RegisterGameObjectForShaderSwap(examplePiece3.Prefab, MaterialReplacer.ShaderType.UseUnityShader);
            
            // What if you want to use a custom shader from the game (like Custom/Piece that allows snow!!!) but your unity shader isn't set to Custom/Piece? Format: (GameObject, MaterialReplacer.ShaderType.)
            //MaterialReplacer.RegisterGameObjectForShaderSwap(examplePiece3.Prefab, MaterialReplacer.ShaderType.PieceShader);

            // Detailed instructions on how to use the MaterialReplacer can be found on the current PieceManager Wiki. https://github.com/AzumattDev/PieceManager/wiki
            
            //If you need to add snappoints to your piece, you can use the SnapPointMaker class.
            // Detailed instructions on how to use the SnapPointMaker can be found on the current PieceManager Wiki. https://github.com/AzumattDev/PieceManager/wiki
            SnapPointMaker.AddObjectForSnapPoints(examplePiece3.Prefab);
            SnapPointMaker.ApplySnapPoints(SnapPointType.Vertices | SnapPointType.Center);
        }
    }
}
```

# SnapPointMaker

## Overview

`SnapPointMaker` is a static class used to create snap points on `GameObject`s in Unity. A snap point is a specific point on a `GameObject` where other objects can "snap" or attach to, based on a game's mechanics.

Supported collider types are `BoxCollider` and `SphereCollider`. You can specify the types of snap points you want to create using the `SnapPointType` enumeration.

## Usage

### Add Object for Snap Points

To register a `GameObject` for snap points creation, use the `AddObjectForSnapPoints` method:

```csharp
SnapPointMaker.AddObjectForSnapPoints(myGameObject);
```

### Apply Snap Points

To apply snap points to all registered `GameObject`s, use the `ApplySnapPoints` method:

```csharp
SnapPointMaker.ApplySnapPoints(SnapPointType.Vertices | SnapPointType.Center);
```

This example will apply snap points at the vertices and the center of the registered game objects, given they have either a `BoxCollider` or a `SphereCollider` component.

## SnapPointType Enumeration

`SnapPointType` is an enumeration used to specify the types of snap points to create:

* `Vertices`: Snap points will be created at the vertices of a `BoxCollider`.
* `Center`: A snap point will be created at the center of the collider.
* `Poles`: Snap points will be created at the top and bottom of a `SphereCollider`.
* `Equator`: Snap points will be created at the cardinal directions on the equator of a `SphereCollider`.

These enumeration values can be combined using the bitwise OR operator (`|`). For example, `SnapPointType.Vertices | SnapPointType.Center` will create snap points at both the vertices and the center.

## Note

The `SnapPointMaker` currently supports `BoxCollider` and `SphereCollider` types. If your `GameObject` has a different type of collider, you will need to provide additional logic to determine the snap points. The `Unsupported collider type` warning will be displayed in the console for unsupported collider types.

## Improvements

The snap points created by the `SnapPointMaker` are static and do not adjust dynamically if the `GameObject` changes shape or size. If you require dynamic snap points, you'd have to implement a system that updates snap points in response to changes to the `GameObject`.