# FBX test version file format specification

*This post is extract from [blender Doc :FBX File Structure](https://archive.blender.org/wiki/index.php/User:Mont29/Foundation/FBX_File_Structure/).*

## Generalities

FBX is a nodal format, with a root element (which is never explicitly written) and a tree of children.

Each element has an id (byte string), and can have data and children elements.

Data are a set of (values, type) tuples, available types are: bool, short, int, long, float, double, bytes, string, and arrays of those types.

Here is a (JSON) representation of an element:

```json
["Element_ID",                         # ID
     ["data_string", 13],              # Data
     "SI",                             # Data types, as single-char codes (S for String, I for Integer, etc.)
     [__other_children_elements__]
]
```

Everything is based on this simple schema.

## Properties

Properties are a way to add heavily typed data.

Properties are represented by elements children of a "Properties70" element, which does not contain any data.

Each property is an element. Its ID does not seem to be important (usually, it's "P" or "PS" for all of them). Their data layout follow that schema:

```JSON
["PropName", "PropType", "Label(?)", "Flags", __values__, …]
```

In other words, the four first data of a property are always strings (its name, its type, presumably its label (often empty), and some flags (optional)), they are the "metadata" of the property. The other data are the property's value (usually only one, but e.g. for colors or 3D vectors they are three - and some property types have no value :/), and their type depends on the data type!

Bellow are some basic examples of properties:

```JSON
Integer: ["P", ["some_name", "int", "Integer", "", 1], "SSSSI", []]
Double:  ["P", ["some_name", "double", "Number", "", 1.0], "SSSSD", []]
Color:   ["P", ["some_name", "ColorRGB", "Color", "", 0.0, 0.0, 0.0], "SSSSDDD", []]
```

## Templates

Each template is defined by an "ObjectType" element, which takes a single property, the name of the template, which is also the name of the Object they “define” (e.g. "Model", "Geometry", "Material", etc.).

Each template has two children: "Count" is the number of Objects using this template, and "PropertyTemplate", which simply contains some properties.

To summarize, here is the generic structure of a template definition:

```JSON
["ObjectType", ["Geometry"], "S", [            # Start of template definition
    ["Count", [1], "I", []],                   # Number of Objects using this template
    ["PropertyTemplate", ["KFbxMesh"], "S", [  # Start of template's properties
        ["Properties70", [], "", [
            ["P", ["Color", "ColorRGB", "Color", "", 0.8, 0.8, 0.8], "SSSSDDD", []],
            ["P", ["BBoxMin", "Vector3D", "Vector", "", 0.0, 0.0, 0.0], "SSSSDDD", []],
            ["P", ["BBoxMax", "Vector3D", "Vector", "", 0.0, 0.0, 0.0], "SSSSDDD", []],
            ...
        ]]
    ]]
]],
```

## Top Structure

A valid FBX file must contain a set of standard elements:

-   **FBXHeaderExtension**: Mandatory? Contains the metadata of the file.
-   **FileId**: Mandatory. Some kind of (currently) obscure hash, apparently based on the CreationTime data.
-   **CreationTime**: Mandatory. The date/time of creation of that file, as a single string (e.g. "2013-08-05 10:27:44:000").
-   **Creator**: Mandatory? A string identifying the tool used to create that FBX.
-   **GlobalSettings**: Mandatory? A set of properties defining general data, like the orientation of the scene…
-   **Documents**: Optional. Not sure what it is used for actually, so far only saw FBX files with a single doc definition here… Maybe it allows for several scenes in a single FBX?
-   **References**: ???
-   **Definitions**: Optional. Here templates are defined.
-   **Objects**: Optional. Here real, useful, static (?) data are stored (objects, geometries, textures, materials, etc.).
-   **Connections**: Optional. Here are defined links between data defined in Objects (e.g. which object uses which material, etc.).
-   **Takes**: Optional. Here animations are defined?

## Main Data - Objects

### Mesh Data

```JSON
["Geometry", [152167664, "Name::Geometry", "Mesh"], "LSS", [
     ["Vertices", [[<array_of_floats>]], "d", []],
     ["PolygonVertexIndex", [[<array_of_integers>]], "i", []],
     ["Edges", [[<array_of_integers>]], "i", []],
     ["GeometryVersion", [124], "I", []],
     ["LayerElementNormal", [0], "I", [
         ["Version", [101], "I", []],
         ["Name", [""], "S", []],
         ["MappingInformationType", ["ByVertice"], "S", []],
         ["ReferenceInformationType", ["Direct"], "S", []],
         ["Normals", [[<array_of_floats>]], "d", []]
     ]],
     ["LayerElementSmoothing", [0], "I", [
         ["Version", [102], "I", []],
         ["Name", [""], "S", []],
         ["MappingInformationType", ["ByPolygon"], "S", []],
         ["ReferenceInformationType", ["Direct"], "S", []],
         ["Smoothing", [[<array_of_integers>]], "i", []]  # Yep, int32 for bool values...
     ]],
     ["LayerElementUV", [0], "I", [
         ["Version", [101], "I", []],
         ["Name", ["UVMap"], "S", []],
         ["MappingInformationType", ["ByPolygonVertex"], "S", []],
         ["ReferenceInformationType", ["IndexToDirect"], "S", []],
         ["UV", [[<array_of_floats>]], "d", []],
         ["UVIndex", [[<array_of_integers>]], "i", []],
     ]],
     ["LayerElementUV", [1], "I", [
         ["Version", [101], "I", []],
         ["Name", ["UVMap.001"], "S", []],
         ["MappingInformationType", ["ByPolygonVertex"], "S", []],
         ["ReferenceInformationType", ["IndexToDirect"], "S", []],
         ["UV", [[<array_of_floats>]], "d", []],
         ["UVIndex", [[<array_of_integers>]], "i", []],
     ]],
     ["LayerElementMaterial", [0], "I", [
         ["Version", [101], "I", []],
         ["Name", ["gold"], "S", []],
         ["MappingInformationType", ["AllSame"], "S", []],
         ["ReferenceInformationType", ["IndexToDirect"], "S", []],
         ["Materials", [[0]], "i", []]
     ]],
     ["Layer", [0], "I", [
         ["Version", [100], "I", []],
         ["LayerElement", [], "", [
             ["Type", ["LayerElementNormal"], "S", []],
             ["TypedIndex", [0], "I", []]
         ]],
         ["LayerElement", [], "", [
             ["Type", ["LayerElementMaterial"], "S", []],
             ["TypedIndex", [0], "I", []]
         ]],
         ["LayerElement", [], "", [
             ["Type", ["LayerElementSmoothing"], "S", []],
             ["TypedIndex", [0], "I", []]
         ]]
     ]]
     ["Layer", [1], "I", [
         ["Version", [100], "I", []],
         ["LayerElement", [], "", [
             ["Type", ["LayerElementUV"], "S", []],
             ["TypedIndex", [1], "I", []]
         ]],
     ]]
 ]]
```

### Armature and Bones

In FBX, there is no real armature concept, you rather have chains of bones, which are nearly only defined by "Model" elements (FbxNode), i.e. loc/rot/scale from parent bone. Root bones are children of an empty object (Model), which plays the role of an armature.

In other words, armatures are presented more like a set of chains of parented-hooks.

Each bone/hook is represented by a "LimbNode", which also contains a "Size" parameter (its length), not sure this is used/understood by all importers though.

Bones and mesh are "linked" first by a BindPose element, which stores the (transform) matrix of the mesh and all bones in global (world) space, at the time of binding.

### Animation

(New) animation in FBX is actually fairly simple: you have a Stack (usually only one per FBX file, else you may have compatibility issues), to which a set of Layers are linked.

A set of CurveNodes is linked to each AnimLayer, which define which properties are animated:

```JSON
["AnimationCurveNode", [2045776, "T::AnimCurveNode", ""], "LSS", [
     ["Properties70", [], "", [
         ["P", ["d", "Compound", "", ""], "SSSS", []]]]]]]],
         ["P", ["d|X", "Number", "", "A", 2.227995], "SSSSD", []],
         ["P", ["d|Y", "Number", "", "A", 3.0238389999999997], "SSSSD", []],
         ["P", ["d|Z", "Number", "", "A", -1.49012e-08], "SSSSD", []]]]]]
```

```JSON
 ["AnimationCurve", [924545958, "::AnimCurve", ""], "LSS", [
     ["Default", [-120.30426094607161], "D", []],
     ["KeyVer", [4008], "I", []],
     ["KeyTime", [[1847446320, 3694892640, 5542338960, 7389785280, 9237231600, 11084677920, 12932124240, 14779570560, 16627016880, 18474463200, 20321909520, 22169355840, 24016802160, 25864248480, 27711694800, 29559141120, 31406587440, 33254033760, 35101480080, 36948926400, 38796372720, 90524869680]], "l", []],
     ["KeyValueFloat", [[-90.00000762939453, -90.16700744628906, -90.67182159423828, -91.51591491699219, -92.69371032714844, -94.19062042236328, -95.98118591308594, -98.02799224853516, -100.28105926513672, -102.67914581298828, -105.1521224975586, -107.62510681152344, -110.02317810058594, -112.27626037597656, -114.32304382324219, -116.11365509033203, -117.61054229736328, -118.78833770751953, -119.63243103027344, -120.13725280761719, -120.30426025390625, -120.30426025390625]], "f", []],
     ["KeyAttrFlags", [[24840]], "i", []],
     ["KeyAttrDataFloat", [[0.0, 0.0, 9.419963346924634e-30, 0.0]], "f", []],
     ["KeyAttrRefCount", [[22]], "i", []]]]
```



```JSON
 ["KeyAttrFlags", [[A, B, A]], "i", []],
 ["KeyAttrDataFloat", [[A1, A2, A3, A4, B1, B2, B3, B4, A1, A2, A3, A4]], "f", []],
 ["KeyAttrRefCount", [[10, 5, 10]], "i", []]]]
```









